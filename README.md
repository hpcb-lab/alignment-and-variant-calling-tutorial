# NGS alignment and variant calling

## Part 0: Setup

We're going to use a bunch of fun tools for working with genomic data:

1. [bwa](https://github.com/lh3/bwa)
2. [samtools](https://github.com/samtools/samtools)
3. [htslib](https://github.com/samtools/htslib)
4. [vt](https://github.com/atks/vt)
5. [freebayes](https://github.com/ekg/freebayes)
6. [vcflib](https://github.com/ekg/vcflib/)
7. [seqtk](https://github.com/lh3/seqtk)

In most cases, you can download and build these using this kind of pattern:

```bash
git clone https://github.com/lh3/bwa
cd bwa && make
```

or

```bash
git clone --recursive https://github.com/ekg/vcflib
cd vcflib && make
```

Let's assume you're in an environment where you've already got them available.

## Part 1: Aligning E. Coli data with `bwa mem`

[E. Coli K12](https://en.wikipedia.org/wiki/Escherichia_coli#Model_organism) is a common laboratory strain that has lost its ability to live in the human intestine, but is ideal for manipulation in a controlled setting.
The genome is relatively short, and so it's a good place to start learning about alignment and variant calling.

### E. Coli K12 reference

We'll get some test data to play with. First, [the E. Coli K12 reference](http://www.ncbi.nlm.nih.gov/nuccore/556503834), from NCBI. It's a bit of a pain to pull out of the web interface so [you can also download it here](http://hypervolu.me/~erik/genomes/E.coli_K12_MG1655.fa).

```bash
# the start of the genome, which is circular but must be represented linearly in FASTA
curl -s http://hypervolu.me/%7Eerik/genomes/E.coli_K12_MG1655.fa | head
# >NC_000913.3 Escherichia coli str. K-12 substr. MG1655, complete genome
# AGCTTTTCATTCTGACTGCAACGGGCAATATGTCTCTGTGTGGATTAAAAAAAGAGTGTCTGATAGCAGC
# TTCTGAACTGGTTACCTGCCGTGAGTAAATTAAAATTTTATTGACTTAGGTCACTAAATACTTTAACCAA
# ...
```

### E. Coli K12 Illumina 2x300bp MiSeq sequencing results

For testing alignment, let's get some data from a [recently-submitted sequencing run on a K12 strain from the University of Exeter](http://www.ncbi.nlm.nih.gov/sra/?term=SRR1770413). We can us the [sratoolkit](https://github.com/ncbi/sratoolkit) to directly pull the sequence data (in paired FASTQ format) from the archive:

```bash
fastq-dump --split-files SRR1770413
```

`fastq-dump` is in the SRA toolkit. It allows directly downloading data from a particular sequencing run ID. SRA stores data in a particular compressed format (SRA!) that isn't directly compatible with any downsteam tools, so it's necessary to put things into [FASTQ](https://en.wikipedia.org/wiki/FASTQ_format) for further processing. The `--split-files` part of the command ensures we get two files, one for the first and second mate in each pair. We'll use them in this format when aligning.

```bash
# alternatively, you may want to first download, and then dump
# but this seems to fail sometimes for me
wget ftp://ftp-trace.ncbi.nlm.nih.gov/sra/sra-instant/reads/ByRun/sra/SRR/SRR177/SRR1770413/SRR1770413.sra
sra-dump --split-files SRR1770413.sra
```

These appear to be paired 300bp reads from a modern MiSeq.

### E. Coli O104:H4 HiSeq 2000 2x100bp

As a point of comparison, let's also pick up a [sequencing data set from a different E. Coli strain](http://www.ncbi.nlm.nih.gov/sra/SRX095630[accn]). This one is [famous for its role in foodborne illness](https://en.wikipedia.org/wiki/Escherichia_coli_O104%3AH4#Infection) and is of medical interest.

```bash
fastq-dump --split-files SRR341549
```

## Setting up our reference indexes

### FASTA file index

First, we'll want to allow tools (such as our variant caller) to quickly access certain regions in the reference. This is done using the samtools `.fai` FASTA index format, which records the lengths of the various sequences in the reference and their offsets from the beginning of the file.

```bash
samtools faidx E.coli_K12_MG1655.fa
```

Now it's possible to quickly obtain any part of the E. Coli K12 reference sequence. For instance, we can get the 200bp from position 1000000 to 1000200. We'll use a special format to describe the target region `[chr]:[start]-[end]`.

```bash
samtools faidx E.coli_K12_MG1655.fa NC_000913.3:1000000-1000200
```

We get back a small FASTA-format file describing the region:

```text
>NC_000913.3:1000000-1000200
GTGTCAGCTTTCGTGGTGTGCAGCTGGCGTCAGATGACAACATGCTGCCAGACAGCCTGA
AAGGGTTTGCGCCTGTGGTGCGTGGTATCGCCAAAAGCAATGCCCAGATAACGATTAAGC
AAAATGGTTACACCATTTACCAAACTTATGTATCGCCTGGTGCTTTTGAAATTAGTGATC
TCTATTCCACGTCGTCGAGCG
```

### BWA's FM-index

BWA uses the [FM-index](https://en.wikipedia.org/wiki/FM-index), which a compressed full-text substring index based around the [Burrows-Wheeler transform](https://en.wikipedia.org/wiki/Burrows%E2%80%93Wheeler_transform).
To use this index, we first need to build it:

```bash
bwa index E.coli_K12_MG1655.fa
```

You should see `bwa` generate some information about the build process:

```text
[bwa_index] Pack FASTA... 0.04 sec
[bwa_index] Construct BWT for the packed sequence...
[bwa_index] 2.26 seconds elapse.
[bwa_index] Update BWT... 0.04 sec
[bwa_index] Pack forward-only FASTA... 0.03 sec
[bwa_index] Construct SA from BWT and Occ... 0.72 sec
[main] Version: 0.7.8-r455
[main] CMD: bwa index E.coli_K12_MG1655.fa
[main] Real time: 3.204 sec; CPU: 3.121 sec
```

And, you should notice a new index file which has been made using the FASTA file name as prefix:

```bash
ls -rt1 E.coli_K12_MG1655.fa*
# -->
E.coli_K12_MG1655.fa
E.coli_K12_MG1655.fa.fai
E.coli_K12_MG1655.fa.bwt
E.coli_K12_MG1655.fa.pac
E.coli_K12_MG1655.fa.ann
E.coli_K12_MG1655.fa.amb
E.coli_K12_MG1655.fa.sa
```

### Aligning our data against the E. Coli K12 reference

Here's an outline of the steps we'll follow to align our K12 strain against the K12 reference:

1. use bwa to generate SAM records for each read
2. convert the output to BAM
3. sort the output
4. remove PCR duplicates that result from exact duplication of a template during amplification
5. convert to compressed BAM, and save

We could the steps one-by-one, generating an intermediate file for each step.
However, this isn't really necessary unless we want to debug the process, and it will make a lot of excess files which will do nothing but confuse us when we come to work with the data later.
Thankfully, it's easy to use [unix pipes](https://en.wikiepdia.org/wiki/Pipeline_%28Unix%29) to stream these tools together (see this [nice thread about piping bwa and samtools together on biostar](https://www.biostars.org/p/43677/) for a discussion of the benefits and possible drawbacks of this).

You can now run the alignment using a piped approach. _Replace `$threads` with the number of CPUs you would like to use for alignment._ Not all steps in `bwa` run in parallel, but the alignment, which is the most time-consuming step, does. You'll need to set this given the available resources you have.

```bash
bwa mem -t $threads -R '@RG\tID:K12\tSM:K12' \
    ~/ref/E.coli_K12_MG1655.fa SRR1770413_1.fastq.gz SRR1770413_2.fastq.gz \
    | samtools view -Shu - \
    | samtools sort -o - x \
    | samtools rmdup - - >SRR1770413.bam
```

Breaking it down by line:

- *alignment with bwa*: `bwa mem -t $threads -R '@RG\tID:K12\tSM:K12'` --- this says "align using so many threads" and also "give the reads the read group K12 and the sample name K12"
- *reference and FASTQs* `~/ref/E.coli_K12_MG1655.fa SRR1770413_1.fastq.gz SRR1770413_2.fastq.gz` --- this just specifies the base reference file name (`bwa` finds the indexes using this) and the input alignment files. The first file should contain the first mate, the second file the second mate.
- *conversion to BAM*: `samtools view -Shu -` --- this reads SAM from stdin (`-S` and the `-` specifier in place of the file name indicate this) and converts to uncompressed BAM (there isn't need to compress, as it's just going to be parsed by the next program in the pipeline.
- *sorting the BAM file*: `samtools sort -o - x` --- read from stdin and output to stdout (`-o`). The `x` means nothing, it's just a placeholder to work around [a bug in samtools](https://github.com/samtools/samtools/issues/356).
- *removing PCR duplicates*: `samtools rmdup - -` --- this completely removes reads which appear to be redundant PCR duplicates based on their read mapping position.

Now, run the same alignment process for the O104:H4 strain's data. Make sure to specify a different sample name via the `-R '@RG...` flag incantation:

```bash
bwa mem -t $threads -R '@RG\tID:O104_H4\tSM:O104_H4' \
    ~/ref/E.coli_K12_MG1655.fa SRR341549_1.fastq.gz  SRR341549_2.fastq.gz \
    | samtools view -Shu - \
    | samtools sort -o - x \
    | samtools rmdup - - >SRR341549.bam
```

## Part 2: Calling variants

Now that we have our alignments sorted, we can quickly determine variation against the reference by scanning through them using a variant caller.
There are many options, including [samtools mpileup](http://samtools.sourceforge.net/samtools.shtml), [platypus](http://www.well.ox.ac.uk/platypus), and the [GATK](https://www.broadinstitute.org/gatk/).
For this tutorial, we'll keep things simple and use [freebayes](https://github.com/ekg/freebayes). It has a number of advantages in this context (bacterial genomes), such as long-term support for haploid (and polyploid) genomes. However, the best reason to use it is that it just works and produces a very well-annotated VCF output that is suitable for immediate downstream filtering.

### Variant calls with `freebayes`

Because we've set up the samples

```bash
freebayes -f ~/ref/E.coli_K12_MG1655.fa 
```

Calling them jointly can help if we have a population of samples to use to help remove calls from paralagous regions. The Bayesian model in freebayes combines the data likelihoods from sequencing data with an estimate of the probability of observing a given set of genotypes under assumptions of neutral evolution and a [panmictic](https://en.wikipedia.org/wiki/Panmixia) population. For instance, [it would be very unusual to find a locus at which all the samples are heterozygous](https://en.wikipedia.org/wiki/Hardy%E2%80%93Weinberg_principle). It also helps improve statistics about observational biases (like strand bias, read placement bias, and allele balance in heterozygotes) by bringing more data into the algorithm.

However, in this context, we only have two samples and the best reason to call them jointly is to make sure we have a genotype for each one at every locus where a non-reference allele passes the caller's thresholds in either sample.

### Take a peek with `vt`

### Filtering using the transition/transversion ratio (ts/tv)

### Comparing the K12 and O104:H4 strains

## Part 3: When you know the truth

### The NIST Genome in a Bottle truth set for NA12878

### Calling variants in [20p12.2](http://genome-euro.ucsc.edu/cgi-bin/hgTracks?db=hg19&position=chr20%3A9200001-12100000)

To keep things quick enough for the tutorial, let's grab a little chunk of an NA12878 dataset. Let's use [20p12.2](http://genome-euro.ucsc.edu/cgi-bin/hgTracks?db=hg19&position=chr20%3A9200001-12100000).

We don't need to download the entire BAM file to do this. `samtools` can download the BAM index (`.bai`) and then use this to jump in the remote file hosted over FTP or HTTP.

```bash
samtools view -b ftp://ftp-trace.ncbi.nih.gov/giab/ftp/technical/NA12878_data_other_projects/alignment/XPrize_Illumina_WG.bam 20:9200001-12100000 >NA12878.20p12.2.XPrize.bam
```

## Part 4: Exploring alternative genomic contexts

### Simulation with mutatrix and wgsim

### A simulated population

### Polyploids and pooled sequencing contexts

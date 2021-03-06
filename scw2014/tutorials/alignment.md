---
title: "Processing fastq"
output:
  knitrBootstrap::bootstrap_document:
    theme: readable
    highlight: zenburn
    theme.chooser: TRUE
    highlight.chooser: TRUE
  html_document:
    highlight: zenburn

---


# Setting up Orchestra
To get on Orchestra you want to connect via ssh, and to turn X11 forwarding on. Turning
X11 forwarding will let the Orchestra machines open windows on your local machine, which
is useful for looking at the data.

Connect to Orchestra using X11 forwarding:
```bash
    ssh -X your_user_name@orchestra.med.harvard.edu
```
Orchestra is set up with separate login nodes and compute nodes. You don't want to be
doing any work on the login node, as that is set aside for doing non computationally
intensive tasks and running code on there will make Orchestra slow for everybody.
Here is how to connect to a compute node in interactive mode, meaning you can
type commands:
```bash
    bsub -q interactive bash
```
Do that and you're ready to roll. You should see that you are now connected to a node
named by an instrument like ```clarinet```.

# Introduction to the data
The raw data we will be using for this part of the workshop lives here:
```bash
    /groups/pklab/scw2014/ES.MEF/subset
```
Those files are a 1000 reads from ~ 100 samples from a single-cell
RNA-seq experiment looking at mouse embryonic fibroblasts (MEF) and
embryonic stem (ES) cells. These are files you might get from your
sequencing core after you send them off for sequencing. We'll be
taking a subset of those files, looking at them to make sure they are
of good quality, aligning them to the mouse genome and producing a table
of number of reads aligning to each gene for each sample. The counts table
will be the starting point for the more interesting downstream analyses.

We will be using programs that are installed here:
```bash
    /opt/bcbio/local/bin/
```
The output of typing ```which fastqc``` at your prompt should be
`/opt/bcbio/local/bin/fastqc`. If it isn't flag someone down and
we will fix it.


We're going to do this like a cooking show, we'll use those ES.MEF
files as a small example just to make sure everything is working, then
we'll look at pre-computed results on a full set of samples.

# Setup
The first thing we will do is copy the small test data over into your own directory.
```bash
    mkdir ~/workshop
    cd ~/workshop
    cp -r /groups/pklab/scw2014/ES.MEF/subset .
```
These commands mean:

1. make a directory (mkdir) named workshop
2. change into the directory (cd) named workshop
3. copy (cp) the folder  /groups/pklab/scw2014/ES.MEF/subset and everything underneath it to the current directory

# Quality control
With the start of any data analysis it is important to poke around at
your data to see where the warts are. We are expecting single-cell
datasets to be extra messy; we should be expecting failures in
preparing the libraries, failures in adequately capturing the
transcriptome of the cell, libraries that sequence the same reads
repeatedly and other issues. In particular we expect the libraries to
not be very complex; these are just tags of the end of a transcript
and so for each transcript there are a limited number of positions
where a read can be placed. We'll see how this feature skews some of the
more commonly used quality metrics now.

For RNA-seq data many common issues can be detected right off the bat
just by looking at some features of the raw reads. The most commonly
used program to look at the raw reads is
[FastQC](http://www.bioinformatics.babraham.ac.uk/projects/fastqc/).
FastQC is pretty fast, especially on small files, so we can run FastQC
on one of the entire files. First lets copy one of those files over:
```bash
    mkdir ~/workshop/fastq
    cp /groups/pklab/scw2014/ES.MEF/fastq/L139_ESC_1.fq ~/workshop/fastq/
    cd ~/workshop/fastq
```
Now we can run FastQC on the file by typing:
```bash
    fastqc L139_ESC_1.fq
```
And look at the nice HTML report it generates with Firefox:

```bash
    firefox L139_ESC_1_fastqc.html
```

This library looks pretty rough. The per base sequence quality plot shows some major
quality problems during sequencing; having degrading quality as you sequence further
is normal, but this is severe drop off. Severe quality drop off like this is generally due
to a technical issue with the sequencer, it is possible it ran out or was running low on a
reagent. The good news is it doesn't affect all of the reads, the median value
still has a PHRED score > 20 (so 1 in 100 probability of an error), and most aligners can
take into account the poor quality so this isn't as bad as it looks.

More worrying is the non-uniform per base sequence content plot. It depends on the genome, but
for the mouse if you are randomly sampling
from the transcriptome then you should expect there to be a pretty even distribution of
GC-AT in each sequence with a slight GC/AT bias. We can see that is not the case at all,
and the next plot, the per sequence GC content plot, has a huge GC spike in the middle.
Usually you see plots like these when the overall complexity of the reads that are
sequenced is low; by that we mean you have tended to sequence the same sample repeatedly.

That notion is reinforced looking at the duplication plot. If we de-duplicate the reads,
meaning remove reads where we have seen the same exact read twice, we'd throw out > 75%
of the data. It is also reinforced by the list of kmers that are more enriched than
would be expected by chance; a kmer is just every possible k length mer that is seen in
a sequence. For example all 3-mers of ```ACGT``` are ```ACG CGT```. We'd expect all
of these features if we were sequencing the same sequences repeatedly.

One thing we would not expect, however, is the big uptick at the end of the kmer content
plot; the sequences at the end look like some kind of adapter contamination issue. We'd
expect these reads to not align unless we trimmed the adapter sequence off.

What are those sequences? We can search for the reads that have one of
those enriched sequences with grep (*g*lobally search a *r*egular
*e*xpression and *p*rint) which print out every line in a file that
matches a search string. grep the L139_ESC_1.fq file like this:

```bash
    grep TTGATATGGG L139_ESC_1.fq
```

You should see a lot of sequences that look like this:

```bash
    AATTCGTGGAGAAAGAAATGGCTCGTCTGGCAGCATTTGATATGGG
```

If we BLAST this sequence in the mouse, we come up empty, so it is
some kind of contaminant sequence, it isn't clear where it comes from.
The protocol listed
[here](http://genome.cshlp.org/content/21/7/1160.long) doesn't have
too many clues either. If we could figure out what these sequences
are, it would help troubleshoot the preparation protocol and we might
be able to align more reads. As it is, these sequences are unlikely to align to
the mouse genome, so they mostly represent wasted sequencing.


# Alignment
For aligning RNA-seq reads it is necessary to use an aligner that is splicing
aware; reads crossing splice junctions have gaps when aligned to the
genome and the aligner has to be able to handle that possibility.
There are a wide variety of aligners to choose from that handle
spliced reads but the two most commonly used are
[Tophat](http://ccb.jhu.edu/software/tophat/index.shtml) and
[STAR](https://code.google.com/p/rna-star/). They both have similar
accuracy but STAR is much, much faster than Tophat at the cost of
using much more RAM; to align to the human genome you need ~ 40 GB of
RAM, so it isn't something you will be able to run on your laptop or
another type of RAM restricted computing environment. For this exercise
we will use Tophat.

To align reads with Tophat you need three things.

1. The genome sequence of the organism you are working with in FASTA format.
2. A gene annotation for your genome in Gene Transfer Format (GTF). For example from Ensembl.
3. The FASTQ file of reads you want to align.

First you must make an Bowtie2 index of the genome sequence; this
allows the Tophat algorithm to quickly find regions of the genome
where each read might align. We have done this step already, so don't
type in these commands, but if you need to do it on your own, here is
how to do it:

```bash
    # don't type this in
    bowtie2-build your_fasta_file genome_name
```
We've precomputed indexes for the mm10 genome here and downloaded a GTF file of
Ensembl release 75 here:

```bash
    /groups/bcbio/biodata/genomes/Mmusculus/mm10/bowtie2/
    /groups/bcbio/biodata/genomes/Mmusculus/mm10/rnaseq/ref-transcripts.gtf
```

Finally we've precomputed an index of the gene sequences from the ```ref-transcripts.gtf```
file here:

```bash
     /groups/bcbio/biodata/genomes/Mmusculus/mm10/rnaseq/tophat/
```

We will use this precomputed index of the gene sequences instead of the GTF file because
it is much faster; you can use either and they have the same output.


Now we're ready to align the reads to the mm10 genome. We will align two ESC samples and two MEF samples:

```bash
    cd ~/workshop/subset
    bsub -J L139_ESC_1 -W 00:20 -q short "tophat -o L139_ESC_1_tophat --no-coverage-search --transcriptome-index=/groups/bcbio/biodata/genomes/Mmusculus/mm10/rnaseq/tophat/mm10_transcriptome /groups/bcbio/biodata/genomes/Mmusculus/mm10/bowtie2/mm10 L139_ESC_1.subset.fastq; mv L139_ESC_1_tophat/accepted_hits.bam L139_ESC_1_tophat/L139_ESC_1.bam"
    bsub -J L139_ESC_2 -W 00:20 -q short "tophat -o L139_ESC_2_tophat --no-coverage-search --transcriptome-index=/groups/bcbio/biodata/genomes/Mmusculus/mm10/rnaseq/tophat/mm10_transcriptome /groups/bcbio/biodata/genomes/Mmusculus/mm10/bowtie2/mm10 L139_ESC_2.subset.fastq; mv L139_ESC_2_tophat/accepted_hits.bam L139_ESC_2_tophat/L139_ESC_2.bam"
    bsub -J L139_MEF_49 -W 00:20 -q short "tophat -o L139_MEF_49_tophat --no-coverage-search --transcriptome-index=/groups/bcbio/biodata/genomes/Mmusculus/mm10/rnaseq/tophat/mm10_transcriptome /groups/bcbio/biodata/genomes/Mmusculus/mm10/bowtie2/mm10 L139_MEF_49.subset.fastq; mv L139_MEF_49_tophat/accepted_hits.bam L139_MEF_49_tophat/L139_MEF_49.bam"
    bsub -J L139_MEF_50 -W 00:20 -q short "tophat -o L139_MEF_50_tophat --no-coverage-search --transcriptome-index=/groups/bcbio/biodata/genomes/Mmusculus/mm10/rnaseq/tophat/mm10_transcriptome /groups/bcbio/biodata/genomes/Mmusculus/mm10/bowtie2/mm10 L139_MEF_50.subset.fastq; mv L139_MEF_50_tophat/accepted_hits.bam L139_MEF_50_tophat/L139_MEF_50.bam"
```

Each of these should complete in about ten minutes. Since we ran them all in parallel on
the cluster, the whole set should take about ten minutes instead of 40. Full samples would
take hours. ```-J``` names the job so you can see what it is when you run ```bjobs``` to
check the status of the jobs. ```-W 00:20``` tells the scheduler the job should take about
20 minutes. ```-q short``` submits the job to the short queue. At the end we tack on a command
to move the Tophat output filename ```accepted_hits.bam``` to something more evocative.

This is fine for a small number of samples, but if you wanted to run a full set of hundreds of cells, doing this manually for every sample is a waste of time and prone to errors. You can run all of these automatically by writing a loop:

```bash
    # don't type this in, it is here for future reference
    for file in *.fastq; do
        samplename=`basename $file .fastq`
        bsub -W 00:20 -q short "tophat -o $samplename_tophat --no-coverage-search --transcriptome-index=/groups/bcbio/biodata/genomes/Mmusculus/mm10/rnaseq/tophat/mm10_transcriptome /groups/bcbio/biodata/genomes/Mmusculus/mm10/bowtie2/mm10 $file; mv $samplename_tophat/accepted_hits.bam $samplename_tophat/$samplename.bam
    done
```

This will loop over all of the files with a .fastq extension in the current directory and
align them with Tophat. We'll skip ahead now to doing some quality control of the alignments
and finally counting the reads mapping to each feature.

## Quality checking the alignments
There are several tools to spot check the alignments, it is
common to run [RNA-SeQC](http://www.broadinstitute.org/cancer/cga/rna-seqc)
on the alignment files to generate some alignment stats and to determine the
overall coverage, how many genes were detected and so on. Another option for a suite of
quality checking tools is
[RSeQC](http://dldcc-web.brc.bcm.edu/lilab/liguow/CGI/rseqc/_build/html/). For real
experiments it is worth it to look at the output of these tools and see if anything
seems amiss.

# Counting reads with featureCounts
The last step is to count the number of reads mapping to the features are are interested in.
Quantitation can be done at multiple levels; from the level of counting the number of reads
supporting a specific splicing event, to the number of reads aligning to an isoform of
a gene or the total reads mapping to a known gene. We'll be quantitating the latter, the
total number of reads that can uniquely be assigned to a gene. There are several tools to
do this, we will use [featureCounts](http://bioinf.wehi.edu.au/featureCounts/) because
it is very fast and accurate.

```bash
    featureCounts --primary -a /groups/bcbio/biodata/genomes/Mmusculus/mm10/rnaseq/ref-transcripts.gtf -o combined.featureCounts L139_ESC_1_tophat/L139_ESC_1.bam L139_ESC_2_tophat/L139_ESC_2.bam L139_MEF_49_tophat/L139_MEF_49.bam L139_MEF_50_tophat/L139_MEF_50.bam
```

```--primary``` tells featureCounts to only count the primary alignment for reads that
map multiple times to the genome. This ensures we do not double count reads that map to
multiple places in the genome.

We need to massage the format of this file so we can use it. We'll take the first column,
which is the gene ids and every column after the 6th, which has the counts of each sample.

```bash
    sed 1d combined.featureCounts | cut -f1,7- | sed s/Geneid/id/ > combined.counts
```

This command means steam edit (sed) the file combined.featureCounts, delete the first line,
keep the first column and everything the 7th column on and change the phrase ```Geneid```
to ```id```. This outputs a file in a format with *I* rows of genes and the *J* columns of samples.
Each cell is the number of reads that can be uniquely assigned to the gene *i* the sample
*j*. This file is of suitable format for loading into R.


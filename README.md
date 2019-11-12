---
title: "RNA-seq workflow: gene-level exploratory analysis and differential expression"
author:
- name: Michael I. Love
  affiliation: 
  - Department of Biostatistics, UNC-Chapel Hill, Chapel Hill, NC, US
  - Department of Genetics, UNC-Chapel Hill, Chapel Hill, NC, US
- name: Simon Anders
  affiliation: Zentrum für Molekulare Biologie der Universität Heidelberg, Heidelberg, Germany
- name: Vladislav Kim
  affiliation: &EMBL European Molecular Biology Laboratory (EMBL), Heidelberg, Germany
- name: Wolfgang Huber
  affiliation: *EMBL
date: 16 October, 2019
output: BiocStyle::html_document
bibliography: bibliography.bib
vignette: >
  %\VignetteIndexEntry{RNA-seq workflow at the gene level}
  %\VignetteEngine{knitr::rmarkdown}
abstract: |
  Here we walk through an end-to-end gene-level RNA-seq differential
  expression workflow using Bioconductor packages.  We will start from
  the FASTQ files, show how these were quantified to the reference
  transcripts, and prepare gene-level count datasets for downstream
  analysis. We will perform exploratory data analysis (EDA) for
  quality assessment and to explore the relationship between samples,
  perform differential gene expression analysis, and visually explore
  the results.
---

<!-- to compile this: rmarkdown::render("rnaseqGene.Rmd") -->

<!--
     # a list of all required libraries:
     reqlibs = sub(".*library\\(\"(.*?)\"\\).*","\\1",grep("library\\(",readLines("rnaseqGene.Rmd"),value=TRUE))
     find.package(reqlibs)
-->


```{r style, echo=FALSE, message=FALSE, warning=FALSE, results="asis"}
library("BiocStyle")
library("knitr")
library("rmarkdown")
opts_chunk$set(message = FALSE, error = FALSE, warning = FALSE,
               cache = FALSE, fig.width = 5, fig.height = 5)
```

**R version**: `r R.version.string`

**Bioconductor version**: `r BiocManager::version()`

**Package**: `r packageVersion("rnaseqGene")`


# Introduction

Bioconductor has many packages which support analysis of
high-throughput sequence 
data, including RNA sequencing (RNA-seq). The packages which we
will use in this workflow include core packages maintained by the
Bioconductor core team for working with gene annotations (gene and
transcript locations in the genome, as well as gene ID lookup).
We will also use contributed packages for statistical analysis and
visualization of sequencing data. Through scheduled releases every 6
months, the Bioconductor project ensures that all the packages within
a release will work together in harmony (hence the "conductor"
metaphor).  The packages used in this workflow are loaded with the
*library* function and can be installed by following the
[Bioconductor package installation instructions](http://bioconductor.org/install/#install-bioconductor-packages).

* A published version of this workflow, including reviewer reports and comments 
  is available
  at [F1000Research](http://f1000research.com/articles/4-1070).
  The version you are reading now differs from this one, primarily in
  that we now give code for performing fast **transcript quantification**
  followed by import in R/Bioconductor to perform gene-level analysis.
* Another Bioconductor workflow covering **differential transcript usage**
  (DTU) is the 
  [rnaseqDTU](https://bioconductor.org/packages/rnaseqDTU) workflow,
  with the published version likewise available at
  [F1000Research](https://f1000research.com/articles/7-952/v3).
* If you have questions about this workflow or any Bioconductor
  software, please post these to the
  [Bioconductor support site](https://support.bioconductor.org/).
  If the questions concern a specific package, you can tag the post with
  the name of the package, or for general questions about the workflow,
  tag the post with `rnaseqgene`. Note the
  [posting guide](http://www.bioconductor.org/help/support/posting-guide/)
  for crafting an optimal question for the support site.

## Experimental data

The data used in this workflow is stored in the
`r Biocexptpkg("airway")` package that summarizes an RNA-seq experiment
wherein airway smooth muscle cells were treated with dexamethasone, a
synthetic glucocorticoid steroid with anti-inflammatory effects
[@Himes2014RNASeq].  Glucocorticoids are used, for example, 
by people with asthma to reduce inflammation of the airways. In the experiment,
four primary human airway smooth muscle cell lines were treated with 1
micromolar dexamethasone for 18 hours. For each of the four cell
lines, we have a treated and an untreated sample. For more description
of the experiment see the
[PubMed entry 24926665](http://www.ncbi.nlm.nih.gov/pubmed/24926665)
and for raw data see the
[GEO entry GSE52778](http://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE52778).

<a id="quant"></a>

# Preparing quantification input to DESeq2

As input, the count-based statistical methods, such as
`r Biocpkg("DESeq2")` [@Love2014Moderated],
`r Biocpkg("edgeR")` [@Robinson2009EdgeR],
`r Biocpkg("limma")` with the voom method [@Law2014Voom],
`r Biocpkg("DSS")` [@Wu2013New], 
`r Biocpkg("EBSeq")` [@Leng2013EBSeq] and
`r Biocpkg("baySeq")` [@Hardcastle2010BaySeq],
expect input data as obtained, e.g., from RNA-seq or another
high-throughput sequencing experiment, in the form of a matrix of
un-normalized counts.  The value in the *i*-th row and the *j*-th
column of the matrix tells how many reads (or fragments, for
paired-end RNA-seq) can be assigned to gene *i* in sample *j*.
Analogously, for other types of assays, the rows of the matrix might
correspond e.g., to binding regions (with ChIP-Seq), or peptide
sequences (with quantitative mass spectrometry).

The values in the matrix should be counts or estimated counts of
sequencing reads/fragments. This is important for *DESeq2*'s
statistical model to hold, as only counts allow assessing the
measurement precision correctly. It is important to *never* provide
counts that were pre-normalized for sequencing depth/library size, as
the statistical model is most powerful when applied to un-normalized
counts, and is designed to account for library size differences
internally.

## Transcript quantification and *tximport* / *tximeta*

A previous version of this workflow (including the published version)
demonstrated how to align reads to the genome and then count the
number of reads that are consistent with gene models. We now recommend
a faster, alternative pipeline to genome alignment and read counting.
This workflow will demonstrate how to import transcript-level
quantification data, aggregating to the gene-level with *tximport* or
*tximeta*. Transcript quantification methods such as 
[Salmon](https://combine-lab.github.io/salmon/) [@Patro2017Salmon],
[kallisto](https://pachterlab.github.io/kallisto/) [@Bray2016Near], or 
[RSEM](http://deweylab.github.io/RSEM/) [@Li2011RSEM] perform mapping
or alignment of reads to reference transcripts, outputting estimated
counts per transcript as well as effective transcript lengths which
summarize bias effects. After running one of these tools, the
`r Biocpkg("tximport")` [@Soneson2015Differential] or
`r Biocpkg("tximeta")` [@Love2019]
packages can be used to assemble estimated count and offset matrices
for use with Bioconductor differential gene expression packages, as
will be demonstrated below.

A tutorial on how to use the *Salmon* software for quantifying
transcript abundance can be
found [here](https://combine-lab.github.io/salmon/getting_started/).
We recommend using the `--gcBias` 
[flag](http://salmon.readthedocs.io/en/latest/salmon.html#gcbias)
which estimates a correction factor for systematic biases
commonly present in RNA-seq data [@Love2016Modeling; @Patro2017Salmon], 
unless you are certain that your data do not contain such bias.

The advantages of using the transcript abundance quantifiers in
conjunction with *tximport*/*tximeta* to produce gene-level count matrices and
normalizing offsets, are: (1) this approach corrects for any potential
changes in gene length across samples (e.g. from differential isoform
usage) [@Trapnell2013Differential]; (2) some of these methods are
substantially faster and require less memory and disk usage compared
to alignment-based methods; and (3) it is possible to avoid discarding
those fragments that can align to multiple genes with homologous
sequence [@Robert2015Errors].  Note that transcript abundance
quantifiers skip the generation of large files which store read
alignments, instead producing smaller files which store estimated
abundances, counts, and effective lengths per transcript.  For more
details, see the manuscript describing this approach
[@Soneson2015Differential], and the `r Biocpkg("tximport")` package
vignette for software details.

`r Biocpkg("tximeta")` [@Love2019]
extends *tximport*, offering the same functionality, plus the
additional benefit of automatic addition of annotation metadata for
commonly used transcriptomes (GENCODE, Ensembl, RefSeq for human and
mouse). See the 
[tximeta vignette](https://bioconductor.org/packages/release/bioc/vignettes/tximeta/inst/doc/tximeta.htmlm)
package vignette for more details. *tximeta* produces a
*SummarizedExperiment* that can be loaded easily into *DESeq2* using
the `DESeqDataSet` function, which will be demonstrated below. We will
also discuss the various possible inputs into *DESeq2*, whether using
*tximport*, *tximeta*, *htseq* [@Anders2015HTSeqa], or a pre-computed
count matrix. 

## Quantifying with *Salmon*

As mentioned above, a short tutorial on how to use *Salmon* can be
found [here](https://combine-lab.github.io/salmon/getting_started/),
so instead we will provide the code that was used to quantify the
files used in this workflow. *Salmon* can be conveniently run on a
cluster using
the [Snakemake](https://snakemake.readthedocs.io/en/stable/) workflow
management system [@snakemake].

The following `Snakemake` file was used to quantify the eight samples that
were downloaded from the SRA (the SRR identifier is the run
identifier, and there was only one run per sample for these eight samples).

```
DATASETS = ["SRR1039508",
            "SRR1039509",
            "SRR1039512",
            "SRR1039513",
            "SRR1039516",
            "SRR1039517",
            "SRR1039520",
            "SRR1039521"]
SALMON = "/path/to/salmon_0.14.1/bin/salmon"
rule all:
  input: expand("quants/{dataset}/quant.sf", dataset=DATASETS)
rule salmon_quant:
    input:
        r1 = "fastq/{sample}_1.fastq.gz",
        r2 = "fastq/{sample}_2.fastq.gz",
        index = "/path/to/gencode.v29_salmon_0.14.1"
    output:
        "quants/{sample}/quant.sf"
    params:
        dir = "quants/{sample}"
    shell:
        "{SALMON} quant -i {input.index} -l A -p 6 --validateMappings \
         --gcBias --numGibbsSamples 20 -o {params.dir} \
		 -1 {input.r1} -2 {input.r2}"
```

The last line is the key one which runs *Salmon*. It says to quantify
using a specific *index*, with automatic library type detection, using
6 threads, with the validate mappings setting (this is default in
versions of *Salmon* $\ge$ 0.99), with GC bias correction, and writing
out 20 Gibbs samples (this is optional). The last three arguments
specify the output directory and the two paired read files.

The above Snakemake file requires that an index be created at
`/path/to/gencode.vVV_salmon_X.Y.Z`, where VV and X,Y,Z should help
specify the release of the reference transcripts and of *Salmon*.
For human and mouse reference transcripts, we recommend to use
[GENCODE](https://gencodegenes.org) [@gencode].

The *Salmon* index can be created easily with the following command:

```
salmon index -t transcripts.fa.gz -i name_of_index
```

If the transcripts are downloaded from GENCODE, it is recommended to
use something similar to the following command (which simply helps to
strip the extra information from the transcript names):

```
salmon index --gencode -t gencode.v29.transcripts.fa.gz \
  -i gencode.v29_salmon_X.Y.Z
```

The above Snakemake file can then be used to 
[execute Snakemake](https://snakemake.readthedocs.io/en/stable/executable.html)
in various ways, including submitting multiple jobs to a compute
cluster or in the cloud. The above Snakemake file was executed on a
cluster with SLURM scheduling, with the following line in a separate
job submitted to the cluster:

```
snakemake -j 4 --latency-wait 30 --cluster "sbatch -N 1 -n 6"
```

## Reading in data with *tximeta*

Later in the workflow, we will load an object that contains the
quantification data at the gene-level for all eight samples. However,
the `r Biocexptpkg("airway")` package also contains two quantification
directories output by *Salmon*, in order to demonstrate reading this data
into R/Bioconductor. In order to make the data package smaller, the
`quant.sf` files in the quantification directories have been gzipped,
so below where you see `quant.sf.gz`, you would probably use
`quant.sf` on your own machine. 

After we demonstrate importing with *tximeta*, we will load the full
count matrix corresponding to all samples and all data, which is
already provided in the same package, and will continue the analysis
with that full data object.

We first load the data package with the example data:

```{r loadairway}
library("airway")
```

The R function *system.file* can be used to find out where on your
computer the files from a package have been installed. Here we ask for
the full path to the `extdata` directory, where R packages store
external data, that is part of the `r Biocexptpkg("airway")` package.

```{r dir}
dir <- system.file("extdata", package="airway", mustWork=TRUE)
```

In this directory, we find a number of files, including eight BAM
files that were used in the previous version of this workflow
demonstrating alignment and counting. We will focus on the two
directories that are in the `quants` directory, which contain the
output from *Salmon* on two files.

```{r list.files}
list.files(dir)
list.files(file.path(dir, "quants"))
```

Typically, we have a table with detailed information for each of our
samples that links samples to the associated FASTQ and *Salmon* directories.
For your own project, you might create such a comma-separated
value (CSV) file using a text editor or spreadsheet software such as Excel.

We load such a CSV file with *read.csv*:

```{r sampleinfo}
csvfile <- file.path(dir, "sample_table.csv")
coldata <- read.csv(csvfile, row.names=1, stringsAsFactors=FALSE)
coldata
```

To demonstrate loading *Salmon* quantifiation data into R, we will
just work with the two samples that are provided in the *airway*
package. We create a column called `names` and a column called
`files`:

```{r makecoldata}
coldata <- coldata[1:2,]
coldata$names <- coldata$Run
coldata$files <- file.path(dir, "quants", coldata$names, "quant.sf.gz")
file.exists(coldata$files)
```

Now we load the *tximeta* package and run its main function:

```{r tximeta, message=TRUE}
library("tximeta")
se <- tximeta(coldata)
```

If the reference transcriptome checksum was recognized by tximeta
(details on this in the `r Biocpkg("tximeta")` vignette), and if we
have a working internet connection, *tximeta* will locate and download
the relevant annotation data from various sources. A few details: the
annotation data is only downloaded and parsed once, subsequently it
will used locally cached versions of the metadata as needed (if you
load data a second time that was quantified against the same reference
transcripts). Also, the very first time that one uses *tximeta*, it
will ask you to approve the default cache location (following the
paradigm of the cache location used by other R and Bioconductor
packages). You can change this location at any point later.

We will discuss what is the structure of the `se` object in the next
section, but we can first just consider the dimensions. Note that
*tximeta* imports data at the *transcript level*.

```{r lookse}
dim(se)
head(rownames(se))
```

As this workflow is concerned with gene-level analysis, we will now
summarize the transcript-level quantifications to the gene level
(which internally makes use of the methods in *tximport*
[@Soneson2015Differential]). The correct transcript-to-gene mapping
table is automatically created based on the metadata stored within the
`se` object.

```{r summarize, message=TRUE}
gse <- summarizeToGene(se)
```

Now we can check that the dimensions are reduced and the row IDs are
now gene IDs:

```{r lookgse}
dim(gse)
head(rownames(gse))
```

## *DESeq2* import functions

While the above section described use of *Salmon* and *tximeta*, there
are many possible inputs to *DESeq2*, each of which have their own
dedicated import functions.
The following tools can be used generate or compile count data for use
with *DESeq2*:
*tximport* [@Soneson2015Differential],
*tximeta* [@Love2019],
*htseq-count* [@Anders2015HTSeqa],
*featureCounts* [@Liao2014FeatureCounts],
*summarizeOverlaps* [@Lawrence2013Software].

function            | package                                              | framework      | output                 | *DESeq2* input function
--------------------|------------------------------------------------------|----------------|------------------------|-------------------------
*tximport*          | `r Biocpkg("tximport")`                              | R/Bioconductor | list of matrices       | *DESeqDataSetFromTximport*
*tximeta*           | `r Biocpkg("tximeta")`                               | R/Bioconductor | *SummarizedExperiment* | *DESeqDataSet*
*htseq-count*       | [HTSeq](http://www-huber.embl.de/users/anders/HTSeq) | Python         | files                  | *DESeqDataSetFromHTSeq* 
*featureCounts*     | `r Biocpkg("Rsubread")`                              | R/Bioconductor | matrix                 | *DESeqDataSetFromMatrix*
*summarizeOverlaps* | `r Biocpkg("GenomicAlignments")`                     | R/Bioconductor | *SummarizedExperiment* | *DESeqDataSet*

We will next describe the class of object created by *tximeta* which
was saved above as `se` and `gse`, and how to create a *DESeqDataSet*
object from this for use with *DESeq2* (the other functions above also
create a *DESeqDataSet*).

## SummarizedExperiment

```{r sumexp, echo=FALSE}
par(mar=c(0,0,0,0))
plot(1,1,xlim=c(0,100),ylim=c(0,100),bty="n",
     type="n",xlab="",ylab="",xaxt="n",yaxt="n")
polygon(c(45,90,90,45),c(5,5,70,70),col="pink",border=NA)
polygon(c(45,90,90,45),c(68,68,70,70),col="pink3",border=NA)
text(67.5,40,"assay(s)")
text(67.5,35,'e.g. "counts", ...')
polygon(c(10,40,40,10),c(5,5,70,70),col="skyblue",border=NA)
polygon(c(10,40,40,10),c(68,68,70,70),col="skyblue3",border=NA)
text(25,40,"rowRanges")
polygon(c(45,90,90,45),c(75,75,95,95),col="palegreen",border=NA)
polygon(c(45,47,47,45),c(75,75,95,95),col="palegreen3",border=NA)
text(67.5,85,"colData")
```

**The component parts of a *SummarizedExperiment* object.** The `assay` (pink
block) contains the matrix of counts, the `rowRanges` (blue block) contains
information about the genomic ranges and the `colData` (green block)
contains information about the samples. The highlighted line in each
block represents the first row (note that the first row of `colData`
lines up with the first column of the `assay`). 

The *SummarizedExperiment* container is diagrammed in the Figure above
and discussed in the latest Bioconductor paper [@Huber2015Orchestrating].
In our case, *tximeta* has created an object `gse` with three
matrices: "counts" - the estimated fragment counts for each gene and
sample, "abundance" - the estimated transcript abundances in TPM, and
"length" - the effective gene lengths which include changes in length
due to biases as well as due to transcript usage. The names of the
assays can be examined with *assayNames*, and the assays themselves
are stored as `assays` (a list of matrices). The first matrix in the
list can be pulled out via `assay`.
The `rowRanges` for our object is the *GRanges* of the genes (from the
left-most position of all the transcripts to the right-most position
of all the transcripts).
The component parts of the *SummarizedExperiment* are accessed with an
R function of the same name: `assay` (or `assays`), `rowRanges` and `colData`.

We now will load the full count matrix corresponding to all samples
and all data, which is provided in the *airway* package, and will
continue the analysis with the full data object. 
We can investigate this *SummarizedExperiment* object by looking at
the matrices in the `assays` slot, the phenotypic data about the samples in
`colData` slot, and the data about the genes in the `rowRanges` slot.

```{r loadfullgse}
data(gse)
gse
```

The counts are the first matrix, so we can examine them with just
`assay`: 

```{r assaysgse}
assayNames(gse)
head(assay(gse), 3)
colSums(assay(gse))
```

The `rowRanges`, when printed, shows the ranges for the first five and
last five genes:

```{r rowrangesgse}
rowRanges(gse)
```

The `rowRanges` also contains metadata about the sequences
(chromosomes in our case) in the `seqinfo` slot:

```{r lookseqinfo}
seqinfo(rowRanges(gse))
```

The `colData` for the *SummarizedExperiment* reflects the *data.frame*
that was provided to the `tximeta` function for importing the
quantification data. Here we can see that there are columns indicating
sample names, as well as the donor ID, and the treatment condition
(treated with dexamethasone or untreated).

```{r coldatagse}
colData(gse)
```

## Branching point

At this point, we have counted the fragments which overlap the genes
in the gene model we specified. This is a branching point where we
could use a variety of Bioconductor packages for exploration and
differential expression of the count data, including
`r Biocpkg("edgeR")` [@Robinson2009EdgeR],
`r Biocpkg("limma")` with the voom method [@Law2014Voom],
`r Biocpkg("DSS")` [@Wu2013New],
`r Biocpkg("EBSeq")` [@Leng2013EBSeq] and
`r Biocpkg("baySeq")` [@Hardcastle2010BaySeq].
@Schurch2016How
[compared performance](https://www.ncbi.nlm.nih.gov/pmc/articles/pmid/27022035/) 
of different statistical methods
for RNA-seq using a large number of biological replicates and can help
users to decide which tools make sense to use, and how many
biological replicates are necessary to obtain a certain sensitivity.
We will continue using `r Biocpkg("DESeq2")` [@Love2014Moderated].
The *SummarizedExperiment* object is all we
need to start our analysis. In the following section we will show how
to use it to create the data object used by `r Biocpkg("DESeq2")`.

<a id="construct"></a>

# The *DESeqDataSet* object, sample information and the design formula

Bioconductor software packages often define and use a custom class for
storing data that makes sure that all the needed data slots are
consistently provided and fulfill the requirements.  In addition,
Bioconductor has general data classes (such as the
*SummarizedExperiment*) that can be used to move data between
packages. Additionally, the core Bioconductor classes provide useful
functionality: for example, subsetting or reordering the rows or
columns of a *SummarizedExperiment* automatically subsets or reorders
the associated *rowRanges* and *colData*, which can help to prevent
accidental sample swaps that would otherwise lead to spurious
results. With *SummarizedExperiment* this is all taken care of behind
the scenes.

In *DESeq2*, the custom class is called *DESeqDataSet*. It is built on
top of the *SummarizedExperiment* class, and it is easy to convert
*SummarizedExperiment* objects into *DESeqDataSet* objects, which we
show below.  One of the two main differences is that the `assay` slot is
instead accessed using the *counts* accessor function, and the
*DESeqDataSet* class enforces that the values in this matrix are
non-negative integers.

A second difference is that the *DESeqDataSet* has an associated
*design formula*. The experimental design is specified at the
beginning of the analysis, as it will inform many of the *DESeq2*
functions how to treat the samples in the analysis (one exception is
the size factor estimation, i.e., the adjustment for differing library
sizes, which does not depend on the design formula).  The design
formula tells which columns in the sample information table (`colData`)
specify the experimental design and how these factors should be used
in the analysis.

First, let's examine the columns of the `colData` of `gse`.
We can see each of the columns just using the `$` directly on the 
*SummarizedExperiment* or *DESeqDataSet*.

```{r gsevars}
gse$donor
gse$condition
```

We can rename our variables if we want. Let's use `cell` to denote the
donor cell line, and `dex` to denote the treatment condition.

```{r gsevarsrename}
gse$cell <- gse$donor
gse$dex <- gse$condition
```

We can also change the names of the levels. It is critical when one
renames levels to not change the order. Here we will rename
`"Untreated"` as `"untrt"` and `"Dexamethasone"` as `"trt"`:

```{r renamelevels}
levels(gse$dex)
# when renaming levels, the order must be preserved!
levels(gse$dex) <- c("untrt", "trt")
```

The simplest design formula for differential expression would be
`~ condition`, where `condition` is a column in `colData(dds)` that
specifies which of two (or more groups) the samples belong to.
For the airway experiment, we will specify `~ cell + dex`
meaning that we want to test for the effect of dexamethasone (`dex`)
controlling for the effect of different cell line (`cell`). 

**Note:** it is prefered in R that the first level of a factor be the
reference level (e.g. control, or untreated samples). In this case,
when the `colData` table was assembled the untreated samples were
already set as the reference, but if this were not the case we could
use *relevel* as shown below. While `levels(...) <-` above was
simply for renaming the character strings associated with levels,
*relevel* is a very different function, which decides how the
variables will be coded, and how contrasts will be computed. For a
two-group comparison, the use of *relevel* to change the reference
level would flip the sign of a coefficient associated with a contrast
between the two groups.

```{r gsedex}
library("magrittr")
gse$dex %<>% relevel("untrt")
gse$dex
```

`%<>%` is the compound assignment pipe-operator from
the `r CRANpkg("magrittr")` package, the above line of code is a
concise way of saying:

```{r explaincmpass, eval = FALSE}
gse$dex <- relevel(gse$dex, "untrt")
```

For running *DESeq2* models, you can use R's formula notation to
express any fixed-effects experimental design.
Note that *DESeq2* uses the same formula notation as, for instance, the *lm*
function of base R. If the research aim is to determine for which
genes the effect of treatment is different across groups, then
interaction terms can be included and tested using a design such as
`~ group + treatment + group:treatment`. See the manual page for
`?results` for more examples. We will show how to use an interaction
term to test for condition-specific changes over time in a 
time course example below.

In the following sections, we will demonstrate the construction of the
*DESeqDataSet* from two starting points:

* from a *SummarizedExperiment* object 
* from a count matrix and a sample information table

For a full example of using the *HTSeq* Python package for read
counting, please see the `r Biocexptpkg("pasilla")` vignette.  For an
example of generating the *DESeqDataSet* from files produced by
*htseq-count*, please see the `r Biocpkg("DESeq2")` vignette.

## Starting from *SummarizedExperiment*

Again, we can quickly check the millions of fragments that could be
mapped by *Salmon* to the genes (the second argument of *round* tells
how many decimal points to keep).

```{r countreads}
round( colSums(assay(gse)) / 1e6, 1 )
```

Once we have our fully annotated *SummarizedExperiment* object,
we can construct a *DESeqDataSet* object from it that will then form
the starting point of the analysis.
We add an appropriate design for the analysis:

```{r loaddeseq2}
library("DESeq2")
```

```{r makedds}
dds <- DESeqDataSet(gse, design = ~ cell + dex)
```

## Starting from count matrices

In this section, we will show how to build an *DESeqDataSet* supposing
we only have a count matrix and a table of sample information.

**Note:** if you have prepared a *SummarizedExperiment* you should skip this
section. While the previous section would be used to construct a
*DESeqDataSet* from a *SummarizedExperiment*, here we first extract
the individual object (count matrix and sample info) from the
*SummarizedExperiment* in order to build it back up into a new object
-- only for demonstration purposes.
In practice, the count matrix would either be read in from a file or
perhaps generated by an R function like *featureCounts* from the
`r Biocpkg("Rsubread")` package [@Liao2014FeatureCounts].

The information in a *SummarizedExperiment* object can be
accessed with accessor functions. For example, to see the actual data,
i.e., here, the fragment counts, we use the *assay* function. (The *head*
function restricts the output to the first few lines.)

```{r}
countdata <- round(assays(gse)[["counts"]])
head(countdata, 3)
```

In this count matrix, each row represents a gene, each column a
sequenced RNA library, and the values give the estimated counts of
fragments that were probabilistically assigned to the respective gene
in each library by *Salmon*. We also have information on each of the
samples (the columns of the count matrix). If you've imported the
count data in some other way, for example loading a pre-computed count
matrix, it is **very important** to check manually that the columns of
the count matrix correspond to the rows of the sample information
table.

```{r}
coldata <- colData(gse)
```

We now have all the ingredients to prepare our data object in a form
that is suitable for analysis, namely:

* `countdata`: a table with the fragment counts
* `coldata`: a table with information about the samples

To now construct the *DESeqDataSet* object from the matrix of counts and the
sample information table, we use:

```{r}
ddsMat <- DESeqDataSetFromMatrix(countData = countdata,
                                 colData = coldata,
                                 design = ~ cell + dex)
```

We will continue with the object generated from the
*SummarizedExperiment* section. 

<a id="eda"></a>

# Exploratory analysis and visualization

There are two separate paths in this workflow; the one we will
see first involves *transformations of the counts*
in order to visually explore sample relationships.
In the second part, we will go back to the original raw counts
for *statistical testing*. This is critical because
the statistical testing methods rely on original count data
(not scaled or transformed) for calculating the precision of measurements. 

## Pre-filtering the dataset

Our count matrix with our *DESeqDataSet* contains many rows with only
zeros, and additionally many rows with only a few fragments total. In
order to reduce the size of the object, and to increase the speed of
our functions, we can remove the rows that have no or nearly no
information about the amount of gene expression.  Here we apply the
most minimal filtering rule: removing rows of the *DESeqDataSet* that
have no counts, or only a single count across all samples. Additional
weighting/filtering to improve power is applied at a later step in the
workflow. 

```{r}
nrow(dds)
keep <- rowSums(counts(dds)) > 1
dds <- dds[keep,]
nrow(dds)
```

For some datasets, it may make sense to perform additional
filtering. For example, one can specify that at least 3 samples have a
count of 10 or higher. One recommendation for the number of samples
would be set to the smallest group size. Such a rule could be
specified by creating a logic vector and subsetting the `dds` as
above. Here is an example of another rule we could have used (here not
used for filtering): 

```{r}
# at least 3 samples with a count of 10 or higher
keep <- rowSums(counts(dds) >= 10) >= 3
```

## The variance stabilizing transformation and the rlog

Many common statistical methods for exploratory analysis of
multidimensional data, for example clustering and *principal
components analysis* (PCA), work best for data that generally has the
same range of variance at different ranges of the mean values. When
the expected amount of variance is approximately the same across
different mean values, the data is said to be *homoskedastic*. For
RNA-seq counts, however, the expected variance grows with the mean. For
example, if one performs PCA directly on a matrix of
counts or normalized counts (e.g. correcting for differences in
sequencing depth), the resulting plot typically depends mostly
on the genes with *highest* counts because they show the largest
absolute differences between samples. A simple and often used
strategy to avoid this is to take the logarithm of the normalized
count values plus a pseudocount of 1; however, depending on the
choice of pseudocount, now the genes with the very *lowest* counts
will contribute a great deal of noise to the resulting plot, because
taking the logarithm of small counts actually inflates their variance.
We can quickly show this property of counts with some simulated
data (here, Poisson counts with a range of lambda from 0.1 to 100).
We plot the standard deviation of each row (genes) against the mean:

```{r meanSdCts}
lambda <- 10^seq(from = -1, to = 2, length = 1000)
cts <- matrix(rpois(1000*100, lambda), ncol = 100)
library("vsn")
meanSdPlot(cts, ranks = FALSE)
```

And for logarithm-transformed counts:

```{r meanSdLogCts}
log.cts.one <- log2(cts + 1)
meanSdPlot(log.cts.one, ranks = FALSE)
```

The logarithm with a small pseudocount amplifies differences when the
values are close to 0. The low count genes with low signal-to-noise
ratio will overly contribute to sample-sample distances and PCA
plots. 

As a solution, *DESeq2* offers two transformations for count data that
stabilize the variance across the mean: 
the *variance stabilizing transformation* (VST)
for negative binomial data with a dispersion-mean trend
[@Anders2010Differential], implemented in the *vst* function,
and the *regularized-logarithm transformation* or *rlog* [@Love2014Moderated].

For genes with high counts, both the VST and the rlog will give similar result
to the ordinary log2 transformation of normalized counts.  For genes
with lower counts, however, the values are shrunken towards a middle
value. The VST or rlog-transformed data then
become approximately homoskedastic (more flat trend in the *meanSdPlot*), 
and can be used directly for computing distances between samples,
making PCA plots, or as input to downstream methods which perform best
with homoskedastic data.

**Which transformation to choose?** 
The VST is much faster to compute and is less sensitive to high count
outliers than the rlog. The rlog tends to work well on
small datasets (n < 30), potentially outperforming the VST when there is
a wide range of sequencing depth across samples (an order of magnitude difference). 
We therefore recommend the VST for medium-to-large datasets (n > 30). 
You can perform both transformations and compare the `meanSdPlot` or
PCA plots generated, as described below.

Note that the two transformations offered by DESeq2 are provided for
applications *other* than differential testing. 
For differential testing we recommend the
*DESeq* function applied to raw counts, as described later
in this workflow, which also takes into account the dependence of the
variance of counts on the mean value during the dispersion estimation
step.

Both *vst* and *rlog* return a *DESeqTransform* object which is based
on the *SummarizedExperiment* class. The transformed values are no
longer counts, and are stored in the *assay* slot. The *colData* that
was attached to `dds` is still accessible:

```{r vst}
vsd <- vst(dds, blind = FALSE)
head(assay(vsd), 3)
colData(vsd)
```

Again, for the *rlog*:

```{r rlog}
rld <- rlog(dds, blind = FALSE)
head(assay(rld), 3)
```

In the above function calls, we specified `blind = FALSE`, which means
that differences between cell lines and treatment (the variables in
the design) will not contribute to the expected variance-mean trend of
the experiment. The experimental design is not used directly in the
transformation, only in estimating the global amount of variability in
the counts.  For a fully *unsupervised* transformation, one can set
`blind = TRUE` (which is the default).

To show the effect of the transformation, in the figure below
we plot the first sample
against the second, first simply using the *log2* function (after adding
1, to avoid taking the log of zero), and then using the VST and rlog-transformed
values. For the *log2* approach, we need to first estimate *size factors* to
account for sequencing depth, and then specify `normalized=TRUE`.
Sequencing depth correction is done automatically for the *vst* and *rlog*.

```{r transformplot, fig.width = 6, fig.height = 2.5}
library("dplyr")
library("ggplot2")
dds <- estimateSizeFactors(dds)
df <- bind_rows(
  as_data_frame(log2(counts(dds, normalized=TRUE)[, 1:2]+1)) %>%
         mutate(transformation = "log2(x + 1)"),
  as_data_frame(assay(vsd)[, 1:2]) %>% mutate(transformation = "vst"),
  as_data_frame(assay(rld)[, 1:2]) %>% mutate(transformation = "rlog"))
  
colnames(df)[1:2] <- c("x", "y")  
ggplot(df, aes(x = x, y = y)) + geom_hex(bins = 80) +
  coord_fixed() + facet_grid( . ~ transformation)  
```

**Scatterplot of transformed counts from two samples**. Shown are
scatterplots using the log2 transform of normalized counts (left),
using the VST (middle), and using the rlog (right). While the rlog is
on roughly the same scale as the log2 counts, the VST has a upward
shift for the smaller values. It is the differences between samples
(deviation from y=x in these scatterplots) which will contribute to
the distance calculations and the PCA plot.

We can see how genes with low counts (bottom left-hand corner) seem to
be excessively variable on the ordinary logarithmic scale, while the
VST and rlog compress differences for the low count genes
for which the data provide little information about differential
expression.

## Sample distances

A useful first step in an RNA-seq analysis is often to assess overall
similarity between samples: Which samples are similar to each other,
which are different? Does this fit to the expectation from the
experiment's design?

We use the R function *dist* to calculate the Euclidean distance
between samples. To ensure we have a roughly equal contribution from
all genes, we use it on the VST data. We need to
transpose the matrix of values using *t*, because the *dist* function
expects the different samples to be rows of its argument, and
different dimensions (here, genes) to be columns.

```{r}
sampleDists <- dist(t(assay(vsd)))
sampleDists
```

We visualize the distances in a heatmap in a figure below, using the function
*pheatmap* from the `r CRANpkg("pheatmap")` package.

```{r}
library("pheatmap")
library("RColorBrewer")
```

In order to plot the sample distance matrix with the rows/columns
arranged by the distances in our distance matrix,
we manually provide `sampleDists` to the `clustering_distance`
argument of the *pheatmap* function.
Otherwise the *pheatmap* function would assume that the matrix contains
the data values themselves, and would calculate distances between the
rows/columns of the distance matrix, which is not desired.
We also manually specify a blue color palette using the
*colorRampPalette* function from the `r CRANpkg("RColorBrewer")` package.

```{r distheatmap, fig.width = 6.1, fig.height = 4.5}
sampleDistMatrix <- as.matrix( sampleDists )
rownames(sampleDistMatrix) <- paste( vsd$dex, vsd$cell, sep = " - " )
colnames(sampleDistMatrix) <- NULL
colors <- colorRampPalette( rev(brewer.pal(9, "Blues")) )(255)
pheatmap(sampleDistMatrix,
         clustering_distance_rows = sampleDists,
         clustering_distance_cols = sampleDists,
         col = colors)
```

**Heatmap of sample-to-sample distances using the variance stabilizing transformed values.**

Note that we have changed the row names of the distance matrix to
contain treatment type and patient number instead of sample ID, so
that we have all this information in view when looking at the heatmap.

Another option for calculating sample distances is to use the
*Poisson Distance* [@Witten2011Classification], implemented in the
`r CRANpkg("PoiClaClu")` package.
This measure of dissimilarity between counts
also takes the inherent variance
structure of counts into consideration when calculating the distances
between samples. The *PoissonDistance* function takes the original
count matrix (not normalized) with samples as rows instead of columns,
so we need to transpose the counts in `dds`.

```{r}
library("PoiClaClu")
poisd <- PoissonDistance(t(counts(dds)))
```

We plot the heatmap in a Figure below.

```{r poisdistheatmap, fig.width = 6.1, fig.height = 4.5}
samplePoisDistMatrix <- as.matrix( poisd$dd )
rownames(samplePoisDistMatrix) <- paste( dds$dex, dds$cell, sep=" - " )
colnames(samplePoisDistMatrix) <- NULL
pheatmap(samplePoisDistMatrix,
         clustering_distance_rows = poisd$dd,
         clustering_distance_cols = poisd$dd,
         col = colors)
```

**Heatmap of sample-to-sample distances using the *Poisson Distance*.**

## PCA plot

Another way to visualize sample-to-sample distances is a
principal components analysis (PCA). In this ordination method, the
data points (here, the samples) are projected onto the 2D plane
such that they spread out in the two directions that explain most of
the differences (figure below). The x-axis is the direction that separates the data
points the most. The values of the samples in this direction are
written *PC1*. The y-axis is a direction (it must be *orthogonal* to
the first direction) that separates the data the second most. The
values of the samples in this direction are written *PC2*.
The percent of the total variance that is contained in the direction
is printed in the axis label. Note that these percentages do not add to
100%, because there are more dimensions that contain the remaining
variance (although each of these remaining dimensions will explain
less than the two that we see). 

```{r plotpca, fig.width=6, fig.height=4.5}
plotPCA(vsd, intgroup = c("dex", "cell"))
```

**PCA plot using the VST data.** Each unique combination of
treatment and cell line is given its own color.

Here, we have used the function *plotPCA* that comes with *DESeq2*.
The two terms specified by `intgroup` are the interesting groups for
labeling the samples; they tell the function to use them to choose
colors. We can also build the PCA plot from scratch using the
`r CRANpkg("ggplot2")` package [@Wickham2009Ggplot2].
This is done by asking the *plotPCA* function
to return the data used for plotting rather than building the plot.
See the *ggplot2* [documentation](http://docs.ggplot2.org/current/)
for more details on using *ggplot*.

```{r}
pcaData <- plotPCA(vsd, intgroup = c( "dex", "cell"), returnData = TRUE)
pcaData
percentVar <- round(100 * attr(pcaData, "percentVar"))
```

We can then use these data to build up a second plot in a figure below, specifying that the
color of the points should reflect dexamethasone treatment and the
shape should reflect the cell line.

```{r ggplotpca, fig.width=6, fig.height=4.5}
ggplot(pcaData, aes(x = PC1, y = PC2, color = dex, shape = cell)) +
  geom_point(size =3) +
  xlab(paste0("PC1: ", percentVar[1], "% variance")) +
  ylab(paste0("PC2: ", percentVar[2], "% variance")) +
  coord_fixed() +
  ggtitle("PCA with VST data")
```

**PCA plot using the VST values with custom *ggplot2* code.**
Here we specify cell line (plotting symbol) and dexamethasone treatment (color).

From the PCA plot, we see that the differences between cells (the
different plotting shapes) are considerable, though not stronger than the differences due to
treatment with dexamethasone (red vs blue color). This shows why it will be important to
account for this in differential testing by using a paired design
("paired", because each dex treated sample is paired with one
untreated sample from the *same* cell line). We are already set up for
this design by assigning the formula `~ cell + dex` earlier.

## PCA plot using Generalized PCA

Another technique for performing dimension reduction on data that is
not Normally distributed (e.g. over-dispersed count data) is 
*generalized principal component analysis*, or GLM-PCA, [@Townes2019] 
as implemented in the CRAN package `r CRANpkg("glmpca")`. This package
takes as input the count matrix, as well as the number of latent
dimensions to fit (here, we specify 2). As stated by @Townes2019:

> ...we propose the use of GLM-PCA, a generalization of
> PCA to exponential family likelihoods. GLM-PCA operates on raw counts,
> avoiding the pitfalls of normalization. We also demonstrate that applying PCA
> to deviance or Pearson residuals provides a useful and fast approximation to
> GLM-PCA.
```{r}
library("glmpca")
gpca <- glmpca(counts(dds), L=2)
gpca.dat <- gpca$factors
gpca.dat$dex <- dds$dex
gpca.dat$cell <- dds$cell
```

```{r glmpca, fig.width=6, fig.height=4.5}
ggplot(gpca.dat, aes(x = dim1, y = dim2, color = dex, shape = cell)) +
  geom_point(size =3) + coord_fixed() + ggtitle("glmpca - Generalized PCA")
```

## MDS plot

Another plot, very similar to the PCA plot, can be made using the 
*multidimensional scaling* (MDS) function in base R. This is useful when we
don't have a matrix of data, but only a matrix of distances. Here we
compute the MDS for the distances calculated from the *VST* data
and plot these in a figure below.

```{r mdsvst, fig.width=6, fig.height=4.5}
mds <- as.data.frame(colData(vsd))  %>%
         cbind(cmdscale(sampleDistMatrix))
ggplot(mds, aes(x = `1`, y = `2`, color = dex, shape = cell)) +
  geom_point(size = 3) + coord_fixed() + ggtitle("MDS with VST data")
```

**MDS plot using VST data.**

In a figure below we show the same plot for the *PoissonDistance*:

```{r mdspois, fig.width=6, fig.height=4.5}
mdsPois <- as.data.frame(colData(dds)) %>%
   cbind(cmdscale(samplePoisDistMatrix))
ggplot(mdsPois, aes(x = `1`, y = `2`, color = dex, shape = cell)) +
  geom_point(size = 3) + coord_fixed() + ggtitle("MDS with PoissonDistances")
```

**MDS plot using the *Poisson Distance*.**

<a id="de"></a>

# Differential expression analysis

## Running the differential expression pipeline

As we have already specified an experimental design when we created
the *DESeqDataSet*, we can run the differential expression pipeline on
the raw counts with a single call to the function *DESeq*:

```{r airwayDE}
dds <- DESeq(dds)
```

This function will print out a message for the various steps it
performs. These are described in more detail in the manual page for
*DESeq*, which can be accessed by typing `?DESeq`. Briefly these are:
the estimation of size factors (controlling for differences in the
sequencing depth of the samples), the estimation of
dispersion values for each gene, and fitting a generalized linear model.

A *DESeqDataSet* is returned that contains all the fitted
parameters within it, and the following section describes how to
extract out results tables of interest from this object.

## Building the results table

Calling *results* without any arguments will extract the estimated
log2 fold changes and *p* values for the last variable in the design
formula. If there are more than 2 levels for this variable, *results*
will extract the results table for a comparison of the last level over
the first level. The comparison is printed at the top of the output:
`dex trt vs untrt`.

```{r}
res <- results(dds)
res
```

We could have equivalently produced this results table with the
following more specific command. Because `dex` is the last variable in
the design, we could optionally leave off the `contrast` argument to extract
the comparison of the two levels of `dex`.

```{r}
res <- results(dds, contrast=c("dex","trt","untrt"))
```

As `res` is a *DataFrame* object, it carries metadata
with information on the meaning of the columns:

```{r}
mcols(res, use.names = TRUE)
```

The first column, `baseMean`, is a just the average of the normalized
count values, divided by the size factors, taken over all samples in the
*DESeqDataSet*.
The remaining four columns refer to a specific contrast, namely the
comparison of the `trt` level over the `untrt` level for the factor
variable `dex`. We will find out below how to obtain other contrasts.

The column `log2FoldChange` is the effect size estimate. It tells us
how much the gene's expression seems to have changed due to treatment
with dexamethasone in comparison to untreated samples.  This value is
reported on a logarithmic scale to base 2: for example, a log2 fold
change of 1.5 means that the gene's expression is increased by a
multiplicative factor of \(2^{1.5} \approx 2.82\).

Of course, this estimate has an uncertainty associated with it, which
is available in the column `lfcSE`, the standard error estimate for
the log2 fold change estimate.  We can also express the uncertainty of
a particular effect size estimate as the result of a statistical
test. The purpose of a test for differential expression is to test
whether the data provides sufficient evidence to conclude that this
value is really different from zero. *DESeq2* performs for each gene a
*hypothesis test* to see whether evidence is sufficient to decide
against the *null hypothesis* that there is zero effect of the treatment
on the gene and that the observed difference between treatment and
control was merely caused by experimental variability (i.e., the type
of variability that you can expect between different
samples in the same treatment group). As usual in statistics, the
result of this test is reported as a *p* value, and it is found in the
column `pvalue`. Remember that a *p* value indicates the probability
that a fold change as strong as the observed one, or even stronger,
would be seen under the situation described by the null hypothesis.

We can also summarize the results with the following line of code,
which reports some additional information, that will be covered in
later sections.

```{r}
summary(res)
```

Note that there are many genes with differential expression due to
dexamethasone treatment at the FDR level of 10%. This makes sense, as
the smooth muscle cells of the airway are known to react to
glucocorticoid steroids. However, there are two ways to be more strict
about which set of genes are considered significant:

* lower the false discovery rate threshold (the threshold on `padj` in
  the results table)
* raise the log2 fold change threshold from 0 using the `lfcThreshold`
  argument of *results*

If we lower the false discovery rate threshold, we should also
inform the `results()` function about it, so that the function can use this 
threshold for the optimal independent filtering that it performs:

```{r}
res.05 <- results(dds, alpha = 0.05)
table(res.05$padj < 0.05)
```

If we want to raise the log2 fold change threshold, so that we test
for genes that show more substantial changes due to treatment, we
simply supply a value on the log2 scale. For example, by specifying
`lfcThreshold = 1`, we test for genes that show significant effects of
treatment on gene counts more than doubling or less than halving,
because \(2^1 = 2\).

```{r}
resLFC1 <- results(dds, lfcThreshold=1)
table(resLFC1$padj < 0.1)
```

Sometimes a subset of the *p* values in `res` will be `NA` ("not
available"). This is *DESeq*'s way of reporting that all counts for
this gene were zero, and hence no test was applied. In addition, *p*
values can be assigned `NA` if the gene was excluded from analysis
because it contained an extreme count outlier. For more information,
see the outlier detection section of the *DESeq2* vignette.

If you use the results from an R analysis package in published
research, you can find the proper citation for the software by typing
`citation("pkgName")`, where you would substitute the name of the
package for `pkgName`.  Citing methods papers helps to support and
reward the individuals who put time into open source software for
genomic data analysis.

## Other comparisons

In general, the results for a comparison of any two levels of a
variable can be extracted using the `contrast` argument to
*results*. The user should specify three values: the name of the
variable, the name of the level for the numerator, and the name of the
level for the denominator. Here we extract results for the log2 of the
fold change of one cell line over another:

```{r}
results(dds, contrast = c("cell", "N061011", "N61311"))
```

There are additional ways to build results tables for certain
comparisons after running *DESeq* once.
If results for an interaction term are desired, the `name`
argument of *results* should be used. Please see the 
help page for the *results* function for details on the additional
ways to build results tables. In particular, the **Examples** section of
the help page for *results* gives some pertinent examples.

## Multiple testing

In high-throughput biology, we are careful to not use the *p* values
directly as evidence against the null, but to correct for
*multiple testing*. What would happen if we were to simply threshold
the *p* values at a low value, say 0.05? There are
`r sum(res$pvalue < .05, na.rm=TRUE)` genes with a *p* value
below 0.05 among the `r sum(!is.na(res$pvalue))` genes for which the
test succeeded in reporting a *p* value:

```{r sumres}
sum(res$pvalue < 0.05, na.rm=TRUE)
sum(!is.na(res$pvalue))
```

Now, assume for a moment that the null hypothesis is true for all
genes, i.e., no gene is affected by the treatment with
dexamethasone. Then, by the definition of the *p* value, we expect up to
5% of the genes to have a *p* value below 0.05. This amounts to
`r round(sum(!is.na(res$pvalue)) * .05 )` genes.
If we just considered the list of genes with a *p* value below 0.05 as
differentially expressed, this list should therefore be expected to
contain up to
`r round(sum(!is.na(res$pvalue)) * .05)` /
`r sum(res$pvalue < .05, na.rm=TRUE)` =
`r round(sum(!is.na(res$pvalue))*.05 / sum(res$pvalue < .05, na.rm=TRUE) * 100)`%
 false positives.

*DESeq2* uses the Benjamini-Hochberg (BH) adjustment [@Benjamini1995Controlling] as implemented in
the base R *p.adjust* function; in brief, this method calculates for
each gene an adjusted *p* value that answers the following question:
if one called significant all genes with an adjusted *p* value less than or
equal to this gene's adjusted *p* value threshold, what would be the fraction
of false positives (the *false discovery rate*, FDR) among them, in
the sense of the calculation outlined above? These values, called the
BH-adjusted *p* values, are given in the column `padj` of the `res`
object.

The FDR is a useful statistic for many high-throughput
experiments, as we are often interested in reporting or focusing on a
set of interesting genes, and we would like to put an upper bound on the
percent of false positives in this set. 

Hence, if we consider a fraction of 10% false positives acceptable,
we can consider all genes with an adjusted *p* value below 10% = 0.1
as significant. How many such genes are there?

```{r}
sum(res$padj < 0.1, na.rm=TRUE)
```

We subset the results table to these genes and then sort it by the
log2 fold change estimate to get the significant genes with the
strongest down-regulation:

```{r}
resSig <- subset(res, padj < 0.1)
head(resSig[ order(resSig$log2FoldChange), ])
```

...and with the strongest up-regulation:

```{r}
head(resSig[ order(resSig$log2FoldChange, decreasing = TRUE), ])
```

<a id="plots"></a>

# Plotting results

## Counts plot

A quick way to visualize the counts for a particular gene is to use
the *plotCounts* function that takes as arguments the
*DESeqDataSet*, a gene name, and the group over which to plot the
counts (figure below).

```{r plotcounts}
topGene <- rownames(res)[which.min(res$padj)]
plotCounts(dds, gene = topGene, intgroup=c("dex"))
```

**Normalized counts for a single gene over treatment group.**

We can also make custom plots using the *ggplot* function from the
`r CRANpkg("ggplot2")` package (figures below).

```{r ggplotcountsjitter, fig.width = 4, fig.height = 3}
library("ggbeeswarm")
geneCounts <- plotCounts(dds, gene = topGene, intgroup = c("dex","cell"),
                         returnData = TRUE)
ggplot(geneCounts, aes(x = dex, y = count, color = cell)) +
  scale_y_log10() +  geom_beeswarm(cex = 3)
```

```{r ggplotcountsgroup, fig.width = 4, fig.height = 3}
ggplot(geneCounts, aes(x = dex, y = count, color = cell, group = cell)) +
  scale_y_log10() + geom_point(size = 3) + geom_line()
```

**Normalized counts with lines connecting cell lines.**
Note that the *DESeq* test actually takes into account the cell line
effect, so this figure more closely depicts the difference being tested.

## MA-plot

An *MA-plot* [@Dudoit2002Statistical] provides a useful overview for
the distribution of the estimated coefficients in the model,
e.g. the comparisons of interest, across all genes.
On the y-axis, the "M" stands for "minus" --
subtraction of log values is equivalent to the log of the ratio -- and
on the x-axis, the "A" stands for "average". You may hear this plot
also referred to as a mean-difference plot, or a Bland-Altman plot.

Before making the MA-plot, we use the
*lfcShrink* function to shrink the log2 fold changes for the
comparison of dex treated vs untreated samples.
There are three types of shrinkage estimators in *DESeq2*, which are covered in the
[DESeq2 vignette](https://bioconductor.org/packages/release/bioc/vignettes/DESeq2/inst/doc/DESeq2.html).
Here we specify the *apeglm* method for shrinking coefficients, which
is good for shrinking the noisy LFC estimates while giving low bias
LFC estimates for true large differences [@Zhu2018]. To use *apeglm* we specify a
coefficient from the model to shrink, either by name or number as the
coefficient appears in `resultsNames(dds)`. 

```{r plotma}
library("apeglm")
resultsNames(dds)
res <- lfcShrink(dds, coef="dex_trt_vs_untrt", type="apeglm")
plotMA(res, ylim = c(-5, 5))
```

If it is necessary to specify a contrast not represented in
`resultsNames(dds)`, either of the other two shrinkage methods can be used,
or in some cases, re-factoring the relevant variables and running
`nbinomWaldTest` followed by `lfcShrink` is sufficient. See the DESeq2
vignette for more details.

**An MA-plot of changes induced by treatment.**
The log2 fold change for a particular
comparison is plotted on the y-axis and the average of the counts
normalized by size factor is shown on the x-axis.
Each gene is represented with a dot. Genes with an adjusted *p* value
below a threshold (here 0.1, the default) are shown in red.

The *DESeq2* package uses a Bayesian procedure to moderate (or
"shrink") log2 fold changes from genes with very low counts and highly
variable counts, as can be seen by the narrowing of the vertical
spread of points on the left side of the MA-plot. As shown above, the
*lfcShrink* function performs this operation.  For a detailed
explanation of the rationale of moderated fold changes, please see the
*DESeq2* paper [@Love2014Moderated].

If we had not used statistical moderation to shrink the noisy log2
fold changes, we would have instead seen the following plot:

```{r plotmaNoShr}
res.noshr <- results(dds, name="dex_trt_vs_untrt")
plotMA(res.noshr, ylim = c(-5, 5))
```

We can label individual points on the MA-plot as well. Here we use the
*with* R function to plot a circle and text for a selected row of the
results object. Within the *with* function, only the `baseMean` and
`log2FoldChange` values for the selected rows of `res` are used.

```{r plotmalabel}
plotMA(res, ylim = c(-5,5))
topGene <- rownames(res)[which.min(res$padj)]
with(res[topGene, ], {
  points(baseMean, log2FoldChange, col="dodgerblue", cex=2, lwd=2)
  text(baseMean, log2FoldChange, topGene, pos=2, col="dodgerblue")
})
```

Another useful diagnostic plot is the histogram of the *p* values
(figure below). This plot is best formed by excluding genes with very
small counts, which otherwise generate spikes in the histogram.

```{r histpvalue2}
hist(res$pvalue[res$baseMean > 1], breaks = 0:20/20,
     col = "grey50", border = "white")
```

**Histogram of *p* values for genes with mean normalized count larger than 1.**

## Gene clustering

In the sample distance heatmap made previously, the dendrogram at the
side shows us a hierarchical clustering of the samples. Such a
clustering can also be performed for the genes.  Since the clustering
is only relevant for genes that actually carry a signal, one usually
would only cluster a subset of the most highly variable genes. Here,
for demonstration, let us select the 20 genes with the highest
variance across samples. We will work with the VST data.

```{r}
library("genefilter")
topVarGenes <- head(order(rowVars(assay(vsd)), decreasing = TRUE), 20)
```

The heatmap becomes more interesting if we do not look at absolute
expression strength but rather at the amount by which each gene
deviates in a specific sample from the gene's average across all
samples. Hence, we center each genes' values across samples,
and plot a heatmap (figure below). We provide a *data.frame* that instructs the
*pheatmap* function how to label the columns.

```{r genescluster}
mat  <- assay(vsd)[ topVarGenes, ]
mat  <- mat - rowMeans(mat)
anno <- as.data.frame(colData(vsd)[, c("cell","dex")])
pheatmap(mat, annotation_col = anno)
```

**Heatmap of relative VST-transformed values across samples.**
Treatment status and cell line information are shown with colored bars
at the top of the heatmap.
Blocks of genes that covary across patients. Note that
a set of genes in the heatmap are separating the N061011
cell line from the others, and there is another set
of genes for which the dexamethasone treated samples have higher gene
expression. 

## Independent filtering

The MA plot highlights an important property of RNA-seq data.  For
weakly expressed genes, we have no chance of seeing differential
expression, because the low read counts suffer from such high Poisson
noise that any biological effect is drowned in the uncertainties from
the sampling at a low rate. We can also show this by examining the ratio of
small *p* values (say, less than 0.05) for genes binned by mean
normalized count. We will use the results table subjected to the threshold to show
what this looks like in a case when there are few tests with small *p*
value.

In the following code chunk, we create bins using the *quantile*
function, bin the genes by base mean using *cut*, rename the levels of
the bins using the middle point, calculate the ratio of *p* values
less than 0.05 for each bin, and finally plot these ratios (figure below).

```{r sensitivityovermean, fig.width=6}
qs <- c(0, quantile(resLFC1$baseMean[resLFC1$baseMean > 0], 0:6/6))
bins <- cut(resLFC1$baseMean, qs)
levels(bins) <- paste0("~", round(signif((qs[-1] + qs[-length(qs)])/2, 2)))
fractionSig <- tapply(resLFC1$pvalue, bins, function(p)
                          mean(p < .05, na.rm = TRUE))
barplot(fractionSig, xlab = "mean normalized count",
                     ylab = "fraction of small p values")
```

**The ratio of small *p* values for genes binned by mean normalized count.**
The *p* values are from a test of log2 fold change greater than 1
or less than -1. This plot demonstrates that genes with very low mean count
have little or no power, and are best excluded from testing.

At first sight, there may seem to be little benefit in filtering out
these genes. After all, the test found them to be non-significant
anyway. However, these genes have an influence on the multiple testing
adjustment, whose performance improves if such genes are removed. By
removing the low count genes from the input to the FDR
procedure, we can find more genes to be significant among those that
we keep, and so improved the power of our test. This approach is known
as *independent filtering*.

The *DESeq2* software automatically performs independent filtering
that maximizes the number of genes with adjusted *p* value
less than a critical value (by default, `alpha` is set to 0.1).  This
automatic independent filtering is performed by, and can be controlled
by, the *results* function.

The term *independent* highlights an important caveat. Such filtering
is permissible only if the statistic that we filter on (here the mean
of normalized counts across all samples) is independent of the
actual test statistic (the *p* value) under the null hypothesis.
Otherwise, the filtering would invalidate the
test and consequently the assumptions of the BH procedure.
The independent filtering software used inside
*DESeq2* comes from the `r Biocpkg("genefilter")` package, that
contains a reference to a paper describing the statistical foundation
for independent filtering [@Bourgon2010Independent].

## Independent Hypothesis Weighting

A generalization of the idea of *p* value filtering is to *weight*
hypotheses to optimize power. A Bioconductor
package, `r Biocpkg("IHW")` is available
that implements the method of *Independent Hypothesis Weighting*
[@Ignatiadis2016]. See the *DESeq2* package vignette for an example of
using *IHW* in combination with *DESeq2*. In particular, the following
(here, un-evaluated) code chunk can be used to perform IHW in lieu of
independent filtering described above.

```{r, eval=FALSE}
library("IHW")
res.ihw <- results(dds, filterFun=ihw)
```

<a id="annotate"></a>

# Annotating and exporting results

Our result table so far only contains the Ensembl gene
IDs, but alternative gene names may be more informative for
interpretation.  Bioconductor's annotation packages help with mapping
various ID schemes to each other.
We load the `r Biocpkg("AnnotationDbi")` package and the annotation package
`r Biocannopkg("org.Hs.eg.db")`:

```{r}
library("AnnotationDbi")
library("org.Hs.eg.db")
```

This is the organism annotation package ("org") for
*Homo sapiens* ("Hs"), organized as an *AnnotationDbi*
database package ("db"), using Entrez Gene IDs ("eg") as primary key.
To get a list of all available key types, use:

```{r}
columns(org.Hs.eg.db)
```

We can use the *mapIds* function to add individual columns to our results
table. We provide the row names of our results table as a key, and
specify that `keytype=ENSEMBL`. The `column` argument tells the
*mapIds* function which information we want, and the `multiVals`
argument tells the function what to do if there are multiple possible
values for a single input value. Here we ask to just give us back the
first one that occurs in the database.
To add the gene symbol and Entrez ID, we call *mapIds* twice.

```{r}
ens.str <- substr(rownames(res), 1, 15)
res$symbol <- mapIds(org.Hs.eg.db,
                     keys=ens.str,
                     column="SYMBOL",
                     keytype="ENSEMBL",
                     multiVals="first")
res$entrez <- mapIds(org.Hs.eg.db,
                     keys=ens.str,
                     column="ENTREZID",
                     keytype="ENSEMBL",
                     multiVals="first")
```

Now the results have the desired external gene IDs:

```{r}
resOrdered <- res[order(res$pvalue),]
head(resOrdered)
```

## Exporting results

You can easily save the results table in a CSV file that you can
then share or load with a spreadsheet program such as Excel. The call to
*as.data.frame* is necessary to convert the *DataFrame* object
(`r Biocpkg("IRanges")` package) to a *data.frame* object that can be
processed by *write.csv*. Here, we take just the top 100 genes for
demonstration.

```{r eval=FALSE}
resOrderedDF <- as.data.frame(resOrdered)[1:100, ]
write.csv(resOrderedDF, file = "results.csv")
```

A more sophisticated way for exporting results the Bioconductor
package `r Biocpkg("ReportingTools")` [@Huntley2013ReportingTools].
*ReportingTools* will automatically generate dynamic HTML documents,
including links to external databases using gene identifiers
and boxplots summarizing the normalized counts across groups.
See the *ReportingTools* vignettes for full details. The simplest
version of creating a dynamic *ReportingTools* report is performed
with the following code:

```{r eval=FALSE}
library("ReportingTools")
htmlRep <- HTMLReport(shortName="report", title="My report",
                      reportDirectory="./report")
publish(resOrderedDF, htmlRep)
url <- finish(htmlRep)
browseURL(url)
```

## Plotting fold changes in genomic space

If we have used the *tximeta* function to read in the quantification
data, then our *DESeqDataSet* object is built on top of ready-to-use
Bioconductor objects specifying the genomic coordinates of the genes. We
can therefore easily plot our differential expression results in
genomic space. While the *results* or *lfcShrink* functions by default
return a *DataFrame*, using the `format` argument, we can ask for
*GRanges* or *GRangesList* output (the latter is only possible if we
use the *addExons* function from the *tximeta* package upstream of
creating a *DESeqDataSet*).

```{r}
resGR <- lfcShrink(dds, coef="dex_trt_vs_untrt", type="apeglm", format="GRanges")
resGR
```

We need to add the symbol again for labeling the genes on the plot:

```{r}
ens.str <- substr(names(resGR), 1, 15)
resGR$symbol <- mapIds(org.Hs.eg.db, ens.str, "SYMBOL", "ENSEMBL")
```

We will use the `r Biocpkg("Gviz")` package for plotting the GRanges
and associated metadata: the log fold changes due to dexamethasone treatment.

```{r}
library("Gviz")
```

The following code chunk specifies a window of 1 million base pairs
upstream and downstream from the gene with the smallest *p* value.
We create a subset of our full results, for genes within the window.
We add the gene symbol as a name if the symbol exists and is not duplicated in
our subset.

```{r}
window <- resGR[topGene] + 1e6
strand(window) <- "*"
resGRsub <- resGR[resGR %over% window]
naOrDup <- is.na(resGRsub$symbol) | duplicated(resGRsub$symbol)
resGRsub$group <- ifelse(naOrDup, names(resGRsub), resGRsub$symbol)
```

We create a vector specifying if the genes in this subset had a low
value of `padj`.

```{r}
status <- factor(ifelse(resGRsub$padj < 0.05 & !is.na(resGRsub$padj),
                        "sig", "notsig"))
```

We can then plot the results using `r Biocpkg("Gviz")` functions
(figure below). We
create an axis track specifying our location in the genome, a track
that will show the genes and their names, colored by significance,
and a data track that will draw vertical bars showing the moderated
log fold change produced by *DESeq2*, which we know are only large
when the effect is well supported by the information in the counts.

```{r gvizplot}
options(ucscChromosomeNames = FALSE)
g <- GenomeAxisTrack()
a <- AnnotationTrack(resGRsub, name = "gene ranges", feature = status)
d <- DataTrack(resGRsub, data = "log2FoldChange", baseline = 0,
               type = "h", name = "log2 fold change", strand = "+")
plotTracks(list(g, d, a), groupAnnotation = "group",
           notsig = "grey", sig = "hotpink")
```

**log2 fold changes in genomic region surrounding the gene with smallest
  adjusted *p* value.** Genes highlighted in pink have adjusted *p*
  value less than 0.1.

<a id="batch"></a>

# Removing hidden batch effects

Suppose we did not know that there were different cell lines involved
in the experiment, only that there was treatment with
dexamethasone. The cell line effect on the counts then would represent
some hidden and unwanted variation that might be affecting
many or all of the genes in the dataset. We can use statistical
methods designed for RNA-seq from the `r Biocpkg("sva")` package
[@Leek2014Svaseq] or the `r Biocpkg("RUVSeq")` package
[@Risso2014Normalization] in Bioconductor to
detect such groupings of the samples, and then we can add these to the
*DESeqDataSet* design, in order to account for them. 

The *SVA* package uses the term *surrogate variables* for the estimated
variables that we want to account for in our analysis, while the RUV
package uses the terms *factors of unwanted variation* with the acronym "Remove Unwanted
Variation" explaining the package title. We first use *SVA* to find
hidden batch effects and then *RUV* following. 

## Using SVA with DESeq2

```{r}
library("sva")
```

Below we obtain a matrix of normalized counts for which the average count across
samples is larger than 1. As we described above, we are trying to
recover any hidden batch effects, supposing that we do not know the
cell line information. So we use a full model matrix with the
*dex* variable, and a reduced, or null, model matrix with only
an intercept term. Finally we specify that we want to estimate 2
surrogate variables. For more information read the manual page for the
*svaseq* function by typing `?svaseq`.

```{r}
dat  <- counts(dds, normalized = TRUE)
idx  <- rowMeans(dat) > 1
dat  <- dat[idx, ]
mod  <- model.matrix(~ dex, colData(dds))
mod0 <- model.matrix(~   1, colData(dds))
svseq <- svaseq(dat, mod, mod0, n.sv = 2)
svseq$sv
```

Because we actually do know the cell lines, we can see how well the
SVA method did at recovering these variables (figure below).

```{r svaplot}
par(mfrow = c(2, 1), mar = c(3,5,3,1))
for (i in 1:2) {
  stripchart(svseq$sv[, i] ~ dds$cell, vertical = TRUE, main = paste0("SV", i))
  abline(h = 0)
 }
```

**Surrogate variables 1 and 2 plotted over cell line.**
Here, we know the hidden source of variation (cell line), and
therefore can see how the SVA procedure is able to identify a source
of variation which is correlated with cell line.

Finally, in order to use SVA to remove any effect on the counts from
our surrogate variables, we simply add these two surrogate variables
as columns to the *DESeqDataSet* and then add them to the design:

```{r}
ddssva <- dds
ddssva$SV1 <- svseq$sv[,1]
ddssva$SV2 <- svseq$sv[,2]
design(ddssva) <- ~ SV1 + SV2 + dex
```

We could then produce results controlling for surrogate variables by
running *DESeq* with the new design.

## Using RUV with DESeq2

We can also use the *RUV* method in the *RUVSeq* package to detect the
hidden batch effects.

```{r}
library("RUVSeq")
```

We can use the `RUVg` function to estimate *factors of unwanted
variation*, analogous to *SVA*'s *surrogate variables*. A difference
compared to the *SVA* procedure above, is that we first would run
*DESeq* and *results* to obtain the p-values for the analysis without
knowing about the batches, e.g. just `~ dex`. Supposing that we have
this results table `res`, we then pull out a set of *empirical control
genes* by looking at the genes that do not have a small p-value.

```{r}
set <- newSeqExpressionSet(counts(dds))
idx  <- rowSums(counts(set) > 5) >= 2
set  <- set[idx, ]
set <- betweenLaneNormalization(set, which="upper")
not.sig <- rownames(res)[which(res$pvalue > .1)]
empirical <- rownames(set)[ rownames(set) %in% not.sig ]
set <- RUVg(set, empirical, k=2)
pData(set)
```

We can plot the factors estimated by *RUV*:

```{r ruvplot}
par(mfrow = c(2, 1), mar = c(3,5,3,1))
for (i in 1:2) {
  stripchart(pData(set)[, i] ~ dds$cell, vertical = TRUE, main = paste0("W", i))
  abline(h = 0)
 }
```

**Factors of unwanted variation plotted over cell line.**

As before, if we wanted to control for these factors, we simply
add them to the *DESeqDataSet* and to the design:

```{r}
ddsruv <- dds
ddsruv$W1 <- set$W_1
ddsruv$W2 <- set$W_2
design(ddsruv) <- ~ W1 + W2 + dex
```

We would then run *DESeq* with the new design to re-estimate the
parameters and results.

<a id="time"></a>

# Time course experiments

*DESeq2* can be used to analyze time course experiments, for example
to find those genes that react in a condition-specific manner over
time, compared to a set of baseline samples.
Here we demonstrate a basic time course analysis with the
 `r Biocexptpkg("fission")` data package,
which contains gene counts for an RNA-seq time course of fission
yeast [@Leong2014Global]. The yeast were exposed to oxidative stress, and half of the
samples contained a deletion of the gene *atf21*.
We use a design formula that models the strain difference at time 0,
the difference over time, and any strain-specific differences over
time (the interaction term `strain:minute`).

```{r}
library("fission")
data("fission")
ddsTC <- DESeqDataSet(fission, ~ strain + minute + strain:minute)
```

The following chunk of code performs a likelihood ratio test, where we remove
the strain-specific differences over time. Genes with small *p* values
from this test are those which at one or more time points after time
0 showed a strain-specific effect. Note therefore that this will not
give small *p* values to genes that moved up or down over time in the
same way in both strains.

```{r fissionDE}
ddsTC <- DESeq(ddsTC, test="LRT", reduced = ~ strain + minute)
resTC <- results(ddsTC)
resTC$symbol <- mcols(ddsTC)$symbol
head(resTC[order(resTC$padj),], 4)
```

This is just one of the tests that can be applied to time
series data. Another option would be to model the counts as a
smooth function of time, and to include an interaction term of the
condition with the smooth function. It is possible to build such a
model using spline basis functions within R, and another, more modern approach
is using Gaussian processes [@Tonner2016].

We can plot the counts for the groups over time using
`r CRANpkg("ggplot2")`, for the gene with the smallest adjusted *p* value,
testing for condition-dependent time profile and accounting for
differences at time 0 (figure below). Keep in mind that the interaction terms are the
*difference* between the two groups at a given time after accounting for
the difference at time 0.

```{r fissioncounts, fig.width=6, fig.height=4.5}
fiss <- plotCounts(ddsTC, which.min(resTC$padj), 
                   intgroup = c("minute","strain"), returnData = TRUE)
fiss$minute <- as.numeric(as.character(fiss$minute))
ggplot(fiss,
  aes(x = minute, y = count, color = strain, group = strain)) + 
  geom_point() + stat_summary(fun.y=mean, geom="line") +
  scale_y_log10()
```
**Normalized counts for a gene with condition-specific changes over time.**

Wald tests for the log2 fold changes at individual time points can be
investigated using the `test` argument to *results*:

```{r}
resultsNames(ddsTC)
res30 <- results(ddsTC, name="strainmut.minute30", test="Wald")
res30[which.min(resTC$padj),]
```

We can furthermore cluster significant genes by their profiles. We
extract a matrix of the shrunken log2 fold changes using the *coef* function:

```{r}
betas <- coef(ddsTC)
colnames(betas)
```

We can now plot the log2 fold changes in a heatmap (figure below). 

```{r fissionheatmap}
topGenes <- head(order(resTC$padj),20)
mat <- betas[topGenes, -c(1,2)]
thr <- 3 
mat[mat < -thr] <- -thr
mat[mat > thr] <- thr
pheatmap(mat, breaks=seq(from=-thr, to=thr, length=101),
         cluster_col=FALSE)
```

**Heatmap of log2 fold changes for genes with smallest adjusted *p* value.**
The bottom set of genes show strong induction of expression for the
baseline samples in minutes 15-60 (red boxes in the bottom left
corner), but then have slight differences for the mutant strain
(shown in the boxes in the bottom right corner).

<a id="ref"></a>

# Session information

As the last part of this document, we call the function *sessionInfo*,
which reports the version numbers of R and all the packages used in
this session. It is good practice to always keep such a record of this
as it will help to track down what has happened in case an R script
ceases to work or gives different results because the functions have
been changed in a newer version of one of your packages. By including
it at the bottom of a script, your reports will become more reproducible.

The session information should also *always*
be included in any emails to the
[Bioconductor support site](https://support.bioconductor.org) along
with all code used in the analysis.

```{r}
sessionInfo()
```

# References



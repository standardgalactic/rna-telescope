Telescope [![install with bioconda](https://img.shields.io/badge/install%20with-bioconda-brightgreen.svg?style=flat)](http://bioconda.github.io/recipes/telescope/README.html)
========

###### *Single locus resolution of* **T***ransposable* **ELE***ment expression.*

**Affiliations:**

+ [Computational Biology Institute](http://cbi.gwu.edu) at George Washington University
+ [Weill Cornell Medicine Division of Infectious Diseases](https://medicine.weill.cornell.edu/divisions-programs/infectious-diseases)

**Table of Contents:**

* [Installation](#installation)
* [Usage](#usage)
  * [`telescope sc assign`](#telescope-assign)
  * [`telescope sc resume`](#telescope-resume)
  * [`telescope bulk assign`](#telescope-assign)
  * [`telescope bulk resume`](#telescope-resume)
* [Output](#Output)
  * [Telescope report](#telescope-report)
  * [Updated SAM file](#updated-sam-file)
* [Version History](#version-history)

## Installation

**Recommended:**

Install Telescope using [bioconda](https://bioconda.github.io):

[![install with bioconda](https://img.shields.io/badge/install%20with-bioconda-brightgreen.svg?style=flat)](http://bioconda.github.io/recipes/telescope/README.html) 

```bash
conda install -c bioconda telescope
```

See [Getting Started](https://bioconda.github.io/user/install.html) for
instructions on setting up bioconda.


**Alternative:**

Use conda package manager to install dependencies, then 
use `pip` to install Telescope.

The following has been testing using miniconda3 on macOS and Linux (CentOS 7):

```bash
conda create -n telescope_env python=3.6 future pyyaml cython=0.29.7 \
  numpy=1.16.3 pandas=1.1.3 scipy=1.2.1 pysam=0.15.2 htslib=1.9 intervaltree=3.0.2

conda activate telescope_env
pip install git+git://github.com/mlbendall/telescope.git
telescope assign -h
```

## Testing

A BAM file (`alignment.bam`) and annotation (`annotation.gtf`) are included in
the telescope package for testing. The files are installed in the `data` 
directory of the package root. We've included a subcommand, `telescope [sc/bulk] test`,
to generate an example command line with the correct paths. 
For example, to generate an example command line for the bulk RNA-seq workflow:

```
telescope bulk test
```

The command can be executed using `eval`:

```
eval $(telescope bulk test)
```

The expected output to STDOUT includes the final log-likelihood, which was 
`95252.596293` in our tests. The test also outputs a report,
`telescope-telescope_report.tsv`, which can be compared to the report 
included in the `data` directory. NOTE: The precise values may be 
platform-dependent due to differences in floating point precision.

## Usage

### `telescope [sc/bulk] assign`

The `telescope [sc/bulk] assign` program finds overlapping reads between an alignment
(SAM/BAM) and an annotation (GTF) then reassigns reads using a statistical
model. This algorithm enables locus-specific quantification of transposable
element expression.

#### Basic usage

Basic usage requires a file containing read alignments to the genome and an 
annotation file with the transposable element gene model. The user should specify
whether the data was obtained from single-cell RNA sequencing (`sc`) or bulk
RNA sequencing (`bulk`). For example, to obtain single-cell TE counts from a BAM/SAM file:

```
telescope sc assign [samfile] [gtffile]
```

The alignment file must be in SAM or BAM format must be collated so that all 
alignments for a read pair appear sequentially in the file. Fragments should be
permitted to map to multiple locations (i.e. `-k` option in `bowtie2`).

The annotation file must be in GTF format and indicate the genomic regions that
represent transposable element transcripts. The transcripts are permitted to be
disjoint in order to exclude insertions of other element types. A collection of
valid transposable element gene models are available for download at 
[mlbendall/telescope_annotation_db](https://github.com/mlbendall/telescope_annotation_db).

#### Advanced usage

```
Input Options:

  samfile               Path to alignment file. Alignment file can be in SAM
                        or BAM format. File must be collated so that all
                        alignments for a read pair appear sequentially in the
                        file.
  gtffile               Path to annotation file (GTF format)
  --attribute ATTRIBUTE
                        GTF attribute that defines a transposable element
                        locus. GTF features that share the same value for
                        --attribute will be considered as part of the same
                        locus. (default: locus)
  --no_feature_key NO_FEATURE_KEY
                        Used internally to represent alignments. Must be
                        different from all other feature names. (default:
                        __no_feature)
  --ncpu NCPU           Number of cores to use. (Multiple cores not supported
                        yet). (default: 1)
  --tempdir TEMPDIR     Path to temporary directory. Temporary files will be
                        stored here. Default uses python tempfile package to
                        create the temporary directory. (default: None)

Reporting Options:

  --quiet               Silence (most) output. (default: False)
  --debug               Print debug messages. (default: False)
  --logfile LOGFILE     Log output to this file. (default: None)
  --outdir OUTDIR       Output directory. (default: .)
  --exp_tag EXP_TAG     Experiment tag (default: telescope)
  --updated_sam         Generate an updated alignment file. (default: False)
  
  Run Modes:

  --reassign_mode {exclude,choose,average,conf,unique}
                        Reassignment mode. After EM is complete, each fragment
                        is reassigned according to the expected value of its
                        membership weights. The reassignment method is the
                        method for resolving the "best" reassignment for
                        fragments that have multiple possible reassignments.
                        Available modes are: "exclude" - fragments with
                        multiple best assignments are excluded from the final
                        counts; "choose" - the best assignment is randomly
                        chosen from among the set of best assignments;
                        "average" - the fragment is divided evenly among the
                        best assignments; "conf" - only assignments that
                        exceed a certain threshold (see --conf_prob) are
                        accepted; "unique" - only uniquely aligned reads are
                        included. NOTE: Results using all assignment modes are
                        included in the Telescope report by default. This
                        argument determines what mode will be used for the
                        "final counts" column. (default: exclude)
  --use_every_reassign_mode (single-cell only)
                        Whether to output count matrices using every reassign mode. 
                        If specified, six output count matrices will be generated, 
                        corresponding to the six possible reassignment methods (all, exclude, 
                        choose, average, conf, unique). (default: False)
  --conf_prob CONF_PROB
                        Minimum probability for high confidence assignment.
                        (default: 0.9)
  --overlap_mode {threshold,intersection-strict,union}
                        Overlap mode. The method used to determine whether a
                        fragment overlaps feature. (default: threshold)
  --overlap_threshold OVERLAP_THRESHOLD
                        Fraction of fragment that must be contained within a
                        feature to be assigned to that locus. Ignored if
                        --overlap_method is not "threshold". (default: 0.2)
  --annotation_class {intervaltree,htseq}
                        Annotation class to use for finding overlaps. Both
                        htseq and intervaltree appear to yield identical
                        results. Performance differences are TBD. (default:
                        intervaltree)                    
  --stranded_mode {None, RF, R, FR, F}
                        Options for considering feature strand when assigning reads. 
                        If None, for each feature in the annotation, returns counts 
                        for the positive strand and negative strand. If not None, 
                        this argument specifies the orientation of paired end reads 
                        (RF - read 1 reverse strand, read 2 forward strand) and
                        single end reads (F - forward strand) with respect to the 
                        generating transcript. (default: None)
  --barcode_tag (single-cell only)
                        String specifying the name of the field in the BAM/SAM 
                        file containing the barcode for each read. (default: CB)
Model Parameters:

  --pi_prior PI_PRIOR   Prior on π. Equivalent to adding n unique reads.
                        (default: 0)
  --theta_prior THETA_PRIOR
                        Prior on θ. Equivalent to adding n non-unique reads.
                        (default: 200000)
  --em_epsilon EM_EPSILON
                        EM Algorithm Epsilon cutoff (default: 1e-7)
  --max_iter MAX_ITER   EM Algorithm maximum iterations (default: 100)
  --use_likelihood      Use difference in log-likelihood as convergence
                        criteria. (default: False)
  --skip_em             Exits after loading alignment and saving checkpoint
                        file. (default: False)
```


### `telescope [sc/bulk] resume`

The `telescope [sc/bulk] resume` program loads the checkpoint from a previous run and 
reassigns reads using a statistical model.

#### Basic usage

Basic usage requires a checkpoint file created by an earlier run of 
`telescope assign`. Useful if the run fails after the initial load:

```
telescope sc resume [checkpoint]
```

#### Advanced usage

Options are available for tuning the EM optimization, similar to 
`telescope [sc/bulk] assign`.

```
Input Options:

  checkpoint            Path to checkpoint file.

Reporting Options:

  --quiet               Silence (most) output. (default: False)
  --debug               Print debug messages. (default: False)
  --logfile LOGFILE     Log output to this file. (default: None)
  --outdir OUTDIR       Output directory. (default: .)
  --exp_tag EXP_TAG     Experiment tag (default: telescope)

Run Modes:

  --reassign_mode {exclude,choose,average,conf,unique}
                        Reassignment mode. After EM is complete, each fragment
                        is reassigned according to the expected value of its
                        membership weights. The reassignment method is the
                        method for resolving the "best" reassignment for
                        fragments that have multiple possible reassignments.
                        Available modes are: "exclude" - fragments with
                        multiple best assignments are excluded from the final
                        counts; "choose" - the best assignment is randomly
                        chosen from among the set of best assignments;
                        "average" - the fragment is divided evenly among the
                        best assignments; "conf" - only assignments that
                        exceed a certain threshold (see --conf_prob) are
                        accepted; "unique" - only uniquely aligned reads are
                        included. NOTE: Results using all assignment modes are
                        included in the Telescope report by default. This
                        argument determines what mode will be used for the
                        "final counts" column. (default: exclude)
  --use_every_reassign_mode 
                        Whether to output count matrices using every reassign mode. 
                        If specified, six output count matrices will be generated, 
                        corresponding to the six possible reassignment methods (all, exclude, 
                        choose, average, conf, unique). (default: False)
  --conf_prob CONF_PROB
                        Minimum probability for high confidence assignment.
                        (default: 0.9)

Model Parameters:

  --pi_prior PI_PRIOR   Prior on π. Equivalent to adding n unique reads.
                        (default: 0)
  --theta_prior THETA_PRIOR
                        Prior on θ. Equivalent to adding n non-unique reads.
                        (default: 0)
  --em_epsilon EM_EPSILON
                        EM Algorithm Epsilon cutoff (default: 1e-7)
  --max_iter MAX_ITER   EM Algorithm maximum iterations (default: 100)
  --use_likelihood      Use difference in log-likelihood as convergence
                        criteria. (default: False)
```
                        
## Output

Telescope has three main output files: the transcript counts estimated via EM (`telescope-TE_counts.tsv`), 
a statistical report of the run containing model parameters and additional information
(`telescope-stats_report.tsv`), and an updated SAM file (optional). 
The count file is most important for downstream differential
expression analysis. The updated SAM file is useful for downstream locus-specific analyses. 

### Telescope statistics report

In addition to outputting transcript counts,
bulk RNA-seq Telescope (`telescope bulk assign`) provides a more detailed 
statistical report of each read assignment run. 
The first line in the  report is a comment (starting with a “#”) that
contains information about the run such as the number of fragments processed,
number of mapped fragments, number of uniquely and ambiguously mapped 
fragments, and number of fragments mapping to the annotation. The total number
of mapped fragments may be useful for normalization. 

The rest of the report is a table with expression values for 
individual transposable element locations calculated using a variety of
reassignment methods, as well as estimated and initial model parameters.
Comparing the results from different assignment methods may shed light on the 
model's behaviour. The columns of the table are: 

+ `transcript` - Transcript ID, by default from "locus" field. See --attribute argument to use a different attribute.
+ `transcript_length` - Approximate length of transcript. This is calculated from the annotation, not the data, and is equal to the spanning length of the annotation minus any non-model regions.
+ `final_count` - Total number of fragments assigned to transcript after fitting the Telescope model. This is the column to use for downstream analysis that models data as negative binomial, i.e. DESeq2.
+ `final_conf` - Final confident fragments. The number of fragments assigned to transcript whose posterior probability exceeds a cutoff, 0.9 by default. Set this using the --conf_prob argument.
+ `final_prop` - Final proportion of fragments represented by transcript. This is the final estimate of the π parameter.
+ `init_aligned` - Initial number of fragments aligned to transcript. A given fragment will contribute +1 to each transcript that it is aligned to, thus the sum of this will be greater than the number of fragments if there are multimapped reads.
+ `unique_count` - Unique count. Number of fragments aligning uniquely to this transcript.
+ `init_best` - Initial number of fragments aligned to transcript that have the "best" alignment score for that fragment. Fragments that have the same best alignment score to multiple transcripts will contribute +1 to each transcript.
+ `init_best_random` - Initial number of fragments aligned to transcript that have the "best" alignment score for that fragment. Fragments that have the same best alignment score to multiple transcripts will be randomly assigned to one transcript.

For use with single-cell sequencing data (`telescope sc assign`), only model parameters are included
in the statistics report. If the user would like the tool to output count matrices generated
via each of the six assignment methods, they can use the `--use_every_reassign_mode`
option (`telescope sc assign [samfile] [gtffile] --use_every_reassign_mode`).

### Updated SAM file

The updated SAM file contains those fragments that has at least 1 initial 
alignment to a transposable element. The final assignment and probabilities are
encoded in the SAM tags:

+ `ZF:Z` Assigned Feature - The name of the feature that alignment is assigned to.
+ `ZT:Z` Telescope tag - A value of `PRI` indicates that this alignment is the
     best hit for the feature and is used in the likelihood calculations. 
     Otherwise the value will be `SEC`, meaning that another alignment to the
     same feature has a higher score.
+ `ZB:Z` Best Feature = The name(s) of the highest scoring feature(s) for the fragment.          
+ `YC:Z` Specifies color for alignment as R,G,B.
UCSC sanctioned tag, see documentation
[here.](http://genome.ucsc.edu/goldenpath/help/hgBamTrackHelp.html)
+ `XP:Z` Alignment probability - estimated posterior probability for this alignment.

## Version History

### v1.0.3.1

  + Checks that GTF feature type is "exon"
  + Skips GTF lines missing attribute (with warning)

### v1.0.3
  + Added cimport statements to calignment.pyx (MacOS bug fix)
  + Fixed warning about deprecated PyYAML yaml.load
  + Compatibility with intervaltree v3.0.2

### v1.0.2
  + Temporary files are written as BAM

### v1.0.1
  + Changed default `theta_prior` to 200,000

### v1.0
  + Removed dependency on `git`
  + Release version

### v0.6.5
  + Support for sorted BAM files
  + Parallel reading of sorted BAM files
  + Improved performance of EM
  + Improved memory-efficiency of spare matrix

#### v0.5.4.1
  + Fixed bug where random seed is out of range
  
#### v0.5.4
  + Added MIT license
  + Changes to logging/reporting
  
#### v0.5.3
  + Improvements to `telescope resume`

#### v0.5.2
  + Implemented checkpoint and `telescope resume`

#### v0.5.1
  + Refactoring Telescope class with TelescopeLikelihood
  + Improved memory usage

#### v0.4.2
  +  Subcommand option parsing class
  +  Cython for alignment parsing
  +  HTSeq as alternate alignment parser

#### v0.3.2
  +  Python3 compatibility  

#### v0.3
  + Implemented IntervalTree for Annotation data structure
  + Added support for annotation files where a locus may be non-contiguous.
  + Overlapping annotations with the same key value (locus) are merged
  + User can set minimum overlap criteria for assigning read to locus, default = 0.1

#### v0.2

  + Implemented checkpointing
  + Output tables as pickled objects
  + Changes to report format and output  

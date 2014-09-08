Visualizations
==============

A collection of next-gen sequencing visualisation scripts.

* [Count Biotypes](#count-biotypes)
	* Uses HTSeq to plot read overlaps with different feature biotype flags
* [Subsampled Gene Observations](#subsampled-gene-observations)
    * Group of scripts to plot the number of observed genes at varying sample
    subsampling proportions. Can give an impression of library complexity on
    a biological level.
* Bismark Addons
	* [Bismark Coverage Curves](#bismark-coverage-curves) - Plots the proportion of cytosines meeting increasing coverage thresholds
	* [Bismark Window Sizes](#bismark-window-sizes) - Plots the proportion of windows passing observation thresholds with increasing window sizes
* [Alignment Summaries](#alignment-reports)
	* Two scripts to parse log files containing alignment stats from bowtie,
		bowtie 2 or tophat and generate overview HTML reports

## Count Biotypes

This script takes an aligned BAM file and counts the overlaps with
different biotype flags within a GTF annotation file.

Annotation GTF (Gene Transfer Format) files can contain information about
coding sequences within the genome. In addition to specifying feature type
and position, GTF features can have associated annotation fields. One such
field is *biotype*, typically denoted by the `gene_type` or `biotype` flag
(see the [GENCODE format specifications of biotype flags](http://www.gencodegenes.org/gencode_biotypes.html).)

To get an overview of where reads from a next-generation sequencing library
are aligned within your reference genome, it can be interesting to annotate
overlaps with different types of features - for instance `rRNA` genes,
`protein_coding` genes and `miRNA` transcripts. This script does just that,
generating plots which show the frequency with which different biotype labels
are overlapped and how these overlaps are distributed throughout different
alignment lengths.

The script is written in Python and can be run on the command line or imported into another python script. Overlaps are measured using the [HTSeq library](http://www-huber.embl.de/users/anders/HTSeq/).

### Example output
The following plots were generated from a Total Small RNA run in Human cells,
accession [SRR1304304](http://www.ncbi.nlm.nih.gov/sra/?term=SRR1304304).

![Biotype overlaps](https://raw.githubusercontent.com/ewels/visualizations/master/examples/SRR1304304_trimmed_aligned_biotypeCounts.png)

![Biotype lengths](https://raw.githubusercontent.com/ewels/visualizations/master/examples/SRR1304304_trimmed_aligned_biotypeLengths.png)

![Biotype length percentages](https://raw.githubusercontent.com/ewels/visualizations/master/examples/SRR1304304_trimmed_aligned_biotypeLengthPercentages.png)


### Usage

On the command line:
```bash
python count_biotypes.py -g <annotation.gtf> <aligned_1.bam> .. <aligned_n.bam>
```

Within a python script:

```python
import count_biotypes
count_biotypes.count_biotypes(annotation_file_path_, input_bam_file_paths):
```

If importing, individual functions can be called for a more 
fine-grained approach:

```python
(ftrs, bt_cts, bt_lnths) = count_biotypes.parse_gtf_biotypes(annotation_file_path)
(counts, lengths, output) = count_biotype_overlaps (aligned_bam, ftrs, bt_cts, bt_lnths)
(bargraph_png_fn, bargraph_pdf_fn) = plot_bars(counts, fn_basename)
(hist_png_fn, hist_pdf_fn) = plot_epic_histogram (lengths, fn_basename)
```

#### Translations
To customise the plots, it is possible to "translate" biotype labels. A file
called `bt_translations.txt` comes with the script - each line can contain a
search pattern and a replacement label (tab separated). Search wildcards can be
included by using an asterisk (`*`). Multiple search strings can be given the
same biotype label to manually group biotypes together.

### Parameters

Arguments shown in order received by `count_biotypes()`.

Command Line Flag | `count_biotypes()` argument name | Description
----------------- | -------------------- | -----------
`-a`, `--genome-feature-file` | `annotation_file` | Required.<br>Path to annotation file.
`<input_bam_list>` | `input_bam_list` | Required.<br>List of paths to aligned BAM files.
`-g`, `--gene-feature` | `gene_feature` | Default: `gene`.<br>Feature type for genes.
`-t`, `--transcript-feature` | `transcript_feature` | Default: `exon`.<br>Feature type for transcribed regions (for which biotypes will be counted).
`-r`, `--rrna-biotype` | `rrna_biotype` | Default: `rRNA`.<br>Biotype label which identifies ribosomal RNA.
`-m`, `--mt-chr` | `mt_chr` | Default: `MT`.<br>Label for mitochondrial chromosome. `chr` will be automatically stripped. Case insensitive.
`-b`, `--biotype-flag` | `biotype_flag` | Default: `gene_type` (will also look for any flag containing `biotype`).<br>Name of annotation flag to collect biotype label from.
`-t`, `--genome-feature` | `feature_type` | Default: `exon`.<br>Type of feature to inspect within GTF file.
`-n`, `--num-lines` | `num_lines` | Default: 10 million.<br>Number of lines to read from aligned BAM file.
`-o`, `--no-overlap` | `no_overlap` | Default: False.<br>Include features with no feature overlap in plots.
`-c`, `--cols` | `equidistant_cols` | Default: False.<br>Plot graphs using equidistant colours to prevent duplicated label colours.
`-l`, `--log` | `log_level` | Default: info.<br>Specify the level of logging: debug, info or warning.
`-u`, `--log-output` | `log_output` | Default: stdout.<br>Log output filename.

### Dependencies

The script is written in Python. The following libraries are required:

* [HTSeq](http://www-huber.embl.de/users/anders/HTSeq/)
* [matplotlib](http://matplotlib.org/)
* [numpy](http://www.numpy.org/)
* argparse
* collections (defaultdict)
* logging
* os


---------------------------------------------------------------------------

## Subsampled Gene Observations

Software such as [Preseq](http://smithlabresearch.org/software/preseq/) can
show how sequencing library diversity changes with increasing sequencing 
depth. This tool is an attempt to generate a similar metric using a measurement
which is more biologically relevant for RNA-Sequencing datasets: the number
of different genes which have been observed _(default: FPKM > 0)_.

The package is comprised of four scripts which must be run separately:

* `submit_subsample_jobs.sh`: Subsample a set of aligned BAM files using [Picard](http://picard.sourceforge.net/command-line-overview.shtml#DownsampleSam)
* `submit_cufflinks_jobs.sh`: Count the FPKM gene counts using [Cufflinks](http://cufflinks.cbcb.umd.edu/)
* `count_aligned_reads.sh`: Count the aligned reads in the subsampled BAM files using [samtools](http://samtools.sourceforge.net/)
* `plot_observed_genes.py`: A Python script which takes this output and plots a graph.

**Note:** The first two scripts are currently written to work with our setup
in SciLifeLab, and will require modification to run on different systems.

### Example output
![Subsampled gene observations](https://raw.githubusercontent.com/ewels/visualizations/master/examples/subsampled_gene_observations.png)

### Step 1: Subsample the BAM files
```bash
bash submit_subsample_jobs.sh *.bam
```
This will set off SLURM sbatch jobs to create 9 subsampled files for each
input BAM file (10% to 90% in 10% steps). A soft-link is created for the
original file so that there is a file for the 100% step.

The script will check for existing files and skip that step if the target
file already exists. As such, if some jobs fail you can run the script
again to fill in the gaps.

Command Line Flag | Description
----------------- | ------------
`-l` | Directory for log files. Default: `./logs/`
`-o` | Directory for output. Default: `./downsampled/`

### Step 2: Cufflinks analysis
Once the subsampling is complete, cufflinks must be run on each file.
```bash
bash submit_cufflinks_jobs.sh -b <fasta reference> -g <gtf reference> *.bam
```
This will create jobs for the cufflinks analysis. As with the subsampling
script, the script will check for existing files and any where the target
file already exists. As such, if some jobs fail you can run the script
again to fill in the gaps.

Command Line Flag | Description
----------------- | ------------
`-b` | FASTA reference file. Required.
`-g` | GTF reference file. Required.
`-l` | Directory for log files. Default: `./logs/`
`-o` | Directory for output. Default: `./cufflinks/`
`-n` | Number of cores to use. Default: `1`

### Step 3: Count Reads _(optional)_
If you would like the plot the number of detected genes versus actual subsampled
read counts (as shown in the example above), you need to count the aligned
reads in each BAM file. This script uses samtools to count the reads in each
input and output a tab-delimited file with filename and read count.

**Note**: You can skip this step and just plot the x
axis as percentages instead of read counts - just omit the `-c` paramter when
running the plotting script.

```bash
bash count_aligned_reads.sh *.bam
```

Command Line Flag | Description
----------------- | ------------
`-o` | Directory for output file. Default: `./read_counts.txt`

### Step 4: Plotting
Finally, submit the directories of the completed cufflinks analysis to the
plotting script:
```bash
python plot_observed_genes.py *_cufflinksAnalysis/
```
The script will parse the directory names, assuming the structure
`<sample_name>_<subsample_proportion>`. Next, it will go through the
directories looking for a file called `genes.fpkm_tracking`. It will open
this and loop through each line (each gene) and count those where the FPKM
is greater than the specified threshold (default: `0`).

If a read counts file is specified with `-c`, the script will attempt to find
a read count for each subsample point and use this value on the x axis.

Finally, the script creates a plot using the proportions as the x axis.

Command Line Flag | `plot_observed_genes()` argument name | Description
----------------- | -------------------- | -----------
`<input directories>` | `input_dirs` | Required.<br>List of cufflinks results directories
`-f`, `--fpkm-cutoff` | `fpkm_cutoff` | Default: `0`<br> Cutoff at which to count genes as observed.
`-c`, `--read-counts` | `read_counts_fn` | Default: `None`<br> File containing BAM file read counts, used for x axis instead of percentages. See Step 3.
`-o`, `--output` | `output_fn` | Default: `gene_counts`<br>Plot output filename base. Default: `gene_counts.png` / `.pdf`
`-l`, `--log` | `log_level` | Default: `info`<br>Level of log messages to display. Can be `debug`, `info` or `warning`.
`-u`, `--log-output` | `log_output` | Default: `stdout`<br>Log output filename.

### Dependencies
The scripts are written in bash and Python. 
[Picard](http://picard.sourceforge.net/),
[Cufflinks](http://cufflinks.cbcb.umd.edu/) and
[samtools](http://samtools.sourceforge.net/) must be installed for the first
three steps.

The following Python libraries are required:

* [matplotlib](http://matplotlib.org/)
* argparse
* collections (defaultdict)
* logging
* os
* re



---------------------------------------------------------------------------


## Bismark Coverage Curves

[Bismark](http://www.bioinformatics.babraham.ac.uk/projects/bismark/) is a tool
used for aligning Bisfulfite-Sequencing libraries, giving information about
DNA methylation.

Amongst other things, Bismark can generate coverage reports which state the
number of observations made of each Cytosine. This script takes these reports
as input and plots the proportion of cytosines seen at increasing levels of 
fold coverage.

This is useful as when analysing BS-Seq data it's important to set a coverage
threshold to avoid low observations skewing percentage information. These plots
help to choose an appropriate cut-off.

Additional options allow you to interrogate coverage on different reference
strands and within regions of interest, as specified by a BED file.

### Example Output
![Bismark Coverage Curves Plot](https://raw.githubusercontent.com/ewels/visualizations/master/examples/coverageStats.png)

See additional [text output](https://raw.githubusercontent.com/ewels/visualizations/master/examples/coverageStats.txt)


### Usage

```bash
perl bismark_coverage_curves.pl <coverage_file.cov>
```

For nicer fonts, download the [OpenSans-Regular.ttf](https://github.com/ewels/visualizations/raw/master/OpenSans-Regular.ttf)
font into the same directory as the script. Font is from [Google Fonts](https://www.google.com/fonts/specimen/Open+Sans).

### Parameters

This script is run on the command line. The following commands control how
it runs.

Command Line Flag | Description
----------------- | -----------
`--regions <regions.bed>` | Default: None<br>Supply a BED file with regions of interest. The script will show coverage inside and outside these regions
`--stranded` | Default: No.<br>Split the report up into forward and reverse strands
`--min_cov` | Default: `0x`.<br>The minimum coverage limit to consider / plot
`--max_cov` | Default: `15x`; `50x` if `--regions` is set.<br>The maximum coverage limit to consider / plot
`--binsize` | Default: `1`.<br>The coverage bin size to use - what size steps to use between `--min_cov` and `--max_cov`
`--numlines` | Default: `1000000`.<br>Number of lines to process. More lines gives more accuracy but takes longer to run. Note: if the imput is sorted and your sample biased it's a good idea to specify a large number.
`--append` | Default: `_coverageStats.txt`.<br>String to append to results filenames
`--quiet` | Suppress status messages
`--help` | Print help message

### Dependencies

The script is written in Perl and run on the command line. The following
core Perl modules are required for generating the numbers:

* [Getopt::Long](http://perldoc.perl.org/Getopt/Long.html)
* [POSIX](http://perldoc.perl.org/POSIX.html)
* [FindBin](http://perldoc.perl.org/FindBin.html)

To plot the graphs, you'll also need the following modules:

* [GD::Graph](http://search.cpan.org/dist/GDGraph/Graph.pm) (lines and colour)
* [GD::Image](http://search.cpan.org/dist/GD/GD.pm)




---------------------------------------------------------------------------


## Bismark Window Sizes

[Bismark](http://www.bioinformatics.babraham.ac.uk/projects/bismark/) is a tool
used for aligning Bisfulfite-Sequencing libraries, giving information about
DNA methylation.

In addition to setting coverage thresholds for individual cytosines, it can
help to set thresholds for the number of different cytosines to be counted
within each window. This script takes coverage reports as input and plots the
percentage of windows retained at increasing window sizes.

Additional options allow you to restrict included cytosines to a specific
reference strand, define a coverage threshold for each cytosine for it to
be considered, the number of different cytosines passing the coverage
threshold for a window to be counted as well as restricting the 
windows to those overlapping regions of interest, as specified by a BED file.

### Example Output
![Bismark Window Sizes Plot](https://raw.githubusercontent.com/ewels/visualizations/master/examples/windowSizes_wholeGenome.png)

![Bismark Window Sizes Plot](https://raw.githubusercontent.com/ewels/visualizations/master/examples/windowSizes_roi.png)

See additional text output: [first plot](https://raw.githubusercontent.com/ewels/visualizations/master/examples/windowSizes_wholeGenome.txt), [second plot](https://raw.githubusercontent.com/ewels/visualizations/master/examples/windowSizes_roi.txt)

### Usage

```bash
perl bismark_window_sizes.pl <coverage_file.cov>
```

For nicer fonts, download the [OpenSans-Regular.ttf](https://github.com/ewels/visualizations/raw/master/OpenSans-Regular.ttf)
font into the same directory as the script. Font is from [Google Fonts](https://www.google.com/fonts/specimen/Open+Sans).

### Parameters

This script is run on the command line. The following commands control how
it runs.

Command Line Flag | Description
----------------- | -----------
`--regions <regions.bed>` | Default: None<br>Supply a BED file with regions of interest. Only reads and windows overlapping these regions will be considered.
`--stranded <for / rev>` | Default: both.<br>Consider reads on only one reference strand
`--coverage` | Default: `10x`.<br>Minumum number of observations required to count a Cytosine
`--min_counts <comma separated integers>` | Default: `1,2,3,4,5,10`.<br>List of count thresholds to use - how many different cytosines must be seen within a window for it to pass
`--window_sizes <comma separated integers, bp>` | Default: `100bp,200bp,300bp,400bp,500bp,1kbp,1.5kbp,2kbp,3kbp,4kbp,5kbp,10kbp,20kbp,30kbp,40kbp,50kbp,100kbp,200kbp,300kbp,400kbp,500kbp,1mbp,2mbp`.<br>Window sizes to use. Specify in base pairs.
`--append` | Default: `_coverageStats.txt`.<br>String to append to results filenames
`--quiet` | Suppress status messages
`--help` | Print help message

### Dependencies

The script is written in Perl and run on the command line. The following
core Perl modules are required for generating the numbers:

* [Getopt::Long](http://perldoc.perl.org/Getopt/Long.html)
* [POSIX](http://perldoc.perl.org/POSIX.html)
* [FindBin](http://perldoc.perl.org/FindBin.html)

To plot the graphs, you'll also need the following modules:

* [GD::Graph](http://search.cpan.org/dist/GDGraph/Graph.pm) (linespoints and colour)
* [GD::Image](http://search.cpan.org/dist/GD/GD.pm)




---------------------------------------------------------------------------


## Alignment Reports

[Bowtie](http://bowtie-bio.sourceforge.net/index.shtml), 
[Bowtie 2](http://bowtie-bio.sourceforge.net/bowtie2/index.shtml) and
[Tophat](http://ccb.jhu.edu/software/tophat/index.shtml) are
commonly used next-gen sequencing read aligners.

It is quite usual to align many different input files in a batch. These
aligners generate statistics about how successful each alignment
is but these are usually held in text format, buried within a log file.
As such, failed alignments can be easy to miss and it is difficult to get
a quick overview as to the success of the batch.

These two Perl scripts simply search through a collection of log files
(one per alignment) and generate overview reports, showing a plot of
how many reads were aligned, how many were discarded due to multiple
mapping (if applicable) and how many failed to align. This plot can
be switched to display percentages instead of read counts.

The reports are generated in HTML with all assets embedded. As such they
can be shared as single files and viewed in any web browser.


### Usage

	bowtie_report.pl <*.log>
	tophat_report.pl <*.log>


### Example Output

![Bowtie Alignments](https://raw.githubusercontent.com/ewels/visualizations/master/examples/bowtie_align_screenshot.png)

See the example report here: [bowtie_report.html](https://rawgit.com/ewels/visualizations/master/examples/bowtie_report.html)

### Parameters

None. These scripts are quite basic. They generate a file in the current directory called `bowtie_report.html` or `tophat_report.html`, overwriting
any that exist with the same name.

**Note**: Each input log file must contain alignment stats for only one alignment.

### Dependencies

These scripts are written in Perl and run on the command line. They do not use
any additional Perl modules. The generated reports use
[jQuery](http://jquery.com/) (embedded) and
[HighCharts](http://www.highcharts.com/)
(embedded) to render the plots.

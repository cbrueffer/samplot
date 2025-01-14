[![Build Status](https://travis-ci.org/ryanlayer/samplot.svg?branch=master)](https://travis-ci.org/ryanlayer/samplot)

<center><img src="/doc/imgs/samplot_logo_v5.png" width="300"/></center>

<center><img src="/doc/imgs/montage.jpg" width="100%"/></center>

`samplot` is a command line tool for rapid, multi-sample structural variant
visualization. `samplot` takes SV coordinates and bam files and produces
high-quality images that highlight any alignment and depth signals that
substantiate the SV.

## Usage
```
Usage: samplot.py [options]

Options:
  -h, --help            show this help message and exit
  --marker_size=MARKER_SIZE
                        Size of marks on pairs and splits (default 3)
  -n TITLES             Space-delimited list of plot titles. Use quote marks to include spaces (i.e. \"plot 1\" \"plot 2\")"
  -r REFERENCE          Reference file for CRAM
  -z Z                  Number of stdevs from the mean (default 4)
  -b BAMS               Bam file names (CSV)
  -o OUTPUT_FILE        Output file name
  -s START              Start range
  -e END                End range
  -c CHROM              Chromosome range
  -w WINDOW             Window size (count of bases to include), default(0.5 *
                        len)
  -d MAX_DEPTH          Max number of normal pairs to plot
  -t SV_TYPE            SV type
  -T TRANSCRIPT_FILE    GFF of transcripts
  -A ANNOTATION_FILE    Space-delimited list of bed.gz tabixed files of annotations (such as repeats, mappability, etc.)
  -a                    Print commandline arguments
  -H PLOT_HEIGHT        Plot height
  -W PLOT_WIDTH         Plot width
  -j                    Create only the json file, not the image plot
  --long_read=LONG_READ
                        Min length of a read to be a long-read (default 1000)
  --common_insert_size  Set common insert size for all plots
```

## Installing
Since samplot runs as a Python script, the only requirements to use it are a working version of Python (2 or 3) and the required Python [libraries](https://raw.githubusercontent.com/ryanlayer/samplot/master/requirements.txt). Installation of these libraries can be performed easily by using conda:
```
conda install -y --file https://raw.githubusercontent.com/ryanlayer/samplot/master/requirements.txt
```

If you have issues with `pysam`, then you may need to update your conda channels:
```
conda config --add channels r
conda config --add channels bioconda
=======
```

All of these libraries are also available from [pip](https://pypi.python.org/pypi/pip). 

You can download samplot by cloning the git repository:
```
git clone https://github.com/ryanlayer/samplot.git
```
No other installation is required.

## Examples: 

Samplot requires either BAM files or CRAM files as primary input. If you use
CRAM, you'll also need a reference genome like one used the the 1000 Genomes
Project
(ftp://ftp-trace.ncbi.nih.gov/1000genomes/ftp/technical/reference/human_g1k_v37.fasta.gz).

### Basic use case
Using data from NA12878, NA12889, and NA12890 in the 
[1000 Genomes Project](http://www.internationalgenome.org/about), we will
inspect a possible deletion in NA12878 at 4:115928726-115931880 with respect
to that same region in two unrelated samples NA12889 and NA12890.

The following command will create an image of that region:
```
time samplot/src/samplot.py \
    -n NA12878 NA12889 NA12890 \
    -b samplot/test/data/NA12878_restricted.bam \
      samplot/test/data/NA12889_restricted.bam \
      samplot/test/data/NA12890_restricted.bam \
    -o 4_115928726_115931880.png \
    -c chr4 \
    -s 115928726 \
    -e 115931880 \
    -t DEL

real    0m9.450s
user    0m9.199s
sys     0m0.217s
```

The arguments used above are:

`-n` The names to be shown for each sample in the plot

`-b` The BAM/CRAM files of the samples (space-delimited)

`-o` The name of the output file containing the plot

`-c` The chromosome of the region of interest

`-s` The start location of the region of interest

`-e` The end location of the region of interest

`-t` The type of the variant of interest

This will create an image file named `4_115928726_115931880.png`, shown below:

<img src="/doc/imgs/4_115928726_115931880.png">

### Downsampling "normal" pairs

The runtime of `samplot` can be reduced by only plotting a portion of the concordant 
pair-end reads (+/- strand orientation, within z s.d. of the mean insert size where z 
is a command line option the defaults to 4). If we rerun the prior example, but only plot
a random sampling of 100 normal pairs we get a similar result 3.6X faster.

```
time samplot/src/samplot.py \
    -n NA12878 NA12889 NA12890 \
    -b samplot/test/data/NA12878_restricted.bam \
      samplot/test/data/NA12889_restricted.bam \
      samplot/test/data/NA12890_restricted.bam \
    -o 4_115928726_115931880.d100.png \
    -c chr4 \
    -s 115928726 \
    -e 115931880 \
    -t DEL \
    -d 100

real    0m2.621s
user    0m2.466s
sys     0m0.124s
```

<img src="/doc/imgs/4_115928726_115931880.d100.png">


### Gene and other genomic feature annotations

Gene annotations (tabixed, gff3 file) and genome features (tabixed, bgzipped, bed file) can be 
included in the plots.

Get the gene annotations:
```
wget ftp://ftp.ensembl.org/pub/grch37/release-84/gff3/homo_sapiens/Homo_sapiens.GRCh37.82.gff3.gz
bedtools sort -i Homo_sapiens.GRCh37.82.gff3.gz \
| bgzip -c > Homo_sapiens.GRCh37.82.sort.gff3.gz
tabix Homo_sapiens.GRCh37.82.sort.gff3.gz
```

Get genome annotations, in this case Repeat Masker tracks and a mappability track:
```
wget http://hgdownload.cse.ucsc.edu/goldenpath/hg19/encodeDCC/wgEncodeMapability/wgEncodeDukeMapabilityUniqueness35bp.bigWig
bigWigToBedGraph wgEncodeDukeMapabilityUniqueness35bp.bigWig wgEncodeDukeMapabilityUniqueness35bp.bed
bgzip wgEncodeDukeMapabilityUniqueness35bp.bed
tabix wgEncodeDukeMapabilityUniqueness35bp.bed.gz

curl http://hgdownload.soe.ucsc.edu/goldenPath/hg19/database/rmsk.txt.gz \
| bgzip -d -c \
| cut -f 6,7,8,13 \
| bedtools sort -i stdin \
| bgzip -c > rmsk.bed.gz
tabix rmsk.bed.gz
```

Plot:
```
samplot/src/samplot.py \
    -n NA12878 NA12889 NA12890 \
    -b samplot/test/data/NA12878_restricted.bam \
      samplot/test/data/NA12889_restricted.bam \
      samplot/test/data/NA12890_restricted.bam \
    -o 4_115928726_115931880.d100.genes_reps_map.png \
    -c chr4 \
    -s 115928726 \
    -e 115931880 \
    -t DEL \
    -d 100 \
    -T Homo_sapiens.GRCh37.82.sort.gff3.gz \
    -A rmsk.bed.gz wgEncodeDukeMapabilityUniqueness35bp.bed.gz

real    0m2.784s
user    0m2.633s
sys 0m0.129s
```

<img src="/doc/imgs/4_115928726_115931880.d100.genes_reps_map.png">

## Generating images from a VCF file
To plot images from structural variant calls in a VCF file, use samplot's
`samplot_vcf.py` script. This accepts a VCF file and the BAM files of samples
you wish to plot, outputting images and the index for a web page for review. 

### Usage
```
$ python samplot/src/samplot_vcf.py -h
usage: note that additional arguments are passed through to samplot.py
       [-h] [--vcf VCF] [-d OUT_DIR] [--ped PED] [--dn_only]
       [--min_call_rate MIN_CALL_RATE] [--filter FILTER]
       [-O {png,pdf,eps,jpg}] [--max_hets MAX_HETS]
       [--min_entries MIN_ENTRIES] [--max_entries MAX_ENTRIES]
       [--max_mb MAX_MB] [--important_regions IMPORTANT_REGIONS] -b BAMS
       [BAMS ...]

optional arguments:
  -h, --help            show this help message and exit
  --vcf VCF, -v VCF     VCF file containing structural variants
  -d OUT_DIR, --out-dir OUT_DIR
                        path to write output PNGs
  --ped PED             path ped (or .fam) file
  --dn_only             plots only putative de novo variants (PED file
                        required)
  --min_call_rate MIN_CALL_RATE
                        only plot variants with at least this call-rate
  --filter FILTER       simple filter that samples must meet. Join multiple
                        filters with '&' and specify --filter multiple times
                        for 'or' e.g. DHFFC < 0.7 & SVTYPE = 'DEL'
  -O {png,pdf,eps,jpg}, --output_type {png,pdf,eps,jpg}
                        type of output figure
  --max_hets MAX_HETS   only plot variants with at most this many
                        heterozygotes
  --min_entries MIN_ENTRIES
                        try to include homref samples as controls to get this
                        many samples in plot
  --max_entries MAX_ENTRIES
                        only plot at most this many heterozygotes
  --max_mb MAX_MB       skip variants longer than this many megabases
  --important_regions IMPORTANT_REGIONS
                        only report variants that overlap regions in this bed
                        file
  -b BAMS [BAMS ...], --bams BAMS [BAMS ...]
                        Space-delimited list of BAM/CRAM file names
```

`samplot_vcf.py` can be used to quickly apply some basic filters to variants. Filters are applied via the `--filter` argument, which may be repeated as many times as desired. Each expression specified with the `--filter` option is applied separately in an OR fashion, which `&` characters may be used within a statement for AND operations.

### Example:
```
python samplot_vcf.py \
    --filter "SVTYPE == 'DEL' & SU >= 8" \
    --filter "SVTYPE == 'INV' & SU >= 5" \
    --vcf example.vcf\
    -d test/\
    -O png\
    --important_regions regions.bed\
    -b example.bam > samplot_commands.sh
```
This example will create a directory named test (in the current working directory). A file named `index.html` will be created inside that directory. Samplot commands will be printed out for the creation of plots for all samples/variants that pass the above filters, assuming that the `samplot.py` script is in the same directory as the `samplot_vcf.py` script.

**Filters:** The above filters will remove all samples/variants from output except:
* `DUP` variants with at least `SU` of 8
* `INV` variants with `SU` of at least 5

The specific `FORMAT` fields available in your VCF file may be different. I recommend SV VCF annotation with [duphold](https://github.com/brentp/duphold) by [brentp](https://github.com/brentp).

For more complex expression-based VCF filtering, try brentp's [slivar](https://github.com/brentp/slivar), which provides similar but more broad options for filter expressions.

**Region restriction.** Variants can also be filtered by overlap with a set of region (for example, gene coordinates for genes correlated with a disease). The `important_regions` argument provides a BED file of such regions for this example.

**Filtering for de novo SVs** 
Using a [PED](https://gatkforums.broadinstitute.org/gatk/discussion/7696/pedigree-ped-files) file with `samplot_vcf.py` allows filtering for variants that may be spontaneous/de novo variants. This filter is a simple Mendelian violation test. If a sample 1) has valid parent IDs in the PED file, 2) has a non-homref genotype (1/0, 0/1, or 1/1 in VCF), 3) passes filters, and 4) both parents have homref genotypes (0/0 in VCF), the sample may have a de novo variant. Filter parameters are not applied to the parents. The sample is plotted along with both parents, which are labeled as father and mother in the image. 

Example call with the addition of a PED file:

<pre>
python samplot_vcf.py \
    --filter "SVTYPE == 'DEL' & SU >= 8" \
    --filter "SVTYPE == 'INV' & SU >= 5" \
    --vcf example.vcf\
    -d test/\
    -O png\
    <b>--ped family.ped\</b>
    --important_regions regions.bed\
    -b example.bam > samplot_commands.sh
</pre>

**Additional notes.** 
* Variants where fewer than 95% of samples have a call (whether reference or alternate) will be excluded by default. This can be altered via the command-line argument `min_call_rate`.
* If you're primarily interested in rare variants, you can use the `max_hets` filter to remove variants that appear in more than `max_hets` samples.
* Large variants can now be plotted easily by samplot through use of `samplot.py`'s `zoom` argument. However, you can still choose to only plot variants larger than a given size using the `max_mb` argument. The `zoom` argument takes an integer parameter and shows only the intervals within +/- that parameter on either side of the breakpoints. A dotted line connects the ends of the variant call bar at the top of the window, showing that the region between breakpoint intervals is not shown.
* By default, if fewer than 6 samples have a variant and additional homref samples are given, control samples will be added from the homref group to reach a total of 6 samples in the plot. This number may be altered using the `min_entries` argument.
* Arguments that are optional in `samplot.py` can by given as arguments to `samplot_vcf.py`. They will be applied to each image generated.


#### CRAM inputs
Samplot also support CRAM input, which requires a reference fasta file for
reading as noted above. Notice that the reference file is not included in this
repository due to size. This time we'll plot an interesting duplication at
X:101055330-101067156.

```
samplot/src/samplot.py \
    -n NA12878 NA12889 NA12890 \
    -b samplot/test/data/NA12878_restricted.cram \
      samplot/test/data/NA12889_restricted.cram \
      samplot/test/data/NA12890_restricted.cram \
    -o cramX_101055330_101067156.png 
    -c chrX \
    -s 101055330 \
    -e 101067156 \
    -t DUP \
    -r hg19.fa
```


The arguments used above are the same as those used for the basic use case, with the addition of the following:

`-r` The reference file used for reading CRAM files

#### Plotting without the SV 
Samplot can also plot genomic regions that are unrelated to an SV. If you do
not pass the SV type option (`-t`) then the top SV bar will go away and only
the region that is given by `-c` `-s` and `-e` will be displayed.

#### Long read (Oxford nanopore and PacBio) and linked read support
Any alignment that is longer than 1000 bp are treated as a longread, and
the plot design will focus on aligned regions and gaps. Aligned regions are in orange, and gaps follow the same DEL/DUP/INV color code used for short reads. The height of the alignment is based on the size of its largest gap.

<img src="/doc/imgs/longread_del.png">

If the bam file has an MI tag, then the reads will be treated as linked reads.
The plots will be similar to short read plots, but all alignments with the same MI is plotted at the same height according to alignment with the largest gap in the group. A green line connects all alignments in a group.

<img src="/doc/imgs/linkedread_del.png">

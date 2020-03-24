# SARS-CoV-2 viral genomics

The purpose of this GitHub page is to share data, analyses, and results related to the analysis of genomic variation in the SARS-CoV-2 genome. **This work is on-going - please feel free to reach out with corrections, additions, or questions.**

This analysis was performed primarily by Matt Olm (<mattolm@stanford.edu>) in Justin Sonnenburg's lab at Stanford University and Alex Crits-Christoph (<crist-christoph@berkeley.edu>) in Jill Banfield's lab at University of California, Berkeley and analysis is broken up into the following major sections:

## [Data and Results Summary](#Data-and-Results-Summary-1)

An overview of our results and links to quickly download some of the data parsed and generated during this study, including a sequence alignment of all publicly available genomes, a table of gene annotations, and a table SRA run locations of raw reads.

## [Interpatient variation](#interpatient-variation-1)

Analysis based on comparing the covid19 genomes assembled from different patients. This type of analysis is typically done to understand outbreak clusters and how the genome evolves over time.

## [Intrapatient variation](#Intrapatient-variation-1)

Analysis based on comparing the differences between viral genomes that are generated with a single individual during infection.

## [Citations and acknowledgements](#citations-and-acknowledgements-1)

This work completely depends on the scientists and universities that originally sequenced these genomes and made their data publicly available.

Note: This method requires access to the raw sequencing reads generated when sequencing covid19 genomes. Usually these reads are used to generate a viral genome, the genome is deposited into a public database, and the raw reads are never uploaded publicly. **If you are involved in covid-19 genome sequencing efforts, please consider uploading the raw reads as well so that analyses like this can continue.**

# Data and Results Summary

This is where 3 bullet points of results will be (with links to their figure below) and links to the most key datatables that may be useful for others



# Interpatient variation

### Data processing

We are working with viral genomes from NCBI's data hub:
https://www.ncbi.nlm.nih.gov/labs/virus/vssi/#/virus?SeqType_s=Nucleotide&VirusLineage_ss=SARS-CoV-2,%20taxid:2697049

But a similar analysis could be done on the larger collection of sequences available in [GISAID](gisaid.org). Currently there are about ~170 genomes in NCBI and over 800 in GISAID.

A FASTA file of all currently available NCBI sequences can be found in `./interpatient/ncbi_mar20.fna`.

Filtering:
The script `./interpatient/filter_seqs.py` will perform some quality checks on the NCBI sequences. Firstly, several of the sequences are too short, and do not represent full viral genomes. We filter out all sequences < 29 Kb, as the viral genome is ~30 Kb. There are also several sequences that are too short in GISAID.

This script then filters out viral genomes that are too divergent to represent accurate genomes from the 2019 SARS-CoV-2 pandemic. This second step requires the program `fastANI` be installed in your path. It compares all genomes to the reference SARS-CoV-2 (`./interpatient/reference.fna`) and removes those that are <99% nucleotide identity.

We then align the resulting filtered sequences using MAFFT - `mafft --thread 16 mar20_filtered_seqs.fna > mar20_filtered.aln` to generate a full genome alignment of the high quality viral genomes.

We generate a phylogenetic tree of these sequences using IQTree:

`iqtree -nt 10 -s mar20_filtered.aln`
and the generated tree is available in `./interpatient/mar20_filtered.iqtree`.

**The viral genome alignment is made available in `./interpatient/mar20_filtered.aln.**

### Notebooks and analysis

We conduct a number of analyses on the alignment. In general, we want to know which positions in the genome are variable, and how variable they are. The notebook `interpatient_pi.ipynb` calculates nucleotide diversity in the alignment, with respect to the reference genome's gff annotation. It produces the following figures illustrating nucleotide diversity across the entire genome and by each gene and/or coding sequence:

Tabular data on the raw nucleotide diversity per reference position can be found in `./interpatient/nucleotide_diversity.txt`. Note that this calculation ignores any insertions or deletions from the reference and only tracks single nucleotide polymorphisms.

The notebook `./interpatient_snps.ipynb` then generated the following table of genomic substitutions across patients in `./interpatient/substitutions.txt`:

```
Position,Reference_base,Alternative_base,Ref_frequency,Alt_frequency,CodingVariant
100,G,A,0.8,0.2,S
101,G,A,0.9,0.1,S:V
```
Describing the position, nucleotides, allele frequencies, and impact on amino acid sequence (non-synonymous vs synonymous) of each substitution.

# Intrapatient variation

## Background

As viruses replicate within their hosts during infection they quickly mutate into genetically diverse populations. This is especially true for viruses with RNA genomes (as is the case with SARS-CoV-2). The variation among individuals in single host population is referred to as intrapatient variation, intraspecfic variation, or microdiversity. Analysis of this data has the following potential applications:

* Identification of the genomic loci least likely to mutate during infection. This can be useful for designing robust tests to detect the viral genome

* Comparison of viral evolution within individuals versus global evolution. This can be useful for understanding how the virus functions and can inform wet-lab experiments aimed at creating novel therapeutics

* Estimation of the number of viral particles acquired to start infection and the amount of microdiversity transferred during transmission. This is an important piece of information to understand how the virus spreads from a public health perspective.

## Methods

A typical pipeline for SARS-CoV-2 genome sequencing involves first generating a large number of DNA sequencing reads (short sequences of DNA that come from a single viral particle) and then assembling this data into a patient consensus genome (a representation of the most common viral genotype in a sample). Consensus genomes can be used for identifying outbreak clusters and global spread of the virus, but to understand the microdiversity present in a single sample, we use the raw DNA sequencing reads. Microdiversity (intrapatient variation) was profiled using the program [inStrain](https://github.com/MrOlm/instrain)

### Downloading and processing raw reads

Raw reads were identified by searching the term "SARS-CoV-2" in the NCBI SRA, and selecting the platform "Illumina". This was last performed on **3/17/2020** and led to the identification of **36** runs. This can be accessed at the following [URL](https://www.ncbi.nlm.nih.gov/sra/?term=(%22Severe+acute+respiratory+syndrome+coronavirus+2%22%5BOrganism%5D+OR+SARS-CoV-2%5BAll+Fields%5D)+AND+%22platform+illumina%22%5BProperties%5D). This information was downloaded into a .csv file by clicking "Send to:" -> Choose Destination: "File" -> Format "RunInfo" -> "Create File".

SRA files were downloaded using the URLs in the above file using the command:

```
$ cat SraRunInfo.csv | tail -n +2 | awk -F ',' '{print "wget " $10}' | bash
```

SRA files were converted into .fastq files using the command:

```
$ for s in $(ls SRR* | grep -v SRR11140750); do fastq-dump $s --split-files; done
```

For paired reads (reads where files ending in `_1.fastq` and `_2.fastq` were both created), the BBtools command `repair.sh` was used to remove un-paired reads using the command:

```
repair.sh in={fastq1} in2={fastq2} out={fastq1.repair} out2={fastq2.repair}
```

All reads were then trimmed and filtered using the BBtools command `bbduk.sh`:

```
# For paired reads:
repair.sh in={fastq1.repair} in2={fastq2.repair} out={fastq1.bbduk} out2={fastq2.bbduk}

# For unpaired reads:
bbduk.sh in={fastq1} out={fastq1.bbduk}
```

A table listing all SRA files currently analyzed in this study is available at [datatables/SRA_metadata_v1.csv](datatables/SRA_metadata_v1.csv)

### Mapping reads to a reference genome and calculating microdiversity

Microdiversity is calculated using the program [inStrain](https://github.com/MrOlm/instrain) which uses a `.bam` file as its input. A `.bam` file describes where each individual read or read-pair best maps to reference genome.

The reference genome used in this study is [NCBI Reference Sequence: NC_045512.2](https://www.ncbi.nlm.nih.gov/nuccore/NC_045512). The `.fasta` file was downloaded for mapping by clicking "Sent to:" -> "Complete Record" -> Choose Destination: "File" -> Format "FASTA" -> "Create File". A `Genbank` format version listing gene locations was also downloaded using the same process but clicking Format "GenBank (full)".

Reads were mapped to the reference genome using the program Bowtie2 with the following commands:

```
# Make an index for mapping
$ bowtie2-build NC_045512.2.fasta NC_045512.2.fasta.bt2

# Example command for paired reads
$ bowtie2 -x /home/mattolm/user_data/Covid_19/genomes/NC_045512.2.fasta.bt2 -1 /home/mattolm/user_data/Covid_19/reads/filtered/SRR11278092_bbduk.fastq --no-unal -S /home/mattolm/user_data/Covid_19/inStrain/mapping/NC_045512.2.fasta.bt2-vs-SRR11278092.sam -p 10 2> /home/mattolm/user_data/Covid_19/inStrain/mapping/NC_045512.2.fasta.bt2-vs-SRR11278092.sam.log

# Example command for unpaired reads
$ bowtie2 -x /home/mattolm/user_data/Covid_19/genomes/NC_045512.2.fasta.bt2 -U /home/mattolm/user_data/Covid_19/reads/filtered/SRR11278092_bbduk.fastq --no-unal -S /home/mattolm/user_data/Covid_19/inStrain/mapping/NC_045512.2.fasta.bt2-vs-SRR11278092.sam -p 10 2> /home/mattolm/user_data/Covid_19/inStrain/mapping/NC_045512.2.fasta.bt2-vs-SRR11278092.sam.log
```

InStrain was then run on the resulting `.sam` files using the following commands (inStrain automatically converts `.sam` for `.bam` files):

```
# Example for paired reads
inStrain profile /home/mattolm/user_data/Covid_19/inStrain/mapping/NC_045512.2.fasta.bt2-vs-SRR11241255.sam /home/mattolm/user_data/Covid_19/genomes/NC_045512.2.fasta -o /home/mattolm/user_data/Covid_19/inStrain/profiles/NC_045512.2.fasta.bt2-vs-SRR11241255.sam.IS -p 10 --pairing_filter non_discordant -g /home/mattolm/user_data/Covid_19/genomes/NC_045512.2.gb --skip_mm_profiling

# Example for unpaired reads
inStrain profile /home/mattolm/user_data/Covid_19/inStrain/mapping/NC_045512.2.fasta.bt2-vs-SRR11278167.sam /home/mattolm/user_data/Covid_19/genomes/NC_045512.2.fasta -o /home/mattolm/user_data/Covid_19/inStrain/profiles/NC_045512.2.fasta.bt2-vs-SRR11278167.sam.IS -p 10 --pairing_filter non_discordant -g /home/mattolm/user_data/Covid_19/genomes/NC_045512.2.gb --skip_mm_profiling
```

### Processing inStrain results

Here's how I did that with links to jupyter notebooks. Include all that weird genes stuff.

## Results

[Figure1]: results/QC_boxplots_v2.png
[Figure2.1]: results/CoverageDistrubution_Center_ID_v2.png
[Figure2.2]: results/MicrodiversityDistrubution_Center_ID_v2.png

### Effect of sequencing protocol on resulting reads

The reads from the SRA were prepared using a number of different RNA extraction methods, library preparation methods, and DNA sequencing machines. To identify potential biases associated with different methods, we plotted the distribution of mapping quality metrics as compared to library preparation metadata (as retrieved from the SRA).

![Figure 1][Figure1]

The figure shows that the sequencing protocol does have an impact on metrics such as nucleotide diversity and coverage standard deviation, but it is difficult to disentangle the effects of the different variables. Overall it does not look like any particular bias has a systemic bias associated with it.

We also plotted out the nucleotide [coverage distribution][Figure2.1] and [nucleotide diversity distribution][Figure2.2]. This data led us to conclude that ~1000x coverage is needed for a smooth nucleotide diversity distribution, and that ~50x coverage is needed for a smooth coverage distribution. Going forward we restricted our analysis primarily to the 24 samples that have >= 50x coverage overall.

### Microdiversity analysis

This is where I'll show the validation figure and the gene-level figure

### Comparison of intra- and inter- patient diversity

This will be a nice summary figure on that

## Conclusions

A couple of bullet points

## Data availability

### Raw data

You can acquire either sequencing reads or assembled viral genomes from:

https://www.ncbi.nlm.nih.gov/genbank/sars-cov-2-seqs/

https://gisaid.org


### Parsed data tables (`./datatables`)

* SRA_metadata_v1.csv - Metadata of the SRA samples used for intra-patient analysis

* COVID_genome_coverage_v1.csv - Genome-wide coverage, breadth, nucleotide diversity, etc. of reads mapping to a COVID-19 reference genome

* COVID_positional_coverage_v1.csv - Position by position coverage and nucleotide diversity along the genome in sequencing reads

* COVID_SNPs_v1.csv - Genomic locations of intrapatient SNPs


# Citations and acknowledgements

This work completely depends on the scientists that originally sequenced these genomes and made their data publicly available. For the intrapatient analysis, this includes:

* Technological University of Pereira

* University of Washington

* The Wuhan Institute of Virology

* Beijing Institute of Genomics

* The University of Wisconsin–Madison

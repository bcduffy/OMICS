# Advanced Omics Project ReadMe - Low-dose Tat Project
Advanced Omics Project

22Aug2023 reviewed various functions - pwd, cd, ls and options, mkdir and nano

28Aug2023: Reviewing git commit and other git functions
28Aug2023: Generated SSH keys

15Sep2023: Created project directory in 'common' directory, separate from repos.
Transferred read files from Kortagere lab server using rsync command, imported the
reference transcriptome, and created the [data read me file](http://10.11.19.48/hub/user-redirect/lab/tree/common/LoTat101PFC/data_readme.md)

## Project Summary
Project materials held in common/LoTat101PFC are for analysis of RNA-sequencing data
from rat brain tissue samples. This project is intended to evaluate differential 
expression in a model of HIV-associated neurocognitive disorder (HAND). To this end, 
stereotaxic surgeries were carried out to inject Sprague-Dawley rats (Rattus norvegicus) with recombinant HIV-1 Tat protein of the 101 amino acid isoform (Tat101, 40ng, n=8) or vehicle (Saline, 2ul, n=8) to the prefrontal cortex (PFC) of the male Sprague-Dawley rats species Rattus norvegicus, strain Sprague-Dawley/CD1). Following 21 days of recovery, rats were euthanized and brain tissue was extracted. Prefrontal cortex, inclusive of the injection site, was dissected from each tissue sample. PFC tissue was then homogenized in TRIzol and processed to isolate RNA by TRIzol-chloroform method. RNA (2 micrograms) was provided to GeneWiz (Azenta Life Sciences, South Plainfield, NJ) for standard paired end RNA sequencing (un-stranded, polyA selection). Azenta provided read files as gzipped fastq files (.fq.gz) via SFTP connection to their server; the read files, held in the subdirectory common/LoTat101PFC/reads/, were then copied from the Kortagere lab server by rsync command. 

## Directory Organization 
The project directory (common/LoTat101PFC/) contains individual sub-directories to house distinct
project components. 

The 'reference' subdirectory contains the reference transcriptome (Rattus_norvegicus.mRatBN7.2.cdna.all.fa.gz), which was imported 
from the most recent Ensembl release for the mRatBN7.2 cDNA sequence (FASTA format), 
using the wget command. It also contains an index for the rat transcriptome as generated from Kortagere lab scripts which appended the genome to the transcriptome and utilized salmon for indexing.

The quants subdirectory contains sample folders labeled with the sample IDs as generated by the makefile script, containing the quant.sf file generated for each sample. 
The merged subdirectory contains the merged.sf file, which is the final output of the makefile, merging data from all sample quant.sf files.

The scripts for the project are housed in a separate directory at the same level as the project directory (common/RNAseq-scripts).
These scripts are salmon_script.sh which was used in early development to develop stepwise understanding of the salmon quant command;
and makefile - which was ultimately used to generate the quants and merged folders, each sample's quants.sf file and the merged quants file (merged.sf).

The reads subdirectory (/common/LoTat101PFC/reads/) contains FASTQ format read files, 
for RNA samples from animal IDs BCD1-16. The read files were pulled from the Kortagere Lab
server using the rsync command, indicating the lab server as the source and pulling to 
the reads subdirectory (after first navigating into the reads subdirectory).

	`rsync --progress brenna@10.18.160.45:home/brenna/data/rnaseq/Rat_LoTatPFC_30-873800053/00_fastq/*.gz .`

## Using Salmon and scripting to analyze RNA-seq read files (FASTQ for BCD1-16)
In order to use salmon an index was required to count the reads against an indexed transcriptome. It was recommended to use a previously created index, which was available from prior indexing of the mRatBN7.2 transcriptome. This index was copied in from Kortagere lab server via rsync command, stored in 'reference' subdirectory as 'Prebuilt_Index-mRatBN7.'

To start counting reads against the index, I started by writing 'salmon_script.sh' as a basis for analyzing a single file. In this file, the 'read_files' variable and 'salmon_index' variable were defined to point to the reads directory and the prebuilt index of the rat transcriptome (relative filepaths). Salmon quant was then applied specifying the first sample BCD1 (control group). Running this salmon script (bash salmon_script.sh) generated the quants sub-directory and the contained BCD1 folder, which contains the quant.sf file for this sample.
This salmon quant command utilizes auto detection of the library type, specifies the locations of the read 1 and read 2 files for the sample BCD1, and uses options for mapping validation and designating the output directory.
~~~
!/bin/bash
read_files="./reads/"
salmon_index="./reference/Prebuilt_Index-mRatBN7/"

salmon quant --index $salmon_index -l A  -1 $read_files/BCD1_R1_001.fastq.gz -2 $read_files/BCD1_R2_001.fastq.gz --validateMappings -o quants/BCD1
~~~

## Creating a make file to generate quant.sf files for all reads
To develop a make file that iterates the salmon quantification through all sample read files, initially tried the following script:
~~~
reads="/common/LoTat101PFC/reads"
ref="/common/LoTat101PFC/reference/Prebuilt_Index-mRatBN7.2"
	quants/%: $(reads)/%_R1_001.fastq.gz $(reads)/%_R2_001.fastq.gz
	salmon quant -i $(ref) -l A -1 $(reads)/%_R1_001.fastq.gz -2 $(reads)/%_R2_001.fastq.gz --validateMappings --GCbias -o $@
~~~

I then ran 'make quants/BCD2' to test the script, but received error 'make: *** No rule to make target 'quants/BCD2'.'

After revising the script to replace the capture symbol % in the read file path with $* to indicate that the already captured sample ID should be used to select the corresponding read files for quantification, checking the spelling and case of the --gcbias flag, I ran the script again.
The next error indicated that the index version file /common/LoTat101PFC/reference/Prebuilt_Index-mRatBN7.2/versionInfo.json doesn't seem to exist (could not be located).  This suggests that the wrong path was used to indicate the Ref path to the index (mRatBN7.2). 
Revising to the correct path 'common/LoTat101PFC/reference/Prebuilt_Index-mRatBN7' did not fix this error. After correcting the filepath ("mRatBN7" instead of "7.2") and changing from the absolute path (/common/LoTat101PFC/reads) to the relative path (reads="./reads", and keeping ref="./reference/Prebuilt_Index-mRatBN7"), the program seems to now locate all files.
After a few minutes it finally generated the quant file for BCD2, indicating that the makefile using the sample rule and now using relative file paths should work. 

~~~
reads="./reads"
ref="./reference/Prebuilt_Index-mRatBN7"
	quants/%: $(reads)/%_R1_001.fastq.gz $(reads)/%_R2_001.fastq.gz
	salmon quant -i $(ref) -l A -1 $(reads)/$*_R1_001.fastq.gz -2 $(reads)/$*_R2_001.fastq.gz --validateMappings --gcBias -o $@
~~~

Then I ran 'make quants/BCD3' to ensure that the script generates salmon quant output for a new sample that has not been quantified yet.
I now receive the same error as earlier - "No rule to make target 'quants/BCD3'."
Consulted DVK for help. DVK helped identify that the READS and REF variable paths were not indicated correctly, so I removed the quotation marks and kept the relative path rather than the absolute path. By modifying the reads and ref paths, and implementing code from Lecture 9 modified for my defined variables, I end up with:

The final script allows iteration through all read files in the project directory, due to the following features:
* FASTQ_FILES rule defines FASTQ as a list of prefixes collected from the read files, i.e. sample IDs
* QUANTDIR rule defines QUANTDIR as a list of directories to be named with the pattern 'quants/%' where % is replaced with the sample IDs captured in the FASTQ listing
* 'all' rule indicates the merged quants table as a dependency, and therefore the script will only report "complete" once the merged quants table is created
	* merged/merged.sf is indicated as a dependency, therefore the merged/merged.sf rule must be satisfied before the script is complete;
*  merged/merged.sf rule ensures that the script prints to the screen a message indicating which sample's results are being merged into the merged quants table, and that the QUANT_DIRS are all defined and directories created before script completion.
* quants/% rule is dependent on the read 1 and read 2 files for each sample, with the salmon quant command embedded in this rule ensuring that each sample is quantified as a dependency of quants/%
	* within this rule, the salmon quant command is used as previously in salmon_script.sh, to generate the count of mapped reads per gene in each sample based on the presence of both read files.  
* the 'clean' rule indicates to remove the quants and merged directories recursively

Thus this final script enables the generation of quant.sf files housed within a folder named for the sample ID within the 'quants' directory'; and these are merged together by salmon quantmerge command only once all individual quant tables are generated.

~~~
READS = ./reads
REF = ./reference/Prebuilt_Index-mRatBN7

FASTQ_FILES := $(wildcard $(READS)/*_R1_001.fastq.gz)
QUANT_DIRS := $(patsubst $(READS)/%_R1_001.fastq.gz,quants/%,$(FASTQ_FILES))

all: merged/merged.sf
        echo "complete"

clean:
        rm -r quants
        rm -r merged 

merged/merged.sf: $(QUANT_DIRS)
        echo "Merging: $(QUANT_DIRS)"
        mkdir -p merged
        salmon quantmerge --quants $^ -o merged/merged.sf

quants/%: $(READS)/%_R1_001.fastq.gz $(READS)/%_R2_001.fastq.gz
        mkdir -p quants
        echo "Salmon on $*"
        salmon quant -i $(REF) -l A -1 $(READS)/$*_R1_001.fastq.gz -2 $(READS)/$*_R2_001.fastq.gz --validateMappings --gcBias -o $@
~~~

A dry run using make all -n gives no flags and reports the appropriate output from echo commands. 
Running 'make all' succeeded at generating the remaining quant.sf files for BCD3-16, and merging all existing quant.sf files into merged.sf, accounting for all 16 samples.

## Future Directions
Having generated the merged counts table, containing the counts for reads mapped per transcript in the transcriptome across all samples, downstream analyses can be carried out. I would likely need to use Python or Rstudio to perform differential expression analysis on the counts table. DESeq2 is standardly used to analyze Salmon quant outputs and correct for multiple testing.
Other potential analyses would include enrichment analysis; I would likely need to revisit my quantification or modify the merged table to include the annotations for matching to GO terms. Enrichment analysis in particular would be a strong analysis to show overall transcriptional changes in pathways to indicate changes induced in Tat exposed tissue.
More short-term needs are to finalize the creation of Git repos for the project and merged quant table as project results, for accessibility particularly to the Kortagere lab for future analyses and other projects for which the scripts will be useful.

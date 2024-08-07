# tr-errors-pipeline-v1
Pipeline for calling transcription errors from a rolling-circle (circ-seq) library

# System requirements

## Hardware requirements

This pipeline  requires only a standard computer with enough RAM to support the in-memory operations.

## OS requirements

This package is supported for Linux operating systems only. It may be possible to compile on Mac OS systems, although this has not been tested.

# Dependencies

In addition to a C++ compiler (gcc), this pipeline requires the following software:
* topHat (version 2.1.1 or more recent) : https://ccb.jhu.edu/software/tophat/index.shtml
* blat (https://kentinformatics.com/#BLAT)
* 

# INSTALLATION:

Installation typically takes only a few minutes.

After cloning the repository from github, simply type:
```make all```
to build all the executables necessary for the pipeline.

Alternatively, it is possible to build the libraries first (make libs) and then the executables (make pipeline and make extras).
This should create two directories: libs and bin.
Here is the list of executables that are part of the pipeline:

* findConsensus: Finds the repeats in reads from fastq files (single or paired-end)

* findBreakPointFromBLAT: Identifies the position of breakpoint (=the ends of the initial RNA fragment)

* refineBreakpoint-withIndels: Tries to refine the position of the breakpoint in consensus sequences that do not have a perfect match on the genome.

* generateMaps: prepares the maps used for re-genotyping of the samples

* callTrErrorsWithIndelsFromCircSeq: The 'final' step in the pipeline (=the program that actually calls the transcription errors)

* demulti: generates all possible variants of the consensus from moving the breakpoitn from one point to the other (used only on the reads that have a transcription error called in the previous step)

Extra programs:

* addBaseToObservations: adds the nucleotide information (relative to the coding strand) to an observation map file.

* addBaseAndStrandToObservations: adds the nucleotide (relative to the coding strand) and strand information to a file of observations.

* mergeObservationMaps: merges multiple observation map files.



# RUNNING THE PIPELINE:
-----------------------

Starting material:

- A fastq file containing the read sequences from a circle-seq (rolling circle) experiment.
- The genome assembly and annotation from the organism studied.
  - A fasta file containing all the spliced transcript sequences (+ some flanking sequence ideally, but this is not mandatory)
	
 #Additional software requiered:
	- tophat2 (https://ccb.jhu.edu/software/tophat/index.shtml)
	- blat (http://www.kentinformatics.com/) [Note that it should work also with blast, as long as blast8 output format is used]
	

# Step 0. Trimming the reads (this step is optional, but recommanded):
----------------------------------------------------------------------

Use your favorite program to trim reads, removing low quality base calls at the end of the sequence, and removing primer sequences.
One option is to use cutadapt (http://cutadapt.readthedocs.io/en/stable/guide.html) but any trimming program should work.

# Step 1. Finding the repeats in the reads:
-----------------------------------------
 
The first step consists in finding long repeats (= the repeats of the RNA circle) in the reads from the fastq file(s). This is done with the program 'findConsensus'
 

```findConsensus BASE_NAME reads.fastq [mates.fastq]```

BASE_NAME will be used to name a number of files produced by the pipeline.
This program will generate the following files:
* BASE_NAME-CS.pos: reports the size of the repeats and the starting position of the first full-length repeat for each read from reads.fastq
* BASE_NAME-DCS.fastq: contains a tandem concatenation of each repeat

The program also outputs one the stdout the total number of reads for which repeats have been identified.

# Step 2. Map the tandem repeat to the genome:
--------------------------------------------

Before mapping to the genome with BLAT, it is necessary to convert the BASE_NAME-DCS.fastq into a fasta file.
There are many options to do so. One fast and easy solution is to use awk:

```awk 'BEGIN{P=1}{if(P==1||P==2){gsub(/^[@]/,\">\");print}; if(P==4)P=0; P++}' FastqFile > FastaFile```

Then, run the blat command:

```blat blatDB.fa BASE_NAME-DCS.fastq blatResult-=dna -q=dna -out=blast8```

blatDB.fa should contain every spliced transcript + some extra flanking sequence (typically 200nt)


# Step 3. Infer the position the breakpoint (= the end of the initial RNA framgnets) from the blat mapping:
---------------------------------------------------------------------------------------------------------

```findBreakPointFromBlat blatResult BASE_NAME-DCS.fastq > BASE_NAME-DEBREAKED.fastq 2> blat.map```

# Step 4. Re-map the de-breaked reads with tophat:
 ------------------------------------------------

tophat2 --no-sort-bam --no-convert-bam -o tophat -G gffFile tophatDB BASE_NAME-DEBREAKED.fastq > tophat.log 2> tophat.err

 Where gffFile is the GFF/GTF annotation and tophatDB is the bwa-indexed reference genome

 # Step 5. Refine the breakpoints:
 -------------------------------

 For each reads that does not have a perfect match in the genome, the program tries to move around the breakpoint position and find a perfect match with the updated breakpoint position.

refineBreakpoint-withIndels BASE_NAME blat.map BASE_NAME-CS.pos tophat/accepted_hits.sam T/F referenceGenome.fa exonsWithFake.gff GFF/GTF linkID reads.fastq [mates.fastq]

 BASE_NAME / blat.map / BASE_NAME-CS.pos : same as previously defined
 tophat/accepted_hits.sam : location of the accepted hits from tophat from step 4
 T/F : Should only uniquely mapped reads be kept (True or False) - recommanded: F
 referenceGenome.fa : self-explanatory
 exonsWithFake.gff : a GFF file containing the location of all annotated exons + one 'fake exon' per scaffold simulating an exon covering the entire scaffold.
 GFF/GTF : the format for file exonsWithFake.gff
 linkID : The keyword in the GFF file linking together exons belonging to the same transcript (typically: Parent)


# Step 6. Re-map the refined reads:
 ---------------------------------


```tophat2 --no-sort-bam --no-convert-bam -o tophat-refined -G gffFile tophatDB BASE_NAME-DEBREAKED.refined.fastq ```

# Step 7. Use the mapped reads to 're-genotype' the sample:
 ---------------------------------------------------------

 First, we need to prepare a list of files that go into this step:
```
ls BASE_NAME-NMzero.sam > listFile
ls tophat-refined/accepted_hits.sam >> listFile
```

 Note that if the sequencing run gave multiple files, you will need to include all the files (sharing the same genotype) from the run.

generateMaps listFile 20 referenceGenome.fa mapFile mapIndelsFile

 20 is the cutoff for base quality
 mapFile and mapIndelsFile are the two output files created by the program.


# Step 8. Transcription error calling.
 ------------------------------------

 First, I run the program on the perfectly mapped reads. Obviously, no transcription error should be called here, but this is important to know how many observations were done.


```callTrErrorsWithIndelsFromCircSeq blat.map BASE_NAME-CS.pos BASE_NAME-NMzero.sam F referenceGenome.fa exons.gff GFF/GTF linkID mapFile mapIndelsFile 20 0.05 observations.tab 3 8 2 2 100 T reads.fastq [mates.fastq] > candidates 2> log```

 Then, on the reads that could contain transcription errors:

```callTrErrorsWithIndelsFromCircSeq blat.map.refined BASE_NAME-CS.pos tophat-refined/accepted_hits.sam F referenceGenome.fa exons.gff GFF/GTF linkID mapFile mapIndelsFile 20 0.05 observations-refined.tab 3 8 2 2 100 T reads.fastq [mates.fastq] > candidates-refined 2> log```


# Step 9. Testing all possible breakpoint positions for the reads that had a transcription error called:
 ------------------------------------------------------------------------------------------------------
```
grep "" candidates-refined | sed 's/^//' > candidates-refined.clean
cut -f 2 candidates-refined.clean | sort | uniq > readIDs-tdm
demulti BASE_NAME-DEBREAKED.refined.fastq readIDs-tdm > demult.fastq
tophat2 --no-sort-bam --no-convert-bam -o tophat-demult -G exonsFile tophatDB demult.fastq
grep "NM:i:0" tophat-demult/accepted_hits.sam | cut -f 1 | sed 's/^@//' | sed 's/_/ /' | cut -d ' ' -f 1 | sort | uniq > demult-zero.IDs
grep -v -f demult-zero.IDs candidates-refined.clean > candidates-refined.clean-demult
```
 All the final transcription errors are now in candidates-refined.clean-demult
 

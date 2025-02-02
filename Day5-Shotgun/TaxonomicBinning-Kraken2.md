# Taxonomic binning with Kraken2
- [Taxonomic binning with Kraken2](#taxonomic-binning-with-kraken2)
  - [Setup and installation](#setup-and-installation)
    - [Folder structure](#folder-structure)
    - [Installing Kraken2](#installing-kraken2)
  - [Running Kraken2](#running-kraken2)
  - [Analyzing the output](#analyzing-the-output)
    - [Kraken2 on contigs](#kraken2-on-contigs)

Kraken is a very widely utilized metagenomics tool (dating back to 2014) and Kraken2 is its most recent iteration that boasts higher accuracy and decreased system resource utilization over Kraken1. 

In contrast to mOTUs2 and MetaPhlAn3 which use clade specific marker _genes_, Kraken2 utilizes clade specific marker _kmers_. This is summarized in the first figure of the [Kraken1 manuscript](https://genomebiology.biomedcentral.com/articles/10.1186/gb-2014-15-3-r46):

![Kraken](https://user-images.githubusercontent.com/6362936/128582419-7da7594e-9483-4138-859e-d0f0e332bad3.PNG)

In using k-mers, Kraken2 has the ability to be blazingly fast, but longer query sequences are required to account for the inherent noise when utilizing k-mers in the size ranges that Kraken utilizes. Probably due to it's relatively early implementation and how fast the tool is, Kraken has been used in quite a few situation it should not have. For example, remember when we covered [taxonomic profiling](TaxonomicProfiling.md)? Well, taxonomic profiling doesn't seem _too_ different from taxonomic binning, right? A sequence is a sequence, right? Unfortunately, Kraken is quite sensitive to sequence length (as well as sequencing error). So many people were (incorrectly) using Kraken for taxonomic profiling that the authors of Kraken were motivated to publish [Bracken](https://peerj.com/articles/cs-104/) which is an extension of Kraken designed for taxonomic profiling. In [that manuscript](https://peerj.com/articles/cs-104/) the authors specifically state:
>Therefore, any assumption that Kraken’s raw read assignments can be directly translated into species- or strain-level abundance estimates (e.g.,  Schaeffer et al., 2015) is flawed, as ignoring reads at higher levels of the taxonomy will grossly underestimate some species, and creates the erroneous impression that Kraken’s assignments themselves were incorrect.

In the following, we will contrast the performance of Kraken when applied to raw reads and when applied to assembled contigs. But first, we need to set everything up!

## Setup and installation

### Folder structure
We are going to use out familiar folder structure and copy over both the raw reads as well as the assembled data corresponding to one of our samples:
```bash
mkdir Kraken2_analysis
cd Kraken2_analysis
mkdir data/k2train8gb scripts output output/on_MEGAHIT output/on_GATB output/on_raw_reads
cp ../mOTUs_analysis/data/SRS014464-Anterior_nares.fastq data/SRS014464-Anterior_nares.fastq
cp ../MEGAHIT_analysis/output/default/final.contigs.fa data/MEGAHIT_default_contigs.fasta
cp ../GATB_analysis/output/default.fasta data/GATB_default_contigs.fasta
```

### Installing Kraken2
Kraken2 is available on conda and is quite straightforward to install:
```
conda deactivate
conda create -y -n kraken2 kraken2
conda activate kraken2
```

After this, you need to choose what sort of database to use with Kraken. A mechanism is built in that downloads the default database (which is ~100GB in size), but alternate databases are available at: https://github.com/BenLangmead/aws-indexes/blob/master/docs/k2.md

We will be using the smallest database for the sake of time (it includes archaea, bacteria, viruses, plasmids, human sequences, and UniVec_Core (vector contamination). It is capped to only 8GB in size though, so shouldn't be used for serious research (I'd suggest using the default or the "PlusPF" database). We can obtain the training data via:
```bash
wget https://genome-idx.s3.amazonaws.com/kraken/k2_standard_8gb_20210517.tar.gz -P data/k2train8gb
tar -xzvf data/k2_standard_8gb_20210517.tar.gz -C data/k2train8gb
```
This will take some time to download.

## Running Kraken2

Running Kraken is quite straightforward (i.e. a single line of code), so we will jump directly to creating a script that will do all our analysis for us. First though, let's check out the Kraken parameters:
```
Usage: kraken2 [options] <filename(s)>

Options:
  --db NAME               Name for Kraken 2 DB
                          (default: none)
  --threads NUM           Number of threads (default: 1)
  --quick                 Quick operation (use first hit or hits)
  --unclassified-out FILENAME
                          Print unclassified sequences to filename
  --classified-out FILENAME
                          Print classified sequences to filename
  --output FILENAME       Print output to filename (default: stdout); "-" will
                          suppress normal output
  --confidence FLOAT      Confidence score threshold (default: 0.0); must be
                          in [0, 1].
  --minimum-base-quality NUM
                          Minimum base quality used in classification (def: 0,
                          only effective with FASTQ input).
  --report FILENAME       Print a report with aggregrate counts/clade to file
  --use-mpa-style         With --report, format report output like Kraken 1's
                          kraken-mpa-report
  --report-zero-counts    With --report, report counts for ALL taxa, even if
                          counts are zero
  --report-minimizer-data With --report, report minimizer and distinct minimizer
                          count information in addition to normal Kraken report
  --memory-mapping        Avoids loading database into RAM
  --paired                The filenames provided have paired-end reads
  --use-names             Print scientific names instead of just taxids
  --gzip-compressed       Input files are compressed with gzip
  --bzip2-compressed      Input files are compressed with bzip2
  --minimum-hit-groups NUM
                          Minimum number of hit groups (overlapping k-mers
                          sharing the same minimizer) needed to make a call
                          (default: 2)
  --help                  Print this message
  --version               Print version information
```
We will want to specify the database we downloaded with the `--db` flag, and save the following output files: `--classified-out` (the sequences that were classified along with their classification), `--output` (the normal output of Kraken2), and `--report` the summary of which clades were found in the input according to Kraken.

Let's go ahead and run Kraken on both assemblies, as well as the raw reads. We will put all of this in a script to execute everything in one go:
```
set -e  # exit if there is an error
set -u  # exit if a variable is undefined

baseDir=/home/dmk333/KickStartWorkshop2021/Kraken2_analysis #<<-- replace with your full path
dataDir=${baseDir}/data
trainingDir=${dataDir}/k2train8gb
outputDir=${baseDir}/output
for inOut in "MEGAHIT_default_contigs.fasta on_MEGAHIT" "GATB_default_contigs.fasta on_GATB" "SRS014464-Anterior_nares.fastq on_raw_reads";
do
  args=( $inOut );
  input=${dataDir}/${args[0]}
  output=${outputDir}/${args[1]}
  kraken2 kraken2 --db ${trainingDir} --threads 10 --output ${output}/kraken_default_output.txt --classified-out ${output}/kraken_classified_sequences.fq --use-names --report ${output}/kraken_report.txt ${input}
done
```
Let's put this in a file named `run_kraken2.sh`, make it executable (`chmod +x run_kraken2.sh`) and then let it rip!

## Analyzing the output

The `kraken_classified_sequences.fq` files contain fastq formatted files of the reads/contigs with an NCBI taxID in the header of each sequence. This can be helpful for determining the correspondence between contigs and what organisms they originated from.

The `kraken_default_output.txt` contains a compressed-ish version of the above, but with additional information about what led to Kraken's inference. More details can be found [here](https://github.com/DerrickWood/kraken2/blob/master/docs/MANUAL.markdown#standard-kraken-output-format).

Lastly, the `kraken_report.txt` is what led to some interpreting Kraken2 as usable as a taxonomic profiler (it sure looks like a profile!). The first number is the percent of sequences that covered this taxon, followed by the number assigned to that clade, then to that taxon, a rank code, and then an NCBI taxID. More info can be found [here](https://github.com/DerrickWood/kraken2/blob/master/docs/MANUAL.markdown#sample-report-output-format) about the format.
Let's dig in a bit more:

### Kraken2 on contigs
Take a look at the `kraken_report.txt` in the `output/on_GATB` folder. See how this makes sense considering the BLAST results we saw earlier? In general, _this_ is what you want to be using instead of BLAST when you want to identify the taxa of each of your contigs. The same story emerges when we look at the report in the `output/on_MEGAHIT` directory.

Now, let's contrast this to what happens when we look at the report in the raw reads directory: `output/on_raw_reads`. We get the following species:
```
Moraxella nonliquefaciens
Moraxella catarrhalis
Moraxella bovis
Moraxella bovoculi
Moraxella ovis
Pseudomonas tolaasii
Escherichia coli
Klebsiella aerogenes
Xanthomonas euvesicatoria
Haemophilus parainfluenzae
Methylophilus medardicus
Dolosigranulum pigrum
Streptococcus oralis
Staphylococcus epidermidis
Staphylococcus sp. SB1-57
Staphylococcus warneri
Paenibacillus kribbensis
Finegoldia magna
Fastidiosipila sanguinis
Cutibacterium acnes
Cutibacterium avidum
Corynebacterium propinquum
Corynebacterium segmentosum
Corynebacterium macginleyi
Corynebacterium sp. FDAARGOS 1242
Corynebacterium sp. LMM-1652
Corynebacterium kefirresidentii
Corynebacterium stationis
Corynebacterium glucuronolyticum
Corynebacterium epidermidicanis
Corynebacterium kutscheri
Corynebacterium diphtheriae
Corynebacterium xerosis
Lawsonella clevelandensis
Streptomyces tsukubensis
Prevotella denticola
Homo sapiens
Propionibacterium virus Ouroboros
Propionibacterium virus PHL041M10
```
This is what the Braken paper was refering to when it said not to trust the species classifications. Of course one could take a look higher ranks, like the Genus level:
```
Moraxella
Pseudomonas
Escherichia
Klebsiella
Xanthomonas
Haemophilus
Pseudoalteromonas
Neisseria
Methylophilus
Sphingobium
Dolosigranulum
Streptococcus
Staphylococcus
Paenibacillus
Finegoldia
Fastidiosipila
Cutibacterium
Corynebacterium
Lawsonella
Streptomyces
Bacteroides
Prevotella
Homo
Pahexavirus
```
but here again we see lots of false positives. One is left to determine on their own what is an acceptable threshold of "% reads classified." Moral of the story? Use Kraken for taxonomic binning of contigs, use Braken (or MetaPhlAn3, or mOTUs2) for taxonomic profiling.

# Congrats on finishing the "intro to WGS metagenomics" tutorial!

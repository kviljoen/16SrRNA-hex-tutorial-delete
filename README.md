# 16S analysis tutorial - part of Analysing the Microbiome Workshop 2017

## The dataset

The dataset we will be working are the practice dataset from the [H3ABioNet 16S rDNA diversity analysis SOP](http://www.h3abionet.org/tools-and-resources/sops/16s-rrna-diversity-analysis). The source data can be accessed [here](http://h3data.cbio.uct.ac.za/assessments/16SrRNADiversityAnalysis/practice) but for our purposes it is already on the cluster and stored here:`/global/mb/amw/dog_stool_samples`

The table below contains the metadata associated with the dog stool samples. There are three dogs which are treated with increased percentage of a compound in their diet: 5 different treatments (0-4, representing an increased percentage of a compound in their diet).

Sample | Dog | Treatment
------ | --- | ---------
Dog1 | B | 2
Dog2 | G | 3
Dog3 | K | 3
Dog8 | B | 4
Dog9 | G | 0
Dog10 | K | 4
Dog15 | B | 1
Dog16 | G | 4
Dog17 | K | 0
Dog22 | B | 3
Dog23 | G | 1
Dog24 | K | 2
Dog29 | B | 0
Dog30 | G | 2
Dog31 | K | 1

## Outcomes
* Clone a Git repos and use the code
* Edit files and command line options, work with bash variables, work with bash pipes, use awk, use bash loops, use bash snippets
* Run a 16S analysis pipeline from raw reads up to OTU classification and alignment

## Do some local setup
### Maybe pull this code here

### Setup some PATHS
```bash
export PERL5LIB=$PERL5LIB:/global/mb/amw/soft/File-Util-3.27/blib/lib
export PERL5LIB=$PERL5LIB:/global/mb/amw/soft/Class-OOorNO-0.011/blib/lib
export PATH=$PATH:/global/mb/amw/soft/uparse_helpers
export PATH=$PATH:/global/mb/amw/soft/fasta-splitter-0.2.4
```

### Setup some directories
```bash
uparse_dir=/global/mb/amw/run/process/uparse
```
### Add pipeline here
Add image here  

### Tutorial directory structure
Put image here

### When you get lost or something is unclear
1. All the outputs have been generated here `....`
1. Please find someone next to you that looks like they know what they are doing. 
1. Let me know.
## 1. Lets do some QC on the raw data

To be able to run all tools in the tutorial be sure the bionf module is loaded
```bash
module load bioinf
```
### 1.1 Run FastQC
```bash
fastq_dir=/global/mb/amw/dog_stool_samples
fastqc_dir=/global/mb/amw/run/process/fastqc

mkdir $fastqc_dir
fastqc --extract -f fastq -o $fastqc_dir -t 12 fastq_dir/*
```
### 1.2 Combine FastQC reports
```bash
fastqc_dir=/global/mb/amw/run/process/fastqc
/global/mb/amw/run/code/fastqc_combine/fastqc_combine.pl -v --out $fastqc_dir --skip --files "$fastqc_dir/*_fastqc"
```
Now lets view the reports.

## 2. Run the UPARSE pipeline
### 2.1 First rename the read headers so that they are compatible with the UPARSE pipline
```bash
renamed_dir="/global/mb/amw/run/process/usearch/renamed"
mkdir renamed
while read sid_fastq_pair; do sid=`echo $sid_fastq_pair | awk -F ' ' '{print $1}'`; fastq_r1=`echo $sid_fastq_pair | awk -F ' ' '{print $2}'`; fastq_r2=`echo $sid_fastq_pair | awk -F ' ' '{print $3}'`; fastq_r1_renamed=$renamed_dir"/"$(basename $fastq_r1); fastq_r2_renamed=$renamed_dir"/"$(basename $fastq_r2); /global/mb/amw/run/code/rename_fastq_headers.sh $sid $fastq_r1 $fastq_r2 $fastq_r1_renamed $fastq_r2_renamed;done < /global/mb/amw/run/code/sid.fastq_pair.list
```
This will take about 10 minutes to run. Lets have a look at the headers once done.

### 2.2 Merge the paired reads
```bash
fastq_maxdiffs=3
merged_dir="/global/mb/amw/run/process/uparse/merged"
mkdir /global/mb/amw/run/process/uparse/merged

while read sid_fastq_pair; do sid=`echo $sid_fastq_pair | awk -F ' ' '{print $1}'`; fastq_r1=`echo $sid_fastq_pair | awk -F ' ' '{print $2}'`; fastq_r2=`echo $sid_fastq_pair | awk -F ' ' '{print $3}'`; fastq_r1_renamed=$renamed_dir"/"$(basename $fastq_r1); fastq_r2_renamed=$renamed_dir"/"$(basename $fastq_r2); usearch -fastq_mergepairs $fastq_r1_renamed -reverse $fastq_r2_renamed -fastq_maxdiffs $fastq_maxdiffs -fastqout $merged_dir"/"$sid".merged.fastq";done < /global/mb/amw/run/code/sid.fastq_pair.list
```
This will take about 1 minute to run. Lets have a look at the fastq files of the merge reads.

### 2.3 Filter
```bash
fastq_maxee=0.1
filtered_dir="/global/mb/amw/run/process/uparse/filtered"
mkdir $filtered_dir

while read sid_fastq_pair; do sid=`echo $sid_fastq_pair | awk -F ' ' '{print $1}'`;  usearch -fastq_filter $merged_dir"/"$sid".merged.fastq" -fastq_maxee $fastq_maxee -fastqout $filtered_dir"/"$sid".merged.filtered.fastq"  ;done < /global/mb/amw/run/code/sid.fastq_pair.list
```
This will take about 1 minute to run. Lets do a read count on the filtered fastqs.

### 2.4 Run FastQC on the filtered reads
```bash
filtered_fastqc_dir=/global/mb/amw/run/process/filtered.fastqc
mkdir $filtered_fastqc_dir
fastqc --extract -f fastq -o /global/mb/amw/run/process/usearch/filtered.fastqc -t 12 /global/mb/amw/run/process/usearch/filtered/*.fastq
```
This will take about 2 minutes to run.

### 2.5 Combine FastQC reports
```bash

/global/mb/amw/run/code/fastqc_combine/fastqc_combine.pl -v --out $filtered_fastqc_dir --skip --files "$filtered_fastqc_dir/*_fastqc"
```
Lets have a look at the FastQC summaries and see if we notice any changes from the FastQC reports on the raw reads.

### 2.6 Convert Fastq to Fasta
```bash
filtered_fasta_dir=/global/mb/amw/run/process/usearch/filtered.fasta
mkdir $filtered_fasta_dir
for i in `ls -1 /global/mb/amw/run/process/usearch/filtered/*.fastq`; do filename=$(basename "$i"); base="${filename%.*}"; seqtk seq -A $i > $filtered_fasta_dir/$base.fa; done
```
Lets have a look at the fasta format.

### 2.7 Do dereplication

```bash
cat $filtered_fasta_dir/*.fa > $uparse_dir/filtered_all.fa
```
```bash
cat $uparse_dir/filtered_all.fa | grep -v "^>" | grep -v [^ACGTacgt] | sort -d | uniq -c | while read abundance sequence ; do hash=$(printf "${sequence}" | sha1sum); hash=${hash:0:40};printf ">%s;size=%d;\n%s\n" "${hash}" "${abundance}" "${sequence}"; done > $uparse_dir/filtered_all.uniques.fa 2> $uparse_dir/filtered_all.uniques.fa.e
```
This will take about 15 minutes. Lets have a look at the headers.

### 2.8 OTU picking
Sort by size
```bash
minsize=2
usearch -sortbysize $uparse_dir/filtered_all.uniques.fa -fastaout $uparse_dir/filtered_all.uniques.sorted.fa -minsize $min_size
```
Do OTU picking
```bash
otu_radius_pct=3
usearch -cluster_otus $uparse_dir/filtered_all.uniques.sorted.fa -otu_radius_pct $otu_radius_pct -otus $uparse_dir/otus_raw.fa
```
This will take about 30 seconds. Once done lets count how many OTUs were generated.

### 2.9 Chimera removal
```bash
usearch -uchime2_ref $uparse_dir/otus_raw.fa -db /global/mb/amw/dbs/gold.fa -mode high_confidence -strand plus -notmatched $uparse_dir/otus_chimOUT.fa
```
This will take about 10 seconds. Once done lets check how many OTUs were detected as being chimeric.

### 2.10 De-dereplication

Rename OTU headers
```bash
fasta_number.py $uparse_dir/otus_chimOUT.fa OTU_ > $uparse_dir/otus_repsetOUT.fa
```
Split fasta files to reduce memory on `usearch_global` run
```bash
mkdir $uparse_dir/split_files
cd $uparse_dir/split_files

fasta-splitter.pl --n-parts 100 --out-dir $uparse_dir/split_files/ $uparse_dir/filtered_all.fa
```
Do de-dereplication
```bash
for i in $(ls $uparse_dir/split_files/*.fa); do usearch -usearch_global $i -db $uparse_dir/otus_repsetOUT.fa -id 0.97 -strand plus -uc $i.map.uc; done
```
Combine mappings
```bash
cat $uparse_dir/split_files/*.map.uc > $uparse_dir/otus_mappedOUT.uc
```
This will take about 1 minute to complete. Lets have a look at the final mapping file.

### 2.11 Create QIIME compatible OTU table
```bash
uc2otutab.py $uparse_dir/otus_mappedOUT.uc > $uparse_dir/otus_table.tab.txt
```
THis will take about 20 seconds to complete. Have a look that the OTU table generated.










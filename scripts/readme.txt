# Microbiota Rhinosinusitis
Analysis of nanopore reads derived from patients with chronic rhinosinusitis that underwent surgery. Samples were taken before surgery (T0), and after one (T1), three (T3), six (T6) and twelve (T12) months after it. 

## R advice
Use a specific conda environment for all analysis. Open rstudio on terminal, after loading that conda env. After this, download all packages and libraries required in R using this env. 
When having trouble downloading R packages:
1. download them straight from R (opened in the terminal) - select any USA mirror
2. download them through conda repositories

# Make metadata_nreads file for raw fastq
```
echo -e "ID_Sample\tFilename\tnreads\tPatient\tTimepoint" > metadata_nreads.tsv # make the header

for file in *.fastq; do 
    nreads=$(($(wc -l < "$file") / 4))  # Count reads efficiently
    ID_Sample="${file%.fastq}"  # Remove .fastq extension
    Patient=$(echo "$ID_Sample" | sed -E 's/_T[0-9]+$//')  # Extract Patient ID
    Timepoint=$(echo "$ID_Sample" | grep -oE 'T[0-9]+$')  # Extract Timepoint
    echo -e "$ID_Sample\t$file\t$nreads\t$Patient\t$Timepoint" >> metadata_nreads.tsv # append it at the bottom of the header
done
```

# Preprocessing files and metadata 
Check scripts: 
01.Rename_files.ipynb
02.Concat_repeat_files.ipynb *
03.Process_metadata.ipynb

* PS.: I've previously checked the quality of those samples that were sequenced more than once. Since the quality overall wasn't improved considerably, I simply concatenated them in a single fastq file to ease downstream processing.  

# Quality control - before trimming raw ONT reads - NanoPack
```
cd raw_data/concatenated # create this dir inside data
mkdir -p nanoqc_out
# NanoPlot
NanoPlot -o nanoqc_out/ --no_static --tsv_stats --N50 --threads 5 --fastq *.fastq 

# NanoComp
NanoComp -o nanoqc_out/ -t 5 --tsv_stats --make_no_static --fastq *.fastq

# nanoQC
for file in *.fastq; do
    nanoQC -o nanoqc_out/ -l 400 "$file" 
done
```

# Trimming raw files - Chopper
```
mkdir -p chopper_filter
for file in *.fastq; do
    chopper --quality 12 --minlength 1200 --maxlength 1800 --headcrop 100 --tailcrop 200 --threads 5 --input "$file" > chopper_filter/"${file%.fastq}_filtered.fastq"
done
```
## Count number of retained reads after 
Check script 04.Chopper_counts.ipynb

# Quality control - after trimming raw ONT reads - NanoPack
```
cd raw_data/concatenated/chopper_filter # create this dir inside data
mkdir -p nanoqc_out
# NanoPlot
NanoPlot -o nanoqc_out/ --no_static --tsv_stats --N50 --threads 5 --fastq *.fastq 

# NanoComp
NanoComp -o nanoqc_out/ -t 5 --tsv_stats --make_no_static --fastq *.fastq

# nanoQC
for file in *.fastq; do
    nanoQC -o nanoqc_out/ -l 400 "$file" 
done
```
# Visualize the distribution of the number of reads across samples
Check script: 05.Visualize_nreads_distribution.ipynb

# SILVA 16S Database downloaded for annotation
## Download and extract full SILVA 16S database 
https://www.arb-silva.de/no_cache/download/archive/current/Exports/ ## choose SILVA_138.2_SSURef_NR99_tax_silva.fasta.gz or the current version 

```
gunzip SILVA_138.2_SSURef_NR99_tax_silva.fasta.gz # move it to data dir and unzip it
```

## Pre-process db before annotation
### Filter out eukaryotic sequences 
```
awk '/^>/ {p = ($0 ~ /Bacteria;/)} p' SILVA_138.2_SSURef_NR99_tax_silva.fasta > SILVA138.2_16S_Bacteria.fasta 
```

### Make a taxonomical table linked with the ID from Silva
```
grep "^>" SILVA138.2_16S_Bacteria.fasta > headers_silva.txt
awk -F' ' '{print $1 "," substr($0, index($0,$2))}' headers_silva.txt > split_headers.csv
sed -i 's/;/,/g' split_headers.csv 
echo "id,Kingdom,Phylum,Class,Order,Family,Genus,Species" > silva_taxonomy.csv
sed 's/^>//' split_headers.csv >> silva_taxonomy.csv
```

### Remove taxonomical information from SILVA fasta headers
```
sed -i 's/ .*//' SILVA138.2_16S_Bacteria.fasta
```

## Check number of bacterial sequences before and after filtering 
```
grep -c "Bacteria" SILVA_138.2_SSURef_NR99_tax_silva.fasta # 431166
grep -c "Bacteria" SILVA138.2_16S_Bacteria.fasta # 431166; both numbers should be the same 
```

## Sending all my filtered fastq files to my data dir
```
mkdir -p silva_16s_index_kma output_16s_silva filtered_fastq # inside data dir 
mv raw_data/filtered/*_filt.fastq ./filtered_fastq
```

## Index SILVA Database for KMA
```
kma_index -i SILVA138.2_16S_Bacteria.fasta  -o silva_16s_index_kma/SILVA_16S
```

## Map Reads to SILVA Database
```
for file in filtered_fastq/*.fastq; do
    filename=$(basename "$file")  # Extract the filename
    filename_no_ext="${filename%.fastq}"  # Remove the file extension
    kma -i "$file" -o "output_16s_silva/${filename_no_ext}_kma" -t_db silva_16s_index_kma/SILVA_16S -bcNano -bc 0.7 -ID 90 -ml 400 -1t1 -t 5
done
```

## Extract and Move Frag Files for Taxonomy Assignment
```
gunzip output_16s_silva/*.frag.gz
mv output_16s_silva/*.frag ./data/
```

# Taxonomy assignation to nanopore reads
Check script: 06.Assin_Taxonomy.ipynb

# Diversity analysis
Check scripts:
07.Diversity_preprocess_rarefy.ipynb
08.Diversity_alpha.ipynb
09.Diversity_beta.ipynb

# Visualize taxonomy at different taxonomical levels
Check script: 10.Visualize_Taxonomy.ipynb

# Correlation analyses
## Numeric variables
Check script: 11.Corr_analysis1.ipynb

## Categoric variables
Check script: 12.Corr_analysis2.ipynb


























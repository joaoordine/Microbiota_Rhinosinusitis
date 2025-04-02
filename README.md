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

# Custom database creation
Used the same files used to annotate reads from mangrove sediments (16S RefSeq DB - NCBI)
*Files to copy*: 16Ssequences.fasta, new_taxdump.tar.gz (uncompressed files), RefseqTaxID.txt, referencetable_taxonomy_RefseqNCBI_16S.txt
*Copy from*: https://github.com/joaoordine/mangrove_microbiota/tree/main - /data/ folder

## Sending all my filtered fastq files to my data dir
```
mkdir filtered_fastq # inside data dir
mkdir 16sequences_index_kma
mkdir output_16s_refseq
mv raw_data/filtered/*_filt.fastq ./filtered_fastq
```

## Create the databases needed to run KMA from a list of FASTA files
```
kma_index -i 16Ssequences.fasta -o 16sequences_index_kma/16Ssequences 
```

## Map and/or align raw reads to a template database created using kma_index
```
for file in filtered_fastq/*.fastq; do
    filename=$(basename "$file")  # Extract the filename
    filename_no_ext="${filename%.}"  # Remove the file extension
    kma -i "$file" -o "output_16s_refseq/${filename_no_ext}_kma" -t_db 16sequences_index_kma/16Ssequences -bcNano -bc 0.7
done 
```

## Unzipping frag files to use as input for taxonomical assignation 
```
gunzip ./*_kma.frag.gz 
mv ./*.frag .. # move them all into data folder to ease downstream coding
```

# Taxonomy assignation to nanopore reads
Check script: 04.Assin_Taxonomy.ipynb

# Visualize the distribution of the number of reads across samples
Check script: 04.Visualize_nreads_distribution.ipynb

# Diversity analysis
Check scripts:
05.Diversity_preprocess_rarefy.ipynb
06.Diversity_alpha.ipynb
07.Diversity_beta.ipynb




























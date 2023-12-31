# Wastewater virus surveillance analysis pipeline  
This bioinformatic pipeline is used to analyze targeted sequencing data of viruses generated from total nucleic acid extracted from wastewater samples using various concentration/extraction methods. The sequencing library was prepared with the Illumina Virus Surveillance Panel (probe-capture enrichment) and paired with RNA preparation using Enrichment and Tagmentation kits.

Notes - This Repo is still under active development and please refer to the updated version. 


## Introduction
This pipeline takes FASTQ files as input and includes both read-based and assembly-based virus classification. Before classification, quality control (QC) is performed, followed by deduplication to generate unique reads. The read-based analysis identifies viruses using Centrifuge and Recentrifuge, and the classified taxIDs were used for retrieving the virus-host information from NCBI taxonomy. The assembly-based classification involves full assembly by SPAdes, virus classification by Virsorter2, and subsequent Blastn against the NCBI nt virus database. The resulting near-complete virus genomes are utilized for phylogenetic analysis and variant calling.

Dataset and code for the manuscript: Evaluation of the impact of concentration and extraction methods on the targeted sequencing of human viruses from wastewater

![Bioinformatic pipeline_2-02](https://github.com/mj2770/Wastewater-virus-surveillance-/assets/45908853/417edc9a-5a8e-40d5-800b-2f976c8a1427)

## Download database and dependencies
### 1. Raw sequencing data 
All are deposited in the NCBI Sequence Read Archive (SRA) under accession number: SUB13892842 and Bioproject ID: PRJNA1047067.
### 2. Database
   * Refseq virus: download using [Entrez](https://www.ncbi.nlm.nih.gov/books/NBK25501/)
     ```
     "Viruses"[Organism] NOT "cellular organisms"[Organism] NOT wgs[PROP] NOT gbdiv syn[prop] AND (srcdb_refseq[PROP] OR nuccore genome samespecies[Filter])
     ```
   * [NCBI nt-virus](https://ftp.ncbi.nlm.nih.gov/blast/db/)
   * Centrifuge and Recentrifuge default database: [NCBI nt decontaminated version](https://github.com/khyox/recentrifuge/wiki/Centrifuge-nt)
### 3. Dependent packages
   * [BBduk from the BBTools suite](https://jgi.doe.gov/data-and-tools/software-tools/bbtools/bb-tools-user-guide/bbduk-guide/)
   * [Seqkit](https://github.com/shenwei356/seqkit/blob/master/README.md)
   * [Centrifuge and Recentrifuge](https://github.com/khyox/recentrifuge/wiki/Running-recentrifuge-for-Centrifuge) 
   * [MASH](https://github.com/marbl/Mash): Fast genome and metagenome distance estimation using MinHash   
   * [bowtie2 (v2.5.1)](https://github.com/BenLangmead/bowtie2), [Samtools, bcftools, and htslib](https://www.htslib.org/download/) 
   * [VirSorter2 (v2.2.4)](https://github.com/jiarong/VirSorter2#detailed-description-on-output-files) 
   * [BLASTn (v2.14.0+)](https://ftp.ncbi.nlm.nih.gov/blast/executables/blast+/LATEST/)
   * [MUSCLE](https://github.com/rcedgar/muscle) and [Gblock 0.91b](http://phylogeny.lirmm.fr/phylo_cgi/one_task.cgi?task_type=gblocks)
   * [Instrain](https://github.com/MrOlm/inStrain/blob/master/docs/user_manual.rst)
   * [Dataset](https://www.ncbi.nlm.nih.gov/datasets/docs/v2/download-and-install/) and [Dataformat](https://www.ncbi.nlm.nih.gov/datasets/docs/v2/reference-docs/command-line/dataformat/)
   * Jupyter notebook: pandas, numpy, plotnine, seaborn, scipy.spatial etc. (All python scripts were uploaded as [google colab scripts](https://colab.google/))
     
## Basic analysis pipeline
### STEP 1. Quality control 
Step 1.1 Quality filter of raw reads
```
bbduk.sh in1=$f in2=$f2 out1=$out_path".forward" out2=$out_path".reverse" \
    outm1=$out_path".forward.unpaired" outm2=$out_path".reverse.unpaired" \
    ref=adapters ktrim=r k=23 mink=11 hdist=1 qtrim=r:4:10 tpe tbo minlen=70
```
Step 1.2 Dedupe the good quality reads
```
seqkit rmdup -s "$cleaned_file" -o "$OUT_DIR/STEP_1/$filename.forward.deduped"
```
Step 1.3 Paired the deduped reads
```
seqkit pair -1 "$forward_file" -2 "$reverse_file" -O "$output_dir" -u
```
Step 1.4 Statistical analysis of all reads
```
 seqkit stats -j 100 "$file" -a > "temp_stats.txt
```
### 2. Read-based classification by Centrifuge and Recentrifuge 
```
# Centrifuge
time $cf_path/centrifuge -x  /p/lustre2/metagen/dbs/nt_wntr23/bld/nt_wntr23 --sample-sheet $config -p 256 -t --min-hitlen 15 -k 1
# Recentrifuge
$rcf_insall_path/rcf -n $taxonomy_dir -f $f1 -e TSV -o "$(basename $f1 _mhl22.out)_mhl40.out" -y 40
```

### 3. Analysis of the read-based classification results
    * Taxonomy domain analysis 
    * Virus reads host-screen 
    * Virus reads genotype analysis (DNA and RNA type)
    * Virus species richness and composition analysis 
    * Virus genome similarity PCoA analysis (MASH distance)
       ```
       # re-extract all virus sequences
       rextract -f "$classification_output" -i "$sample_id" -1 "$fastq1" -2 "$fastq2"
       #Concatenate forward and reverse files
       cat "$forward_file" "$reverse_file" > "$output_dir/${sample_name}.concatenated.fastq"
       #Use Mash to sketch concatenated file
       "$mash_path" sketch "$output_dir/${sample_name}.concatenated.fastq" -o "$output_dir/${sample_name}.msh"
       #Calculate MASH pairwise distances
       "$mash_path" dist "${sample_names[$i]}.msh" "${sample_names[$j]}.msh" 
        ```
### 4. Assembly-based analysis 
Step 4.1 Assembly using SPAdes
```
python3 "$SPADES_PATH" --meta -1 "$f" -2 "$r" -o "$OUT_DIR/$output_name.assembled"
```
Step 4.2 Classification using Virsorter2
```
virsorter run --prep-for-dramv -w "$output_subdir" -i "$input_file" --include-groups "dsDNAphage,lavidaviridae,NCLDV,RNA,ssDNA" -j 128 all
```
Step 4.3 Blastn for assemblies
```
"$blastn_path" -query "$input_file" -db "$db_path" -out "$output_file" -outfmt 6 \
  -evalue 1e-8 -perc_identity 80 -max_hsps 1 -qcov_hsp_perc 90
```
Step 4.4 Quality filter for the blastn results 
```
# Read the BLASTN output file as a DataFrame
blastn_data = pd.read_csv(blastn_file, sep='\t', header=None)

# Extract the coverage value from the first column
blastn_data['Coverage'] = blastn_data[0].str.extract(r'cov_([\d.]+)').astype(float)

# Apply the QC criteria to select the good quality reads
selected_reads = blastn_data[(blastn_data[2] > min_identity) &
                             (blastn_data[3] > min_alignment_length) &
                             (blastn_data[10] < max_evalue) &
                             (blastn_data['Coverage'] > min_coverage)]
```
Step 4.5 Select the best hits for each assembly based on the bitscore
```
# Sort the DataFrame by the Bitscore column in descending order
sorted_data = blastn_data.sort_values(by=11, ascending=False)

# Drop duplicates based on the unique Node (column 0) while keeping only the first occurrence
selected_reads = sorted_data.drop_duplicates(subset=0, keep='first')
```
Step 4.6 Collect the best-hits accession numbers 
```
# Save the selected hits to a file
output_file = os.path.join(output_directory, f"{base_name}_best_hits.blastn")
selected_reads.to_csv(output_file, sep='\t', index=False, header=False)
```
Step 4.7 Qurey the database using NCBI dataset and dataformat 
```
"$DATASETS_PATH" summary virus genome accession --inputfile "$input" --as-json-lines | "$DATAFORMAT_PATH" tsv virus-genome --fields accession,virus-name,virus-tax-id,host-name,host-tax-id,completeness,length > best_hits.tsv
```
### 5. Subtyping and variants calling 
STEP 5.1 Calculate the variants allel frequency using inStrain
```
# Extract the sample name from the input BAM filename
sample_name=$(basename "${input_bam%.*}" | sed 's/sorted_aligned_//')

# Define output VCF filename
output_vcf="${sample_name}_variant_call_trial.pass.vcf"

# Run samtools mpileup and pipe the output to ivar variants
$SAMTOOLS_PATH mpileup -A -B -Q 0 -f "$reference" "$input_bam" | \
$IVAR_PATH variants -p "$sample_name"_variant_call -q 10 -t 0.01 -m 10 -r "$reference"
```
Step 5.2 Phylogenetic analysis
```
# Perform Multiple Sequence Alignment (MSA) with MUSCLE
#$MUSCLE_PATH -in final.fas -out final_MUSCLE.fasta

# Generate Phylogenetic Tree with FastTree
$FASTTREE_PATH  -nt -gtr -boot 100 final_MUSCLE.fasta > tree.nwk
```

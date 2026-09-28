# Basic_Protocol_1_Extracting_participants_variants_of_interest.ipynb

## 1. Create a “JupyterLab Spark Cluster for AoU” App instance under the “Apps” tab in the workspace environment. 
Use the configuration described in “Materials,” Application Instance 1

      JupyterLab Spark cluster for AoU
      Master Node in “n2-standard-2” configuration, 1000 GB boot disk 
      8 Primary Worker nodes with n2-standard-8 configuration,  500 GB boot disk
      Hail (Spark 3.5.3, hail 0.2.135) Installed
      $4.07/hr

## 2. Set up your Python environment by defining environmental variables. See Support Protocol 1 and Support_Protocol_1_Setting_up_python_environment.ipynb
```
%run Support_Protocol_1_Setting_up_python_environment.ipynb
```
## 3. Import the variant annotation table, select genes for analysis and export all variants as a table
At this point _all_ variants are exported to allow for subsequent alternative filtering, and to ease future computational workload. This step takes over 30 minutes to complete, so we recommend exporting an intermediate file with all variants in the gene of interest at this point to save time in pipeline development and/or reanalysis. These commands as executed export 376,663 VAT annotation records in CFTR.
(To select multiple genes, use a comma-separated list here, i.e. [‘CFTR’, ’NF1’]).

```
# Read in variant table
vat_table = hl.import_table(vat_path, force=True, quote='"', delimiter="\t", force_bgz=True)
# Export variants in genes of interest 
genes = ['CFTR']
test_genes = vat_table.filter(hl.set(genes).contains(vat_table.gene_symbol), keep=True)
out_path = f'{bucket}/VAT_screen_CFTR.tsv'
test_genes.export(out_path)
```

## 4. Read in the reduced file with only CFTR variants for subsequent steps and key by the variant column “vid.”

```
CFTR_vat_path = f'{bucket}/VAT_screen_CFTR.tsv'
CFTR_vat_table = hl.import_table(CFTR_vat_path, force=True, quote='"', delimiter="\t", force_bgz=True)
CFTR_vat_table = CFTR_vat_table.key_by("vid")
```

## 5. Select variants with a pathogenic or likely pathogenic assignation in ClinVar

```
clinvar_high = ['likely pathogenic', 'pathogenic', 'likely pathogenic,pathogenic', 'likely pathogenic,pathogenic,drug response', 'pathogenic,drug response']
CFTR_ClinVar = CFTR_vat_table.filter(hl.set(clinvar_high).contains( CFTR_vat_table.clinvar_classification), keep=True)
```

## 6. Select variants with predicted loss-of-function biological consequences. 
Here we select frameshift variants, stop gained variants, and start or stop lost variants, as well as variants that fall in these categories that may also affect splicing.

```
conseq_high = ['frameshift_variant', 'frameshift_variant, splice_region_variant', 'stop_gained', 'stop_gained,splice_region_variant', 'stop_lost', 'start_lost']
CFTR_conseq = CFTR_vat_table.filter(hl.set(conseq_high).contains( CFTR_vat_table.consequence), keep=True)
```

## 7. Select variants predicted to be high confidence (“HC”) loss-of-function given the LoFTEE predictor

```
HC = ['HC']
CFTR_HC = CFTR_vat_table.filter(hl.set(HC).contains(CFTR_vat_table.LoF), keep=True)
```

## 8. Join the tables that fit these criteria set in Steps 5-7 and remove duplicates.

```
CFTR_path = CFTR_ClinVar.union(CFTR_conseq).union(CFTR_HC)
CFTR_path = CFTR_path.distinct()
```

## 9. Format the table for export as a VCF. This includes changing the “position” column to an integer format, and creating (and keying by) “locus” and “alleles” columns

```
CFTR_path = CFTR_path.annotate(position = hl.int(CFTR_path.position))

CFTR_path = CFTR_path.annotate(
    locus=hl.locus(CFTR_path.contig, CFTR_path.position, reference_genome="GRCh38"),
    alleles=[CFTR_path.ref_allele, CFTR_path.alt_allele])

CFTR_path = CFTR_path.key_by("locus", "alleles")
```

## 10. Export the variant set as a sample-free VCF. This VCF contains 442 variants of interest. See “Understanding Results” Table 1 for sample output.

```
vcf_path = f'{bucket}/CFTR_path.vcf' 
hl.export_vcf(CFTR_path, vcf_path)
```

# Retrieve participant-level genotypes for with variants of interest

## 11. Read in the VDS, using the path established in “Setting_up_python_environment.ipynb”

```
vds = hl.vds.read_vds(vds_path)
```

## 12. Set CFTR coordinates to screen the VDS.

```
coord = ['chr7:117480025-117668665']
intervals = [hl.parse_locus_interval(x, reference_genome='GRCh38') for x in coord]
```

## 13. Filter the matrix table for the intervals defined.

```
vds = hl.vds.filter_intervals(vds, intervals)
```

## 14. Annotate the VDS format so it more closely fits the parameters of a VCF. 
This fills out missing alleles, standardizes allele calls across the VDS, and converts the VDS to a matrix table (mt).

```
mt = vds.variant_data.annotate_entries(AD = hl.vds.local_to_global(vds.variant_data.LAD,
        vds.variant_data.LA,
        n_alleles=hl.len(vds.variant_data.alleles),
        fill_value=0, number='R'))
```

## 15. Split multi-allelic calls.

```
mt = mt.annotate_entries(GT=hl.vds.lgt_to_gt(mt.LGT, mt.LA))
mt = hl.split_multi_hts(mt)
mt = mt.filter_entries(mt.GT.is_non_ref())
```

## 16. Read in the vcf of putatively pathogenic calls
VCF exported above in “Filter for putatively pathogenic variants in gene of interest." Filter the matrix table for those variants

```
vcf_path = f'{bucket}/CFTR_path.vcf' 
ds = hl.import_vcf(vcf_path, reference_genome='GRCh38')
ht_PLP = ds.rows().key_by("locus", "alleles")
mt = mt.filter_rows(hl.is_defined(ht_PLP[mt.row_key]))
```

## 17. Filter out calls with no alternate allele and select entries to export. 
GT is the genotype, AD is the allele depth, PS is the phase set, and FT is whether the call passed _All of Us_ genotype-level filtering criteria

```
mt = mt.select_entries(GT = mt.GT, AD = mt.AD, PS = mt.PS, FT =  mt.FT)
mt = mt.select_cols()
```

## 18. Convert the MatrixTable to a table and export the participant-variant entries as a table. 
This table contains 45,220 individuals that are harboring a variant of interest. See “Understanding Results” Table 2 for sample output
(This is an appropriate stopping point for the analysis, as needed)

```
ht = mt.entries()
vcf_PLP_gt_path = f'{vcf_path}.vds_screen.tsv'
ht.export(vcf_PLP_gt_path)
```
# Call filtering for quality control and relatedness
(_This process is in the R programing language for ease of plotting and transitioning to other analysis. Use Application Instance 2 for the below analysis_)

## 19. Create an “R Analysis Environment for AoU” App instance under the “Apps” tab in the workspace environment. 
Use the configuration described in “Materials,” Application Instance 2

      R Analysis Environment for AoU 
      Master Node in “n1-highmem-4” configuration, 500 GB boot disk
      4 CPUs, 26 GB memory
      $0.27/hr


## 20. Set up environmental variables and load in the “tidyverse” and “bigquery” packages. See Support Protocol 2 and Support_Protocol_2_Setting_up_R_environment.R 

```
source("/home/rstudio/workspace/workspace-bucket/Support_Protocol_2_Setting_up_R_environment.R")
```

## 21. Read in the file generated in “Extract person_id of  participants with variants of interest,“ and examine the number of rows, loci, and participants from the screen of the VDS

```
CFTR_path <- read.csv("CFTR_path.vcf.vds_screen.tsv", sep = "\t")
number_total <- dim(CFTR_path)[1]
number_loci <- dim(table(CFTR_path$locus))[1]
number_participants <- dim(table(CFTR_path$s))[1]

print(paste0("There are a total of ", number_total, " rows"))
print(paste0("There are ", number_loci, " loci"))
print(paste0("There are ", number_participants, " participants"))
```

## 22.Parse locus, allele balance and read depth.
The below code creates a variant identifier in a chrom-position-ref-alt format, as well as separating variant call data. 

```
variant_out <- CFTR_path
variant_out$chr <- vapply(strsplit(variant_out$locus,":"), `[`, 1, FUN.VALUE=character(1))
variant_out$pos <- vapply(strsplit(variant_out$locus,":"), `[`, 2, FUN.VALUE=character(1))
variant_out$ref <- vapply(strsplit(variant_out$alleles, ","), `[`, 1, FUN.VALUE=character(1))
variant_out$alt <- vapply(strsplit(variant_out$alleles, ","), `[`, 2, FUN.VALUE=character(1))
variant_out$ref <- gsub('\\[', '', variant_out$ref)
variant_out$alt <- gsub('\\]', '', variant_out$alt)
variant_out$ID <- paste0(variant_out$chr, "-", variant_out$pos, "-", variant_out$ref, "-", variant_out$alt)
variant_out$ref_call <- vapply(strsplit(variant_out$AD, ","), `[`, 1, FUN.VALUE=character(1))
variant_out$alt_call <- vapply(strsplit(variant_out$AD, ","), `[`, 2, FUN.VALUE=character(1))
variant_out$ref_call <- as.numeric(gsub('\\[', '', variant_out$ref_call))
variant_out$alt_call <- as.numeric(gsub('\\]', '', variant_out$alt_call))
variant_out$total <- variant_out$ref_call + variant_out$alt_call
variant_out$AB <- variant_out$alt_call / variant_out$total
print("Summary statistics for allele balance")
summary(variant_out$AB)

print("Summary statistics for read depth")
summary(variant_out$total)
```

## 23. For the purposes of this analysis, remove calls with an allele balance less than 0.30 (i.e. alternate allele calls are less than 30% of the total calls for that position) as well as calls with less than 15 total reads.
This is done as a conservative measure to ensure high-quality alternate allele calls. 

```
variant_out_strict <- variant_out[which(variant_out$AB >= 0.30 & variant_out$total >= 15 & variant_out$FT == “true”),]
new_rows <- dim(variant_out_strict)[1]
print(paste0(new_rows, " of the original ", number_total, " rows remain after filtering"))
print("New summary statistics for allele balance")
summary(variant_out_strict$AB)
print("New summary statistics for read depth")
summary(variant_out_strict$total)
```

## 24, To visualize the distribution of allele balance and read depth, generate the following plots (Figure 1A and B).

```
options(repr.plot.width=7, repr.plot.height=5)

AB_plot <- ggplot(variant_out, aes(x = AB)) + xlim(-.05,1.3) +
            geom_histogram(bins = 50, aes(y=..density..), alpha=0.8, fill = "#345C75FF", color = "black") +
            geom_density(alpha=.4) + theme_classic() + theme(axis.title.y=element_blank(),
            axis.text.y=element_blank(),
            axis.ticks.y=element_blank(),
            axis.text.x=element_text(size=16),
            axis.title.x=element_text(size=20)) + geom_vline(xintercept=0.3) + xlab("Allele Balance")

pdf(“AB_plot.pdf”)
AB_plot
dev.off()
depth_plot <- ggplot(variant_out, aes(x = total)) + xlim(0,100) +
            geom_histogram(bins = 50, aes(y=..density..), alpha=0.8, fill = "#C3AF97FF", color = "black") +
            geom_density(alpha=.4) + theme_classic() + theme(axis.title.y=element_blank(),
            axis.text.y=element_blank(),
            axis.ticks.y=element_blank(),
            axis.text.x=element_text(size=16),
            axis.title.x=element_text(size=20)) + geom_vline(xintercept=15) + xlab("Depth")  
pdf(“total_read_depth_plot.pdf”)
depth_plot 
dev.off()
```

## 25. Simplify the table and add All of Us variant count.
```
working_chr <- unique(variant_out_strict[,c("ID", "s", "GT", "AB", "total")])
variant_table <- data.frame(table(working_chr$ID))
variant_table$Var1 <- as.character(variant_table$Var1)
working_chr2 <- full_join(working_chr, variant_table, by = c("ID"="Var1"))
names(working_chr2)[names(working_chr2) == "Freq"] <- "AoU_Freq"
```

## 26. Remove individuals flagged in the All of Us sequencing pipeline. Depending on the analysis, include or exclude related individuals. In this example, we also exclude related individuals. Lists of related individuals and individuals that failed quality control in the “vwb-aou-datasets-controlled-v9/v9/wgs/short_read/snpindel/aux” directory within the version 9 release of the All of Us CDR. 

```
related <- read.csv("/home/rstudio/workspace/vwb-aou-datasets-controlled-v9/v9/wgs/short_read/snpindel/aux/relatedness/relatedness_flagged_samples.tsv", sep = "\t")
qc_flagged <- read.csv("/home/rstudio/workspace/vwb-aou-datasets-controlled-v9/v9/wgs/short_read/snpindel/aux/qc/flagged_samples.tsv", sep = "\t")
flagged <- data.frame(c(related$sample_id, qc_flagged$s))
names(flagged) <- "s"
samples_pass <- anti_join(working_chr2, flagged, by = c("s"))
```

## 27. Write this file for future analysis. This file will have the same format as Table 2 in “Understanding Results,” with filtered variants. 

```
destination_filename <- "CFTR_working_QC.tsv" 
write.table(samples_pass, destination_filename, sep = "\t", col.names = TRUE, row.names = FALSE, quote = FALSE)
```

## 28. Isolate individuals with different pathogenic variants. 
As cystic fibrosis is an autosomal recessive condition, we are primarily interested in individuals with homozygote alternate calls or compound heterozygotes. Below we isolate a list of individuals with the deltaF508 CFTR variants, which is responsible for 30-70% of cystic fibrosis cases, depending on ancestry. 

```
deltaF508 <- samples_pass[samples_pass$ID == "chr7-117559590-ATCT-A", ]
deltaF508_homozygotes <- deltaF508[grepl("^1[/|]1$", deltaF508$GT), ]
length(unique(deltaF508_homozygotes$s))
write.table(deltaF508_homozygotes, "deltaF508_homozygotes.tsv", sep = "\t", quote= FALSE, col.names = TRUE, row.names = FALSE)
```

Additionally, extract other individuals with homozygous alternate variants:

```
>>> other_homozygotes <-  samples_pass[samples_pass$ID != "chr7-117559590-ATCT-A" & grepl("^1[/|]1$", samples_pass$GT), ]
>>> write.table(other_homozygotes, "other_CFTR_homozygotes.tsv", sep = "\t", quote= FALSE, col.names = TRUE, row.names = FALSE)
```
As well as possible compound heterozygotes: 
(_phased data is required for determining compound heterozygous state (i.e.., 0|1, 1|0). The below command does not take phasing into consideration_)

```
compound_hets <- samples_pass %>%
      group_by(s) %>%
      filter(n() > 1) %>%
      ungroup() %>% 
      arrange(s) 
write.table(compound_hets, "possible_compound_hets.tsv", sep = "\t", quote= FALSE, col.names = TRUE, row.names = FALSE)
```

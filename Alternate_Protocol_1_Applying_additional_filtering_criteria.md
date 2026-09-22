# Alternate Protocol 1 - Applying additional filtering criteria using the Variant Annotation Table and external annotations
Alternate_Protocol_Steps_1_2_Applying_additional_filtering_criteria.ipynb
The VAT contains a catalog of parameters to filter variants, including allele counts and frequencies, predicted biological consequence, and variant scores such as SpliceAI and REVEL, (list of annotations available:  https://support.researchallofus.org/hc/en-us/articles/4615256690836-Variant-Annotation-Table). Step 1 and 2 be selection criteria used in in Basic Protocol 1. 

We include these steps in this notebook, as these alternate filtering strategies can be implemented in Steps 3 and 5-7 of “Filter for putatively deleterious variants in gene of interest,” as documented in Step 1 and 2 of Alternate Protocol 1. 

```
%run Support_Protocol_1_Setting_up_python_environment.ipynb
```
```
CFTR_vat_path = f'{bucket}/VAT_screen_CFTR.tsv'
CFTR_vat_table = hl.import_table(CFTR_vat_path, force=True, quote='"', delimiter="\t", force_bgz=True)
CFTR_vat_table = CFTR_vat_table.key_by("vid")
```

## 1. In Basic Protocol 1, the criteria filtered upon was a “string” data type. Below, we demonstrate filtering on integer and float data types as well.
(Optional filtering after Step 4 in Basic Protocol 1)

Filtering on variants with an allele count of less than or equal to 100, using the gvs_all_ac field (integer data type) which is the count of alternate alleles across the short-read WGS callset. 

```
CFTR_vat_table = CFTR_vat_table.annotate(gvs_all_ac = hl.parse_float(CFTR_vat_table.gvs_all_ac))
CFTR_ac_100 = CFTR_vat_table.filter(CFTR_vat_table.gvs_all_ac <= 100)
CFTR_ac_100.count()
```

Filtering on variants with REVEL scores (a pathogenicity predictor for missense variants (Ioannidis, 2016)) of greater than 0.10. 

```
CFTR_revel_010 = CFTR_vat_table.filter(CFTR_vat_table.revel >= 0.10)
```

## 2. Filter VAT using genomic regions instead of gene name 

```
# Read in variant table
vat_table = hl.import_table(vat_path, force=True, quote='"', delimiter="\t", force_bgz=True)

# Filter on genomic position
regions_list = [("chr7",117480025, 117668665)]
intervals = [hl.parse_locus_interval(f"{chrom}:{start}-{end}", reference_genome='GRCh38') for chrom, start, end in regions_list]
vat_table = vat_table.annotate(position = hl.int32(vat_table.position))
vat_table = vat_table.annotate(locus = hl.locus(vat_table.contig, vat_table.position, reference_genome='GRCh38'))
vat_table = vat_table.filter(hl.literal(intervals).any(lambda i: i.contains(vat_table.locus)))
```

## 3. External Annotations
(_completed in R_)
If a specific desired annotation is absent from the VAT, add external annotations. Below, we added CADD score accessible from https://cadd.bihealth.org/download. 
(This file is over 81G, so allocate your local compute resources accordingly, and ensure you download the tabix file as well. WE conducted the following on a high-performance cluster. CADD is publicly available resource for non-commercial research, but if you are using CADD for commercial purposes, you must obtain a license. Contact the developers here: https://cadd.bihealth.org/contact)

Extract CADD scores for the region of interest. In this example, we used the CFTR genomic coordinates, and extracted in a bash using tabix. 

```
tabix cadd_1.7/whole_genome_SNVs.tsv.gz 7:117480025-117668665 > CFTR_SNV_CADD.txt
```

Upload this file into your workspace bucket. In Figure 2A, we provide a screen shot of the process in RStudio, using Application Instance 2. 
(Note the warning dialog box in Figure 2B and ensure you are always abiding by All of Us Data Use Policies.)  

Harmonize this data with output from Step 26 in Basic Protocol 1. 

```
x <- read.csv("CFTR_SNV_CADD.txt", sep = "\t", header = FALSE)
x$ID <- paste0("chr", x$V1, "-", x$V2, "-", x$V3, "-", x$V4)
CFTR_out <- unique(x[,c("ID", "V6")])
samples_pass_CADD <- left_join(samples_pass, CFTR_out, by = "ID")
```

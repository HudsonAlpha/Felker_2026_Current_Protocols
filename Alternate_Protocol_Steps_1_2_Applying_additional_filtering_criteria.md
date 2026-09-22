# Alternate Protocol 1 - Applying additional filtering criteria using the Variant Annotation Table and external annotations)
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


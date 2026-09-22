## Basic_Protocol_1_Steps_2-18_Extracting_participants_variants_of_interest.ipynb

### 1. Create a “JupyterLab Spark Cluster for AoU” App instance under the “Apps” tab in the workspace environment, as described in “Materials” – Application Instance 1

      JupyterLab Spark cluster for AoU
      Master Node in “n2-standard-2” configuration, 1000 GB boot disk 
      8 Primary Worker nodes with n2-standard-8 configuration,  500 GB boot disk
      Hail (Spark 3.5.3, hail 0.2.135) Installed
      $4.07/hr

### 2. Set up your Python environment by defining environmental variables. See Support Protocol 1 and Support_Protocol_1_Setting_up_python_environment.ipynb

%run Support_Protocol_1_Setting_up_python_environment.ipynb

### 3. Import the variant annotation table, select genes for analysis and export all variants as a table
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

### 4. Read in the reduced file with only CFTR variants for subsequent steps and key by the variant column “vid.”

```
CFTR_vat_path = f'{bucket}/VAT_screen_CFTR.tsv'
CFTR_vat_table = hl.import_table(CFTR_vat_path, force=True, quote='"', delimiter="\t", force_bgz=True)
CFTR_vat_table = CFTR_vat_table.key_by("vid")
```

### 5. Select variants with a pathogenic or likely pathogenic assignation in ClinVar

```
clinvar_high = ['likely pathogenic', 'pathogenic', 'likely pathogenic,pathogenic', 'likely pathogenic,pathogenic,drug response', 'pathogenic,drug response']
CFTR_ClinVar = CFTR_vat_table.filter(hl.set(clinvar_high).contains( CFTR_vat_table.clinvar_classification), keep=True)
```

### 6. Select variants with predicted loss-of-function biological consequences. 
Here we select frameshift variants, stop gained variants, and start or stop lost variants, as well as variants that fall in these categories that may also affect splicing.

```
conseq_high = ['frameshift_variant', 'frameshift_variant, splice_region_variant', 'stop_gained', 'stop_gained,splice_region_variant', 'stop_lost', 'start_lost']
CFTR_conseq = CFTR_vat_table.filter(hl.set(conseq_high).contains( CFTR_vat_table.consequence), keep=True)
```

### 7. Select variants predicted to be high confidence (“HC”) loss-of-function given the LoFTEE predictor

```
HC = ['HC']
CFTR_HC = CFTR_vat_table.filter(hl.set(HC).contains(CFTR_vat_table.LoF), keep=True)
```

### 8. Join the tables that fit these criteria set in Steps 5-7 and remove duplicates.

```
CFTR_path = CFTR_ClinVar.union(CFTR_conseq).union(CFTR_HC)
CFTR_path = CFTR_path.distinct()
```

### 9. Format the table for export as a VCF. This includes changing the “position” column to an integer format, and creating (and keying by) “locus” and “alleles” columns

```
CFTR_path = CFTR_path.annotate(position = hl.int(CFTR_path.position))

CFTR_path = CFTR_path.annotate(
    locus=hl.locus(CFTR_path.contig, CFTR_path.position, reference_genome="GRCh38"),
    alleles=[CFTR_path.ref_allele, CFTR_path.alt_allele])

CFTR_path = CFTR_path.key_by("locus", "alleles")
```

### 10. Export the variant set as a sample-free VCF. This VCF contains 442 variants of interest. See “Understanding Results” Table 1 for sample output.

```
vcf_path = f'{bucket}/CFTR_path.vcf' 
hl.export_vcf(CFTR_path, vcf_path)
```

## Retrieve participant-level genotypes for with variants of interest

### 11. Read in the VDS, using the path established in “Setting_up_python_environment.ipynb”

```
vds = hl.vds.read_vds(vds_path)
```

### 12. Set CFTR coordinates to screen the VDS.

```
coord = ['chr7:117480025-117668665']
intervals = [hl.parse_locus_interval(x, reference_genome='GRCh38') for x in coord]
```

### 13. Filter the matrix table for the intervals defined.

```
vds = hl.vds.filter_intervals(vds, intervals)
```

### 14. Annotate the VDS format so it more closely fits the parameters of a VCF. 
This fills out missing alleles, standardizes allele calls across the VDS, and converts the VDS to a matrix table (mt).

```
mt = vds.variant_data.annotate_entries(AD = hl.vds.local_to_global(vds.variant_data.LAD,
        vds.variant_data.LA,
        n_alleles=hl.len(vds.variant_data.alleles),
        fill_value=0, number='R'))
```

### 15. Split multi-allelic calls.

```
mt = mt.annotate_entries(GT=hl.vds.lgt_to_gt(mt.LGT, mt.LA))
mt = hl.split_multi_hts(mt)
mt = mt.filter_entries(mt.GT.is_non_ref())
```

### 16. Read in the vcf of putatively pathogenic calls
VCF exported above in “Filter for putatively pathogenic variants in gene of interest." Filter the matrix table for those variants

```
vcf_path = f'{bucket}/CFTR_path.vcf' 
ds = hl.import_vcf(vcf_path, reference_genome='GRCh38')
ht_PLP = ds.rows().key_by("locus", "alleles")
mt = mt.filter_rows(hl.is_defined(ht_PLP[mt.row_key]))
```

### 17. Filter out calls with no alternate allele and select entries to export. 
GT is the genotype, AD is the allele depth, PS is the phase set, and FT is whether the call passed _All of Us_ genotype-level filtering criteria

```
mt = mt.select_entries(GT = mt.GT, AD = mt.AD, PS = mt.PS, FT =  mt.FT)
mt = mt.select_cols()
```

### 18. Convert the MatrixTable to a table and export the participant-variant entries as a table. 
This table contains 45,220 individuals that are harboring a variant of interest. See “Understanding Results” Table 2 for sample output
(This is an appropriate stopping point for the analysis, as needed)

```
ht = mt.entries()
vcf_PLP_gt_path = f'{vcf_path}.vds_screen.tsv'
ht.export(vcf_PLP_gt_path)
```

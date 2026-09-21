# Felker_2026_Current_Protocols
Code associated with Felker, et al.: Identifying and analyzing rare deleterious variation using the All of Us Researcher Workbench.

## ABSTRACT:
In this tutorial, we provide a reproducible workflow for the All of Us Researcher Workbench that first prioritizes variants in a gene or genomic region with the Variant Annotation Table (VAT) and then retrieves participant-level genotypes from the Hail VariantDataset (VDS). Using CFTR as an example, the protocol combines ClinVar classifications, predicted loss-of-function consequences, LoFTEE confidence, allele-count filters, and optional external annotations, followed by allele-balance, read-depth, sequencing-quality, and relatedness filters. The workflow emphasizes inheritance-aware interpretation: genotype restrictions should follow a prespecified disease model, and candidate compound heterozygotes require phase confirmation. With the configurations described here, the example workflow completes in approximately 2 hours at an estimated compute cost of about $7. This protocol provides an accessible, modifiable starting point for rare-variant studies in All of Us while preserving required privacy and dissemination controls.

## Protocols
**Basic Protocol 1**: Isolating Putatively Deleterious Variation in a gene or region of interest using the All of Us Variant Annotation Table and VariantDataset

**Alternate Protocol 1**: Alternate filtering criteria applicable to the Variant Annotation Table

**Support Protocol 1**: Setting up an environment for Python in JupyterLab Notebooks

**Support Protocol 2**: Setting up an environment for R in RStudio


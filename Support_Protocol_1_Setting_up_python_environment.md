### 1.	Create cloud storage buckets named “workspace-bucket” and “temporary-resource-bugivencket.” Click the “ + New Resource” button under the “Resources” tab, and selecting “New Cloud Storage Bucket”  from the drop-down menu


### 2. Import required python libraries

```
import json
import subprocess
import os
import re
import datetime
```
### 3. Set GOOGLE_CLOUD_PROJECT and WORKSPACE_CDR variables
```
### Extract variables directly into os.environ
workspace = json.loads(subprocess.run(
    ["wb", "workspace", "describe", "--format=json"],
    capture_output=True, text=True, check=True
).stdout)

os.environ["GOOGLE_CLOUD_PROJECT"] = workspace["googleProjectId"]

resources = json.loads(subprocess.run(
    ["wb", "resource", "list", "--format=json"],
    capture_output=True, text=True, check=True
).stdout)

os.environ["WORKSPACE_CDR"] = ""
```
## 4. Extract workspace resources
```
for r in resources:

    # 1. BUCKET LOGIC
    if r["resourceType"] == "GCS_BUCKET":
        print(f"Found bucket: id={r['id']}, bucketName={r['bucketName']}")
        if "temporary-workspace-bucket" in r["id"]:
            os.environ["WORKSPACE_TEMP_BUCKET"] = f"gs://{r['bucketName']}"
        elif "workspace-bucket" in r["id"]:
            os.environ["WORKSPACE_BUCKET"] = f"gs://{r['bucketName']}"


    # 2. BQ DATASET LOGIC  ← SAME LEVEL as bucket logic
    elif r["resourceType"] in ["BQ_DATASET", "BIGQUERY_DATASET"]:
        if os.environ.get("WORKSPACE_CDR", "") == "":
            dataset_id = r["datasetId"]
            # Match real All of Us CDR pattern like C2024Q3R8
            if re.match(r"^C\d{4}Q\d+R\d+$", dataset_id):
                os.environ["WORKSPACE_CDR"] = f"{r['projectId']}.{dataset_id}"
                print(f"Successfully set WORKSPACE_CDR to: {os.environ['WORKSPACE_CDR']}")

bucket = %env WORKSPACE_BUCKET
google_project_id = %env GOOGLE_CLOUD_PROJECT
```
### 5. Print variables defined
```                
print("\nVariables extracted:")
for var in ["GOOGLE_CLOUD_PROJECT", "WORKSPACE_BUCKET", 
            "WORKSPACE_TEMP_BUCKET", "WORKSPACE_CDR"]:
    value = os.environ.get(var)
    print(f"{var}: {value if value else 'NOT FOUND'}")
```
### 6. Set and print bucket and google_project_id variables
```
bucket = %env WORKSPACE_BUCKET
google_project_id = %env GOOGLE_CLOUD_PROJECT
!echo $bucket
!echo $google_project_id
```
### 7. Import and initialize Hail
```
import hail as hl
hl.init(gcs_requester_pays_configuration=google_project_id)
hl.default_reference(new_default_reference = "GRCh38")
```

### 8. Set paths for the VAT and the VDS

```
auxiliary_path = 'gs://vwb-aou-datasets-controlled/v9/wgs/short_read/snpindel/'
vat_path = f'{auxiliary_path}/aux/vat/vat_complete.bgz.tsv.gz'
vds_path = f'{auxiliary_path}/vds/hail.vds'

print(f"VAT path set as vat_path")
print(f"VDS path set as vds_path")
```

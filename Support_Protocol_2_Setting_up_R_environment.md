## 1.	Load required R libraries
```
library(tidyverse)
install.packages("bigrquery")
library(bigrquery)
library(jsonlite)
```

## 2.	Set GOOGLE_CLOUD_PROJECT and WORKSPACE_CDR variables
```
GOOGLE_CLOUD_PROJECT <- system("wb workspace describe --format=json | jq -r '.googleProjectId'", intern = TRUE)

resources_json <- system("wb resource list --format=json", intern = TRUE)
resources <- fromJSON(resources_json)

# Filter only BigQuery datasets
bq_dataset <- resources[resources$resourceType %in% c("BQ_DATASET","BIGQUERY_DATASET"), ]

# Keep ONLY main CDR dataset using regex
cdr_dataset <- bq_dataset[grepl("^C[0-9]{4}Q[0-9]+R[0-9]+$", bq_dataset$datasetId), ]

# Construct WORKSPACE_CDR
WORKSPACE_CDR <- paste0(cdr_dataset$projectId, ".", cdr_dataset$datasetId)
```
3.	Set workspace bucket variables
```
workspace_bucket <- resources[resources$resourceType == "GCS_BUCKET" &
                              grepl("^workspace-bucket", resources$id), ]  # ^ means starts with
WORKSPACE_BUCKET <- paste0("gs://", workspace_bucket$bucketName)
temp_bucket <- resources[resources$resourceType == "GCS_BUCKET" &
                         grepl("temporary", resources$id), ]
WORKSPACE_TEMP_BUCKET <- paste0("gs://", temp_bucket$bucketName)
```

4. 	Print variables set
```
print("Variables assigned: ")
var <- c("GOOGLE_CLOUD_PROJECT", "WORKSPACE_BUCKET", "WORKSPACE_TEMP_BUCKET", "WORKSPACE_CDR")
for(variable in var) {
  print(paste0(variable, " : ", mget(variable)))
}
```

<img width="468" height="434" alt="image" src="https://github.com/user-attachments/assets/05de5000-0ea6-419f-8b76-fe6a8f2f8696" />

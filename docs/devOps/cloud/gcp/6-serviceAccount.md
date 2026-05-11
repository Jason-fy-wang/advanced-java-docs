---
tags:
  - GCP
  - serviceaccount
---
```shell
# list iam sa (serviceAccount) keys
gcloud iam service-accounts keys list --iam-account=engine-connector-cloudsql@projectID.gserviceaccount.com

# retrieve public key
gcloud beta iam service-account keys get-public-key PIBLIC-KEY-ID --iam-account=engine-connector-cloudsql@PROJECTOD.iam.gserviceaccount.com --output-file=engine-sql.pem

# create sa keys
gcloud iam service-accounts keys create "engine-connector-cloudsql.json" --iam-account=engine-connector-cloudsql@PROJECT.iam.gserviceaccount.com

# delete sa key
gcloud iam service-accounts keys delete KEY_ID --iam-account=engine-connector-cloudsql@PROJECT.iam.gserviceaccount.com

# get metadata for account
cloud iam service-accounts keys list --iam-account=engine-connector-cloudsql@PROJECT.iam.gserviceaccount.com --format="json"

```



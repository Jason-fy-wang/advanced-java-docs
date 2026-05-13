---
tags:
  - bucket
  - GCP
---

```shell
gcloud storage bucket create gs://BUCKET_NAME --location REGION

gcloud storage bucket delete gs://BUCKET_NAME

gsutil ls -lh gs://BUCKET_NAME

# assign role to bucket serviceAccount
PROJECT_NUMBER=$(gcloud projects describe PROJECT_ID --format='value(projectNumber)')

gcloud projects add-iam-policy-binding PROJECT_NAME --member="serviceAccount:service-${PROJECT_NUMBER}@gs-project-accounts.iam.gserviceaccount.com"

```


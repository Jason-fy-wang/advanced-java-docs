---
tags:
  - bucket
  - GCP
---

```shell
gcloud storage bucket create gs://BUCKET_NAME --location REGION

gcloud storage bucket delete gs://BUCKET_NAME

# remove file
gcloud storage rm gs://BUCKET/filename

gsutil ls -lh gs://BUCKET_NAME

# assign role to bucket serviceAccount
PROJECT_NUMBER=$(gcloud projects describe PROJECT_ID --format='value(projectNumber)')

gcloud projects add-iam-policy-binding PROJECT_NAME --member="serviceAccount:service-${PROJECT_NUMBER}@gs-project-accounts.iam.gserviceaccount.com"


# set object content-type
gcloud storage objects update gs://BUCKET_NAME/file --content-type=application/json
## set content-type when cp file
gcloud storage cp request.json gs://BUCKET/result.json --content-type=application/json
```


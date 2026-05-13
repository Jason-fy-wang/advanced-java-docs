---
tags:
  - IAM
  - GCP
---
```shell
# list SA roles
gcloud iam service-accounts list

gcloud projects get-iam-policy PROJECT_ID \ 
  --filter="bindings.memeber:serviceAccount:SA_EMAIL" 
  --flatten="bindings" \
  --format="table(bindings.role)"


# add role to SA
gcloud projects add-iam-policy-binding PROJECT_NAME --member="serviceAccount:service-${PROJECT_NUMBER}@gs-project-accounts.iam.gserviceaccount.com"

# delete roles
gcloud projects remove-iam-policy-binding PROJECT_ID --member="serviceAccount:SA_EMAIL"  --role="roles/ROLE_NAME"

```


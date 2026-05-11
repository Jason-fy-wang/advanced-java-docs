---
tags:
  - GCP
  - secrets
---
```shell
# get secrets
gcloud secrets versions access 1 --secret="app-token" 
gcloud secrets versions access latest --secret="secret-token"
```


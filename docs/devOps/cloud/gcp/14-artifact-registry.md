---
tags:
  - GCP
  - artifact
  - registry
---

```shell
gcloud artifacts repositories create my-repository \
	--repository-format=docker \
	--localtion=$LOCATION \
	--description="docker registry"
	
gcloud auth configure-docker $LOCATION-docker.pkg.dev

cat >Dockerfile << EOF
FROM node:20-slim
WORKDIR /usr/app

COPY package*  ./
RUN nmp install --only=produection
COPY . ./
CMD ["npm", "start"]
EOF

# build
gcloud builds submit --tag $LOCATION-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/my-repository/helloword

# delete 
gcloud artifacts docker image delete $LOCATION-docker.pkg.dev/$GOOGLE_CLOUD_PROJECT/my-repository/helloword

```

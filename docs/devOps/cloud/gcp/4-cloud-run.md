---
tags:
  - GCP
  - cloud-run
---
some basic cloud-run functions operation:
```shell
REGION=us-west1

gcloud config set compute/region $REGION

```

Create a cloud-run function to process the Pub/Sub event
```shell

cat > index.js << EOF
const functions=require('@google-cloud/functions-framework');
functions.cloudEvent('helloPubSub', cloudEvent => {

	const base64name = cloudEvent.data.message.data;
	
	const name = base64name ? Buffer.from(base64name, 'base64').toString() : "World";
	
	console.log(`hello, ${name}!`);
})
EOF

cat > package.json << EOF
{
	"name": "gcf_hello_world",
	"version": "1.0.0",
	"main": "index.js",
	"scripts": {
		"start": "node index.js",
		"test": "echo \"Error: no test\" && exit 1"
	},
	"dependencies": {
		"@google-cloud/functions-framework": "^3.0.0"
	}
}
EOF

gcloud functions deploy nodejs-pubsub-function \
	--gen2 \
	--runtime=nodejs22 \
	--source=. \
	--entry-point=helloPubSub \
	--trigger-topic=cf-demo \
	--stage-bucket PROJECT_ID \
	--service-account cloudfunctions@PROJECT.iam.gserviceaccount.com \
	--allow-unauthenicated

gcloud functions describe nodejs-pubsub-funtion --region $REGION

# trigger pubSub event
gcloud pubsub topics publish cf-demo --message "Cloud function gen2"

gcloud functions logs read nodejs-pubsub-function --region $REGION

```

Create a cloud-run function to process http request
```shell
cat > index.js << EOF
const functions = require('@google-cloud/functions-framework')

functions.http('helloHttp', (req,res) => {
	res.set("Content-type", "text/plain");
	res.send(\`Hello \${req.query.name || req.body.name || 'World'}\`);
})
EOF

cat > package.json << EOF
{
	"dependencies": {
		'@google-cloud/functions-framework': "^3.0.0"
	}
}
EOF

# list runtimes
gcloud functions runtimes list

# deploy cloud-run  
gcloud functions deploy gcfunction --gen2 --source=.   \
	--entry-point=helloHttp  \
	--runtime=nodejs22 \
	--allow-unauthenticated \
	--region=$REGION \
	--trigger-http \
	--max-instance=5
	

# test
cutl -X POST --url https://us-west4-xxxxxx.cloudfunctions.net/gcfunction  -H "Authorization: bearer $(gcloud auth print-identity-token)" -H "Content-type: application/json" -d '{"name" : "Developer"}' 


```


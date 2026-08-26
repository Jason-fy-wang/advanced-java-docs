---
tags:
  - docker
  - dockerfile
  - instruction
---
#### 1. COPY 

```dockerfile
# copy dist from builder
## this copy will keep the origin structer
COPY --from=frondbuild /app/dist ./front/

## this copy will copy all the files under dist to front
COPY --from=frondbuild /app/dist/* ./front/
```

```shell
# the dist folder stuctor as below:

index.html
assert/
	style.css
	dist.js
favicon.svg 

```

```shell
# after below copy, the output structor is same
COPY --from=frondbuild /app/dist ./front/
output:
front/
	index.html
	assert/
		style.css
		dist.js
	favicon.svg 

# below copy will change the file structor
COPY --from=frondbuild /app/dist/* ./front/

output:
front/
	index.html
	style.css
	dist.js
	favicon.svg 


```



# NestJS Cloud Run With Authentication

## pre reqs
- gcloud cli installed --> [here](https://cloud.google.com/sdk)
- NestJs cli installed globally --> [here](https://docs.nestjs.com/) 
 

## Generate nest js project 
1. generate nestjs project ( we will name ours cloud-run ) 
    - `nest new cloud-run`
    
1. update nestjs to respect port environment variable

src/main.ts
```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { Logger } from '@nestjs/common';

const logger = new Logger('APP');
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const port = process.env.PORT || 3000;
  await app.listen(port, () => {
    logger.log(`⛱ server running on port ${port}`)
  });
}
bootstrap();

```

invoke `npm run start:dev` to see the app start up 
invoke `npm run test:cov` to see the unit test run and that sweet 100% code coverage

```
-------------------|----------|----------|----------|----------|-------------------|
File               |  % Stmts | % Branch |  % Funcs |  % Lines | Uncovered Line #s |
-------------------|----------|----------|----------|----------|-------------------|
All files          |      100 |      100 |      100 |      100 |                   |
 app.controller.ts |      100 |      100 |      100 |      100 |                   |
 app.service.ts    |      100 |      100 |      100 |      100 |                   |
-------------------|----------|----------|----------|----------|-------------------|
Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        3.451s
Ran all test suites.


```

invoke `npm run test:e2e` to see the e2e suite run


Now let us do some configuration to get this to run on cloud build

We will create the following files
1. Dockerfile
```dockerfile
# use node 10 image
FROM node:10 as base

# set our working directory
WORKDIR /app

FROM base as development

# copy over dependecy manifests to install
COPY package*.json ./

# install production dependencies and copy them to a temp folder, we will use this for the final build step
RUN npm install --production
RUN cp -R node_modules /tmp/node_modules

# Install all dependcies
RUN npm install

# Copy source code
COPY . ./

# lint, unit test, e2e test and build our app
FROM development as builder
RUN npm run lint
RUN npm run test:cov
RUN npm run test:e2e
RUN npm run build

# release includes bare miniumum to run the app
FROM base as release
COPY --from=builder /tmp/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./

# Run the web service on container boot
CMD ["npm", "run", "start:prod"]

```
1. .dockerignore
```docker
coverage
node_modules/
dist
.idea
.vscode

```
1. cloudbuild.yaml


```yaml
steps:
  # build the container image
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/$_SERVICE_NAME', '.']
  # push the container image to Container Registry
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/$_SERVICE_NAME']
  # Deploy container image to Cloud Run
  - name: 'gcr.io/cloud-builders/gcloud'
    args:
      - 'run'
      - 'deploy'
      - 'cloud-run'
      - '--image'
      - 'gcr.io/$PROJECT_ID/$_SERVICE_NAME'
      - '--region'
      - '$_REGION'
      - '--platform'
      - 'managed'
      - '--no-allow-unauthenticated'
images:
  - 'gcr.io/$PROJECT_ID/$_SERVICE_NAME'
substitutions:
  _SERVICE_NAME: "cloud-run"
  _REGION: "us-central1"

```

## Service account adjustment
1. Log into gcp and go to cloud build settings and give the cloud build service account Cloud Run Admin and Service Account User

## trigger build. 
from you command line type `gcloud builds submit` and watch the build and auto deployment

## test secure service
```shell script
curl -H \
"Authorization: Bearer $(gcloud auth print-identity-token)" \
$(gcloud run services describe SERVICENAME --format 'value(status.url)')
```

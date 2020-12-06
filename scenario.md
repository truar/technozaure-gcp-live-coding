# Deploying a complete modern WebApplication on GCP

* Before starting, execute 
    * `gcloud version` to make sure there is no update
    * start Docker Desktop
* Replace in this file tzp-20201206-297810 with tzp-20201207

* Start with the application backend locally
    * Initialize Spring Web Project
        * group: com.zenika.technozaure.gcplivecoding
        * artifact: backend
        * Description: Backend application deployed on GCP
    * then execute CMD + SHIFT + A and execute 'Add maven project'
    * Start the server once to make sure it works

* Create the first controller endpoint
```java
// src/main/java/com/zenika/technozaure/gcplivecoding/backend
package com.zenika.technozaure.gcplivecoding.backend;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloWorldController {

    @GetMapping("/")
    public String helloWorld() {
        return "Hello World";
    }
}
```

* Execute the server using maven to make sure it works
```shell script
./mvnw clean install
java -jar target/backend-0.0.1-SNAPSHOT.jar 
```

* Add properties to be compliant with Cloud Run
```
# src/main/resources/application.properties
server.port=${PORT:8080}
server.servlet.context-path=/api
```
> Cloud Run recommends you to let the environment set the port via the PORT env.
> Prefix your endpoints with /api, very convenient when communicating to frontend application with proxy and no rewrite url to do

* Dockerize your backend with a multi-stage Dockerfile, very convenient to build all at once
```dockerfile
FROM maven:3.6.3-openjdk-11-slim as builder

WORKDIR /app
COPY pom.xml .
# Use this optimization to cache the local dependencies. Works as long as the POM doesn't change
RUN mvn dependency:go-offline

COPY src /app/src/
RUN mvn package

# Use AdoptOpenJDK for base image.
FROM adoptopenjdk/openjdk11:alpine-jre

# Copy the jar to the production image from the builder stage.
COPY --from=builder /app/target/*.jar /app.jar

# Run the web service on container startup.
CMD ["java", "-jar", "/app.jar"]
```

Test if your dockerfile works as expected
```shell script
docker build -t gcr.io/tzp-20201206-297810/backend:latest .
docker run -d -p 8080:8080 gcr.io/tzp-20201206-297810/backend:latest
```

* Deploy the application onto Cloud Run

* Using the CLI and connect to your GCP account
```shell script
gcloud auth login
```
> Take the Zenika account

* Create a new project
    * Explain what is a project: contains all resources of your GCP application. 
    * Show with the Cloud UI
```shell script
gcloud projects create tzp-20201206-297810 --set-as-default
```

* Now, deploy using gcloud CLI
1. Enable cloud API
```shell script
gcloud services enable run.googleapis.com
```

2. Create a service account to ensure security
````shell script
gcloud iam service-accounts create backend-account \
    --description="Service account that executes the backend" \
    --display-name="Backend service account"
````

3. Create the deployment file for Cloud Run, just like kubernetes
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: application-backend
  labels:
    cloud.googleapis.com/location: europe-west1
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/maxScale: '3'
    spec:
      serviceAccountName: backend-account
      containerConcurrency: 80
      timeoutSeconds: 300
      containers:
      - image: gcr.io/tzp-20201206-297810/backend:latest
        resources:
          limits:
            cpu: 1000m
            memory: 256Mi
  traffic:
    - percent: 100
      latestRevision: true
```

4. Push the previous image to Container registry
```shell script
gcloud auth configure-docker
gcloud services enable containerregistry.googleapis.com
docker push gcr.io/tzp-20201206-297810/backend:latest
```

5. Deploy to cloudRun and allow public access
```shell script
gcloud beta run services replace cloudrun-backend.yaml \
  --platform=managed \
  --region=europe-west1

gcloud run services add-iam-policy-binding application-backend \
  --platform=managed \
  --region=europe-west1 \
  --member="allUsers" \
  --role="roles/run.invoker"
```

* Create the frontend of the application

* Generate the application using the `vue` CLI
    * Just view the final result in the browser
    * !!! Don't forget to stop the previous running instance on 8080
```shell script
vue create frontend
cd frontend
yarn serve
```

* Stop the server CTRL+C

* As you could see, the default frontend port is the same as the backend ones. Let's change it by creating a `vue.config.js` file:
```js
module.exports = {
    devServer: {
        port: 8088,
        proxy: {
            '/api': {
                target: 'http://localhost:8080',
                ws: true,
                changeOrigin: true
            }
        }
    }
}
```
> With this configuration, we avoid configuring CORS (even if now it won't seem to be a problem) by using a reverse proxy. It will work the same way on Firebase

* Modify the `frontend/src/App.vue` component to fetch the message from the server
```vue
<HelloWorld :msg="message"/>
  
,
  data: () => ({
    message: 'Loading...'
  }),
  async created() {
    const response = await fetch('/api/')
    this.message = await response.text()
  }
```

* Test the application fully locally
```shell script
java -jar backend/target/backend-0.0.1-SNAPSHOT.jar
yarn serve
```
* See on the browser the result


* Deploy the application on Firebase
* Add the project tzp-20201206-297810 to Firebase using the UI
    * Add project by looking in Zenika projects if needed, but it should be proposed automatically
    * Quickly browse Firebase to see the features
        * Authentication
        * Hosting
        * Storage
    * Everything's is based on GCP at the end
    
* Firebase has also a CLI to use it locally
* Inside `frontend`
* Create the firebase.json file, description file to deploy on firebase your code
    * We are using only the Hosting features, so our file will be simple enough
    
* `firebase init`
    * Select `Hosting`
    * Set the project ID: tzp-20201206-297810
    * dist as public folder
    * index.html a rewrite rule

* Show the generated file `firebase.json` et `.firebaserc`

* Add the rewrite rule for Cloud run. Be careful of the matching rules priorities. `firebase.json`
```json
{
    "source": "/api/**",
    "run": {
        "serviceId": "application-backend",
        "region": "europe-west1"
    }
}
```


* Given time: now let's add a CI/CD pipeline

* Enable the Cloud Build API from the GCP console or using :
```shell script
gcloud services enable cloudbuild.googleapis.com
```

* Add the rights to deploy Cloud Run and change user to CloudBuild user

* Synchronize your Github project with Cloud Source Repositories
    * Might be in error for quite a while, even if I don't know why... Give it a lot of tries... If it doesn't work, keep going with Cloud-build configuration file

* Create the cloud build trigger.
    * name: tzp-livecoding-trigger
    * description: don't care
    * branch: .* (select all)
    
* Create and import a Firebase image using some command line
```shell script
cd ../cloud-builders-community/firebase
gcloud builds submit .
```
   
* Create the `cloudbuild.yaml` file
```yaml
steps:
  - id: 'dockerize-project'
    name: gcr.io/cloud-builders/docker
    dir: backend
    args: ['build',
           '-t', 'gcr.io/$PROJECT_ID/backend:$SHORT_SHA',
           '-t', 'gcr.io/$PROJECT_ID/backend:latest',
           '.']

  - id: 'push-to-cloud-registry'
    name: gcr.io/cloud-builders/docker
    args: ['push', 'gcr.io/$PROJECT_ID/backend:$SHORT_SHA']

  - id: 'deploy-cloud-run'
    name: gcr.io/cloud-builders/gcloud
    dir: backend
    entrypoint: bash
    args:
      - '-c'
      - |
        apt-get update
        apt-get install -qq -y gettext
        export PROJECT_ID=$PROJECT_ID
        export IMAGE_VERSION=$SHORT_SHA
        envsubst < cloudrun-backend.yaml > cloudrun-backend_with_env.yaml
        gcloud beta run services replace cloudrun-backend_with_env.yaml \
          --platform=managed --region=europe-west1
        gcloud run services add-iam-policy-binding application-backend \
          --platform=managed --region=europe-west1 \
          --member="allUsers" --role="roles/run.invoker"

  - id: 'install-yarn'
    waitFor: ['-']
    name: node
    entrypoint: yarn
    dir: frontend
    args: ['install', '--silent']

  - id: 'build-front'
    waitFor: [ 'install-yarn' ]
    name: node
    entrypoint: yarn
    dir: frontend
    args: [ 'build' ]

  - id: 'deploy-firebase'
    waitFor: [ 'build-front' ]
    name: gcr.io/$PROJECT_ID/firebase
    dir: frontend
    args: [ 'deploy', '--project=$PROJECT_ID', '--only', 'hosting' ]

images:
  - 'gcr.io/$PROJECT_ID/backend:$SHORT_SHA'
  - 'gcr.io/$PROJECT_ID/backend:latest'
```

* Edit the `backend?cloudrun-backend.yaml` to take some variables into account:
```yaml
- image: gcr.io/${PROJECT_ID}/backend:${IMAGE_VERSION}
```

* Change some files to see the deployment working:
```java
// backend/src/main/java/.../HelloWorldController
    public String helloWorld() {
        return "Hello World. Automatic Deployment !!";
    }
``` 

* In `frontend/src/HelloWorld.vue`:
```vue
<h1>From firebase: {{ msg }}</h1>
```

* Commit and push your changes

* If it didn't work as expected, use `gcloud` CLI to easily try your modification:
```shell script
gcloud builds submit . --substitutions SHORT_SHA=local
```

* If build is success, then check the firebase and Cloud run interface
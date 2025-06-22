## FGC Project - Jenkins CICD Pipeline with Docker Deployment

### Overview
This project demonstrates a CI/CD pipeline using Jenkins to build, push, and deploy an Angular application with Docker. The pipeline is parameterized to support multiple systems, versions, clients, and environments.

### Prerequisites
* Jenkins server with Docker installed
* Docker Hub account credentials
* GitHub repository access
* Node.js and Angular CLI (for local development)

### Jenkins Pipeline Features
#### Parameters
* **SYSTEM:** Target system (system-f1, system-f2)

* **VERSION:** Application version (version-v1, version-v2)

* **CLIENT:** Client identifier (client-C1, client-C2)

* **ENVIRONMENT:** Deployment environment (env-TEST, env-UAT, env-PROD)

### Pipeline Stages
1. Checkout Code: Clones the GitHub branch based on parameters
2. Build Docker Image: Builds a Docker image from the Angular app
3. Push to DockerHub: Pushes the image to Docker Hub registry
4. Deploy Locally: Stops any running container and deploys the new version

### Docker Configuration
* Multi-stage build:

  * Build stage: Node.js environment for Angular build

  * Runtime stage: Lightweight nginx server to serve the Angular app

### Dockerfile 
```
FROM node:20-alpine AS build
WORKDIR /app
COPY angular-docker-app/package*.json ./
RUN npm install
COPY angular-docker-app/ ./
RUN npm run build -- --configuration=production

FROM nginx:alpine
COPY --from=build /app/dist/angular-app/browser /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
### containerized Lightweight nginx with Angular app
![image](https://github.com/user-attachments/assets/d5d85d81-7cff-4572-af93-9828bd23a6f1)

### Jenkins Pipeline 
![image](https://github.com/user-attachments/assets/627ffb46-44e2-4742-81f6-01a072443707)

### Jenkins Pipeline Code
```
pipeline {
    agent any

    parameters {
        string(name: 'SYSTEM', defaultValue: 'system-f1', description: 'Enter the system (e.g., system-f1, system-f2, system-f3)')
        string(name: 'VERSION', defaultValue: 'version-v1', description: 'Enter the version (e.g., version-v1, version-v2)')
        string(name: 'CLIENT', defaultValue: 'client-C1', description: 'Enter the client (e.g., client-C1, client-C2)')
        string(name: 'ENVIRONMENT', defaultValue: 'env-TEST', description: 'Enter the environment (e.g., env-TEST, env-PROD)')
    }

    environment {
        DOCKER_IMAGE = 'sandiprathore/fgc_project'
        IMAGE_TAG = "${BUILD_NUMBER}"
        BRANCH_NAME = "${params.SYSTEM}/${params.VERSION}/${params.CLIENT}/${params.ENVIRONMENT}"
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo "Checking out code from GitHub branch: ${BRANCH_NAME}..."
                git branch: "${BRANCH_NAME}", url: 'https://github.com/sandiprathore29/FGC_Project.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image..."
                script {
                    docker.build("${DOCKER_IMAGE}:${IMAGE_TAG}", "-f angular-docker-app/Dockerfile .")
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh """
                        echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                        docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy Locally') {
            steps {
                echo "Deploying locally..."
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh """
                        echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                        docker pull ${DOCKER_IMAGE}:${IMAGE_TAG}
                        docker stop fgc_container || true
                        docker rm fgc_container || true
                        docker run -d --name fgc_container -p 8085:80 ${DOCKER_IMAGE}:${IMAGE_TAG}
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed. Docker Image: ${DOCKER_IMAGE}:${IMAGE_TAG}"
        }
        failure {
            echo "Pipeline failed. Please check the logs above."
        }
    }
}
```

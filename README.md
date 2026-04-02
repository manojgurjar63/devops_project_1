# CI/CD Pipeline with Auto Rollback
A Jenkins-based CI/CD pipeline that automatically rolls back a deployment if the application
fails its health check after deploying. Built on localhost with Jenkins, Docker, and Docker Hub.

# Architecture 
GitHub Repo
    │
    │  git push (manual trigger currently)
    ▼
Jenkins (port 8080)
    │
    ├── Stage 1: Copy files     → rsync project files to Jenkins workspace
    ├── Stage 2: Build          → docker build, tag as v2
    ├── Stage 3: Tag & Push     → push to Docker Hub (manojgurjar22/manoj)
    ├── Stage 4: Deploy         → stop old container, start new on port 8090
    ├── Stage 5: Health Check   → curl localhost:8090
    │
    ├── ✅ Pass → v2 is live at localhost:8090
    └── ❌ Fail → Auto rollback: stop v2, restart v1

# Infrastructure running Jenkins and Docker side by side

# Tech Stack
| Tool         | Purpose    |
|--------------|------------|
| Jenkins      | CI/CD pipeline runner  |
| Docker       | App containerization   |  
| nginx:alpine | Web server inside container |
| Docker Hub   | Image registry (manojgurjar22/manoj)   |
| GitHb        | Source code repository |

# Project Structure
/home/manoj/devops_projects/
├── Dockerfile
├── index.html
└── Jenkinsfile

# Setup Guide

1. Add jenkins user to docker group
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
2. Add Docker Hub credentials to Jenkins
Go to Manage Jenkins → Credentials → Global → Add Credentials:

Kind: Username with password
Username: your Docker Hub username
Password: your Docker Hub password
ID: dockerhub-creds

3. Application files
index.html:
This is Manoj Gurjar
Dockerfile:
dockerfileFROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80

# Jenkinsfile
pipeline {
    agent any
    environment {
        IMAGE_TAG = 'v2'
        STABLE_TAG = 'v1'
    }
    stages {
        stage('Copy files to workspace') {
            steps {
                sh 'rsync -avz /home/manoj/devops_projects/ /var/lib/jenkins/workspace/my_project_1/'
            }
        }
        stage('Build Docker image') {
            steps {
                sh 'docker build -t manoj:${IMAGE_TAG} /var/lib/jenkins/workspace/my_project_1/'
            }
        }
        stage('Tag and push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh 'docker tag manoj:${IMAGE_TAG} manojgurjar22/manoj:${IMAGE_TAG}'
                    sh 'docker push manojgurjar22/manoj:${IMAGE_TAG}'
                }
            }
        }
        stage('Deploy container') {
            steps {
                sh '''
                    docker stop manoj-app || true
                    docker rm manoj-app || true
                    docker run -d --name manoj-app -p 8090:80 manojgurjar22/manoj:v2
                '''
            }
        }
        stage('Health check') {
            steps {
                script {
                    sleep(5)
                    def curlExit = sh(
                        script: 'curl -s -o /dev/null http://localhost:8090 -m 10',
                        returnStatus: true
                    )
                    echo "Curl exit code: ${curlExit}"
                    if (curlExit != 0) {
                        echo "Health check failed — rolling back to v1"
                        sh '''
                            docker stop manoj-app || true
                            docker rm manoj-app || true
                            docker run -d --name manoj-app -p 8090:80 manojgurjar22/manoj:v1
                        '''
                        error("Deployment failed. Rolled back to v1.")
                    } else {
                        echo "v2 is live at http://localhost:8090"
                    }
                }
            }
        }
    }
}

# How Auto Rollback Works

1. Jenkins builds the new Docker image and tags it v2
2. It stops the running container and starts v2 on port 8090
3. After a 5-second wait, it runs curl localhost:8090
4. If curl exit code is not 0 (connection failed), Jenkins stops v2 and restarts v1
5. The pipeline is marked as FAILED with a rollback message in the logs
6. The app stays live on port 8090 serving v1 — zero downtime

# Testing the Rollback
To intentionally trigger a rollback, change port 8090 to 8091
immediately after starting in the Deploy stage:
stage('Deploy container') {
    steps {
        sh '''
            docker stop manoj-app || true
            docker rm manoj-app || true
            docker run -d --name manoj-app -p 8091:80 manojgurjar22/manoj:v2
        '''
    }
}

Jenkins will deploy v2, fail the health check, and automatically restore v1.
Verify rollback worked:
docker container ls
should show manojgurjar22/manoj:v1 running on port 8090

# Rollback Test Result

✅ Build v2         — success
✅ Push to Hub      — success
✅ Deploy v2        — success
❌ Health check     — failed (container crashed)
✅ Rollback to v1   — automatic, zero downtime

# Key Learnings

How CI/CD pipelines are structured (build → push → deploy → verify)
Why health checks are critical in automated deployments
How Docker image tagging enables rollback strategies (v1, v2)
Jenkins pipeline syntax (Jenkinsfile, withCredentials, returnStatus)
Difference between single and double quotes in Groovy shell steps
Why returnStatus: true is needed to prevent curl failures crashing the pipeline

# Next Steps

Move to AWS EC2 for a public IP and proper GitHub webhook trigger
Add GitHub webhook via ngrok for automatic pipeline triggering on push
Add post { failure { } } block for more robust rollback handlin

Author
Manoj Gurjar — Built as part of a DevOps + AI learning project series.

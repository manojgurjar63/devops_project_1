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
bashsudo usermod -aG docker jenkins
sudo systemctl restart jenkins
2. Add Docker Hub credentials to Jenkins
Go to Manage Jenkins → Credentials → Global → Add Credentials:

Kind: Username with password
Username: your Docker Hub username
Password: your Docker Hub password
ID: dockerhub-creds

3. Application files
index.html:
htmlThis is Manoj Gurjar
Dockerfile:
dockerfileFROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80









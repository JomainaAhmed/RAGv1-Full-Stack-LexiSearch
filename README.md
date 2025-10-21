# 🚀 LexiSearch

**LexiSearch** is a full-stack multi-document search web app that leverages **Retrieval Augmented Generation (RAG)** to enhance generative AI results. It combines **React.js**, **LangChain**, **AWS Bedrock**, **ChromaDB**, and **FastAPI**, wrapped in Docker and deployed on AWS infrastructure.

---

## 🛠️ Tech Stack

- **Frontend:** React.js
- **Backend:** Python, FastAPI, Langchain, AWS Bedrock, ChromaDB
- **Containerization:** Docker
- **Deployment:** AWS ECR, ECS, EC2, Nginx
- **Security:** SSL/TLS via Let's Encrypt

---

## 🌐 Deployment Overview

This guide shows how to deploy Docu-Dive using Docker and AWS services.

### ✅ Prerequisites

- AWS CLI installed and configured with IAM permissions
- Docker installed
- An AWS account (with access to ECR, ECS, EC2)
- Domain configured via Route 53

---

## 📦 Step-by-Step Deployment Guide

### 1. Prepare Python Files

Organize backend files:

/backend/
├── main.py
├── query.py
├── .env
└── requirements.txt


### 2. Create Dockerfiles

**Frontend (`frontend/Dockerfile`):**
```Dockerfile
FROM node:14
WORKDIR /app
COPY . .
RUN npm install
CMD ["npm", "start"]

Backend (backend/Dockerfile):

FROM python:3.9
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

3. Build Docker Images
cd frontend
docker build --platform linux/amd64 -t frontend-image .

cd ../backend
docker build --platform linux/amd64 -t backend-image .

4. Push Docker Images to AWS ECR
# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account_id>.dkr.ecr.us-east-1.amazonaws.com

# Tag & Push Frontend
docker tag frontend-image:latest <account_id>.dkr.ecr.us-east-1.amazonaws.com/frontend-repo:latest
docker push <account_id>.dkr.ecr.us-east-1.amazonaws.com/frontend-repo:latest

# Tag & Push Backend
docker tag backend-image:latest <account_id>.dkr.ecr.us-east-1.amazonaws.com/backend-repo:latest
docker push <account_id>.dkr.ecr.us-east-1.amazonaws.com/backend-repo:latest

5. Set Up ECS Cluster

Create an ECS cluster via AWS Console or CLI.

6. Register ECS Task Definitions

Example (JSON file required):

aws ecs register-task-definition --cli-input-json file://frontend-task-definition.json
aws ecs register-task-definition --cli-input-json file://backend-task-definition.json

7. Create ECS Services
aws ecs create-service \
  --cluster docu-dive-cluster \
  --service-name frontend-service \
  --task-definition frontend-task \
  --desired-count 1 \
  --launch-type FARGATE

aws ecs create-service \
  --cluster docu-dive-cluster \
  --service-name backend-service \
  --task-definition backend-task \
  --desired-count 1 \
  --launch-type FARGATE

8. Set Up EC2 for Nginx

Launch EC2 instance

Install Docker + Nginx

Configure security groups (ports 80, 443, 3000, 8000)

9. Nginx Configuration
server {
    listen 80;
    server_name docu-dive.com www.docu-dive.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name docu-dive.com www.docu-dive.com;

    ssl_certificate /etc/letsencrypt/live/docu-dive.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/docu-dive.com/privkey.pem;

    location / {
        proxy_pass http://localhost:3000;  # Frontend
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api {
        proxy_pass http://localhost:8000;  # Backend
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

10. Run Docker Containers on EC2
# Pull from ECR and run frontend
docker run -d -p 3000:3000 --name frontend-container <account_id>.dkr.ecr.us-east-1.amazonaws.com/frontend-repo:latest

# Pull from ECR and run backend
docker run -d -p 8000:8000 --name backend-container <account_id>.dkr.ecr.us-east-1.amazonaws.com/backend-repo:latest


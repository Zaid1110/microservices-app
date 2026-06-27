<div align="center">

# ☁️ CloudOps Microservices Platform

### Production-grade CI/CD pipeline for an 11-microservice e-commerce application  
**Jenkins · Docker · AWS ECR · AWS EKS · ArgoCD · SonarQube · Trivy**

[![Jenkins](https://img.shields.io/badge/Jenkins-Multibranch_Pipeline-D24939?logo=jenkins&logoColor=white)](https://jenkins.io)
[![AWS](https://img.shields.io/badge/AWS-EKS_%7C_ECR-FF9900?logo=amazonaws&logoColor=white)](https://aws.amazon.com)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-Orchestration-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![Docker](https://img.shields.io/badge/Docker-Containerization-2496ED?logo=docker&logoColor=white)](https://docker.com)
[![ArgoCD](https://img.shields.io/badge/ArgoCD-GitOps_CD-EF7B4D?logo=argo&logoColor=white)](https://argoproj.github.io)
[![SonarQube](https://img.shields.io/badge/SonarQube-Code_Quality-4E9BCD?logo=sonarqube&logoColor=white)](https://sonarqube.org)

</div>

---

## 📌 Project Overview

This project demonstrates a **real-world DevSecOps pipeline** for a polyglot microservices application — modelled after Google's [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo). Each of the 11 microservices is independently built, scanned, and deployed through its own dedicated Jenkins pipeline, showcasing true microservice autonomy in a CI/CD context.

**Live deployment** runs on **AWS EKS (N. Virginia — us-east-1)** exposed via an AWS Application Load Balancer.

---

## 🏗️ Architecture

```
Developer → GitHub Push → Jenkins Webhook Trigger
                              │
                    ┌─────────▼──────────┐
                    │  Jenkins Pipeline   │
                    │  (per microservice) │
                    └─────────┬──────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        SonarQube        Docker Build      Trivy Scan
        (Code Quality)   (Image)           (CVE Check)
                              │
                              ▼
                        AWS ECR (Push)
                              │
                    ┌─────────▼──────────┐
                    │  Update GitOps      │
                    │  (image tag in      │
                    │   K8s manifests)    │
                    └─────────┬──────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │  ArgoCD (GitOps CD) │
                    │  Detects manifest   │
                    │  change → syncs     │
                    └─────────┬──────────┘
                              │
                              ▼
                    ┌─────────────────────┐
                    │  AWS EKS Cluster    │
                    │     us-east-1       │
                    └─────────────────────┘
```

---

## 🧩 Microservices

| Service | Language | Port | Responsibility |
|---|---|---|---|
| `frontend` | Go | 8080 | Web UI — renders the entire storefront |
| `cartservice` | C# / .NET | 7070 | Cart management using Redis |
| `checkoutservice` | Go | 5050 | Order orchestration — ties all services together |
| `productcatalogservice` | Go | 3550 | Product listing from JSON catalog |
| `paymentservice` | Node.js | 50051 | Processes payments (mock) |
| `shippingservice` | Go | 50051 | Shipping quote and order tracking |
| `emailservice` | Python | 8080 | Sends order confirmation emails |
| `currencyservice` | Node.js | 7000 | Currency conversion (live rates) |
| `recommendationservice` | Python | 8080 | Product recommendations via gRPC |
| `adservice` | Java | 9555 | Contextual ads served to frontend |
| `loadgenerator` | Python (Locust) | — | Synthetic traffic generation for testing |

> Each service communicates via **gRPC** using Protocol Buffers (`.proto` files). REST is only used for the frontend ↔ user interaction.

---

## ⚙️ CI/CD Pipeline — Per Service

Every microservice has its **own Jenkinsfile** and its **own multibranch pipeline** in Jenkins. A push to any service branch triggers only that service's pipeline — other services remain unaffected.

### Pipeline Stages

```groovy
Stage 1: Checkout          // Clone repo, checkout service branch
Stage 2: SonarQube Scan    // Static code analysis + quality gate
Stage 3: Docker Build      // Build image, tag with build number
Stage 4: Trivy Scan        // Container vulnerability scan (CVE)
Stage 5: Push to ECR       // Push image to AWS ECR (us-east-1)
Stage 6: Update Manifests  // Update image tag in K8s YAML (GitOps repo)
```

### Why 12 Pipelines?

| Pipeline | Branch | Purpose |
|---|---|---|
| `infra-steps` | infra-steps | Provisions EKS cluster (eksctl / Terraform) |
| `main` | main | Applies final K8s deployment YAMLs to EKS |
| 10 × service pipelines | per service | Build → Scan → Push → Update |

---

## 🛡️ DevSecOps — Security at Every Stage

| Tool | Stage | What it does |
|---|---|---|
| **SonarQube** | Pre-build | Detects code bugs, code smells, security hotspots |
| **Trivy** | Post-build | Scans Docker image for OS and library CVEs |
| **AWS ECR** | Registry | Private, IAM-controlled image registry |
| **IRSA** | Runtime | IAM Roles for Service Accounts — no hardcoded AWS keys in pods |

Pipeline fails automatically if SonarQube quality gate is not passed — broken code never reaches ECR.

---

## ☸️ Kubernetes Setup (EKS — us-east-1)

- **Cluster**: AWS EKS managed node group
- **Namespace**: `default` (per service deployments)
- **Exposure**: AWS Application Load Balancer (ALB Ingress Controller)
- **GitOps CD**: ArgoCD watches the manifests repo — any image tag change triggers an automatic sync

### Security Group Inbound Rules

| Port(s) | Purpose |
|---|---|
| 22 | SSH access (EC2 / nodes) |
| 80, 443 | HTTP / HTTPS (frontend ALB) |
| 6443 | Kubernetes API server |
| 3000–10000 | Jenkins (8080), SonarQube (9000), ArgoCD (8080/8443), NodePort services |
| 30000–32767 | Kubernetes NodePort range |
| 465, 25 | SMTP (email service) |
| 27017 | MongoDB (data layer) |

---

## 🗂️ Repository Structure

```
cloudops-microservices-platform/
│
├── adservice/                  # Java — contextual ad service
├── cartservice/                # C# — cart with Redis backend
├── checkoutservice/            # Go — order orchestration
├── currencyservice/            # Node.js — currency conversion
├── emailservice/               # Python — email confirmation
├── frontend/                   # Go — web UI (Thymeleaf templates)
├── loadgenerator/              # Python Locust — load testing
├── paymentservice/             # Node.js — payment processing
├── productcatalogservice/      # Go — product listing
├── recommendationservice/      # Python — recommendations
├── shippingservice/            # Go — shipping quotes
│
├── infra-steps/
│   ├── Jenkinsfile             # EKS cluster provisioning pipeline
│   └── deployment-service.yml # Infra K8s config
│
├── Jenkinsfile                 # Root-level pipeline (main branch)
├── deployment-service.yml      # Main K8s deployment manifest
└── README.md
```

---

## 🚀 How to Run Locally (Minikube)

```bash
# Clone the repo
git clone https://github.com/Zaid1110/cloudops-microservices-platform.git
cd cloudops-microservices-platform

# Start Minikube
minikube start --cpus=4 --memory=8192

# Apply all service manifests
kubectl apply -f deployment-service.yml

# Get frontend URL
minikube service frontend --url
```

---

## 🔧 Tech Stack

| Category | Tools |
|---|---|
| **CI** | Jenkins (Multibranch Pipeline, Webhook trigger) |
| **CD** | ArgoCD (GitOps — pull-based deployment) |
| **Containerization** | Docker, AWS ECR |
| **Orchestration** | Kubernetes, AWS EKS |
| **Code Quality** | SonarQube |
| **Security Scan** | Trivy |
| **Cloud** | AWS (EC2, EKS, ECR, ALB, IAM, VPC) — us-east-1 |
| **Languages** | Go, Python, Node.js, C#/.NET, Java |
| **Service Mesh** | gRPC + Protocol Buffers |
| **Load Testing** | Locust (Python) |

---

## 📸 Screenshots

## | Jenkins — 12 Pipelines
 <img width="1440" height="857" alt="Screenshot 2026-06-26 at 4 57 44 PM" src="https://github.com/user-attachments/assets/7448fab9-b2cb-4b61-9912-e527a805eba3" />
 
## | EKS Cluster
 <img width="1440" height="847" alt="Screenshot 2026-06-26 at 7 19 06 AM" src="https://github.com/user-attachments/assets/bb16a1f7-680a-4976-8cf9-ad53bf245bc1" />
 
## | Live Application
 <img width="1440" height="871" alt="Screenshot 2026-06-26 at 7 27 30 AM" src="https://github.com/user-attachments/assets/95ee27fc-be7c-4a55-958f-5ea7f4e368ff" />
 
## | All Pods Running
 <img width="1286" height="786" alt="Screenshot 2026-06-26 at 7 23 50 AM" src="https://github.com/user-attachments/assets/3443a751-a0e7-4cae-93ad-569332d7b89b" />
 
## | Docker Hub
<img width="1440" height="847" alt="Screenshot 2026-06-26 at 7 16 54 AM" src="https://github.com/user-attachments/assets/b6a51db7-56ee-473f-b8cb-a332d2188c00" />

## | Order Complete
<img width="1440" height="871" alt="Screenshot 2026-06-26 at 7 28 56 AM" src="https://github.com/user-attachments/assets/5bee52e2-b0db-4f1e-aafe-3c353ae6fcfb" />

---

## 👤 Author

**Zaid Aftab** — DevOps Engineer  
[LinkedIn](https://linkedin.com/in/zaidaftab) · [GitHub](https://github.com/Zaid1110)

---

<div align="center">
<sub>Built as a portfolio project demonstrating production-grade DevSecOps practices on AWS</sub>
</div>

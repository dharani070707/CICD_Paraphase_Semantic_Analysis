# Project Walkthrough: Paraphase Semantic Analysis & CI/CD Deployment

## 1. Project Overview

The **Paraphrase Semantic Analysis** project is a comprehensive Machine Learning application designed to detect paraphrases and assess semantic similarity between text inputs. It leverages a fine-tuned Sentence Transformer (MPNet) optimized for complex language, including negation traps and adversarial word swaps.

### Core Features
- **High-Accuracy Semantic Detection:** Fine-tuned on datasets like QQP, MRPC, STS-B, PAWS, and MNLI to handle conversational, formal, and adversarial sentence pairs.
- **Microservices Architecture:** 
  - **Backend:** A Python/FastAPI service hosting the model inference.
  - **Frontend:** A web interface for users to input sentences and receive similarity scores.
- **Hardware Optimization:** Includes support for Apple Silicon (MPS), CUDA, and standard CPU environments.

---

## 2. Infrastructure & Deployment (What We Did)

To make this project robust and scalable for production environments, we established a complete containerized infrastructure:

### Docker Integration
The project natively supports Docker Compose (`docker-compose.yml`), allowing you to spin up the backend and frontend simultaneously connected via an isolated app network.

### Kubernetes (K8s) Integration
We integrated a full Kubernetes deployment strategy to allow for horizontal scaling and self-healing. All manifests are located in the `k8s/` directory:

1. **Persistent Volume Claim (`backend-pvc.yaml`):** Secures 1Gi of persistent storage for the fine-tuned machine learning models, ensuring they aren't lost if a pod restarts.
2. **Backend Deployment & Service (`backend-deployment.yaml`, `backend-service.yaml`):** Deploys the inference API using the `dharaniprasads/semantic-backend:latest` image and exposes it internally via a ClusterIP on port 8000.
3. **Frontend Deployment & Service (`frontend-deployment.yaml`, `frontend-service.yaml`):** Deploys the web UI using the `dharaniprasads/semantic-frontend:latest` image and exposes it externally via a NodePort (`30000`) for easy accessibility.

---

## 3. Getting Started

### Local Setup (Development)
If you want to run the project locally for development or testing:

1. **Clone the repository:**
   ```bash
   git clone https://github.com/dharani070707/CICD_Paraphase_Semantic_Analysis.git
   cd CICD_Paraphase_Semantic_Analysis
   ```
2. **Pull Large Model Files:**
   You must have Git LFS installed to pull the model weights.
   ```bash
   git lfs install
   git lfs pull
   ```
3. **Run via Docker Compose:**
   ```bash
   docker-compose up --build
   ```

### Kubernetes Deployment (Production)
To deploy the application to a Kubernetes cluster (e.g., Minikube, EKS, GKE):

1. **Apply the manifests:**
   ```bash
   kubectl apply -f k8s/
   ```
2. **Verify Pods & Services:**
   ```bash
   kubectl get all -l 'app in (semantic-backend, semantic-frontend)'
   ```
3. **Access the Application:**
   The frontend will be accessible on your Node's IP address at port `30000`. If using Minikube, run:
   ```bash
   minikube service semantic-frontend
   ```

---

## 4. CI/CD & Automation

The repository includes automation structures to maintain code quality and streamline updates:
- **Jenkinsfile:** Contains declarative pipeline stages for building, testing, and deploying the application.
- **Ansible:** Playbooks exist for configuration management of deployment servers.
- **Tests Suite (`tests/`):** Automated tests ensure that the model correctly identifies edge cases like negation traps before any new deployment.

## 5. Future Improvement Plan
As outlined in `IMPROVEMENT_PLAN.md`, the next phase involves migrating to a Multi-Task Training approach utilizing Knowledge Distillation to achieve >85% universal accuracy across highly diverse domains.

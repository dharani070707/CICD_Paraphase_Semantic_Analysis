# Project Walkthrough: Paraphase Semantic Analysis & CI/CD Deployment

## 🚀 Quick Start (Cross-Platform Deployment)
To easily run this project locally on either **macOS** or **Ubuntu**, simply run the automated deployment script:

```bash
./startup.sh
```

**What this script does automatically:**
1. Starts Minikube and enables the Ingress addon.
2. Prompts for `sudo` to securely add `semantic-analysis.test` to your `/etc/hosts` file (bypassing macOS `.local` DNS issues).
3. Connects to the internal Minikube Docker daemon.
4. Dynamically builds the Docker images (it automatically detects if you are on macOS ARM64 or Ubuntu AMD64 and installs the correct libraries to keep images small and prevent crashes).
5. Deploys the Kubernetes `.yaml` manifests and restarts the pods.

**After the script finishes, you must start the network tunnel:**
```bash
minikube tunnel
```
*(Leave that terminal window open in the background).*

Then simply open your browser and go to: **[http://semantic-analysis.test](http://semantic-analysis.test)**

---

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

1.  **Persistent Volume Claim (`backend-pvc.yaml`):** Secures 1Gi of persistent storage for the fine-tuned machine learning models, ensuring they aren't lost if a pod restarts.
2.  **ConfigMaps (`configmap.yaml`):** Centralizes application configuration (like `VITE_API_URL` and `MODEL_NAME`), allowing for easy environment management without rebuilding images.
3.  **Backend Deployment & Service:**
    *   **Probes:** Includes **Liveness** and **Readiness** probes. Readiness ensures the pod doesn't receive traffic until the ML model is fully loaded; Liveness automatically restarts the container if the API hangs.
    *   **Resource Management:** Implements **CPU/Memory Limits and Requests** (e.g., 2Gi Memory limit) to ensure backend stability and prevent resource starvation.
    *   **Service:** Exposed internally via a `ClusterIP` on port 8000.
4.  **Frontend Deployment & Service:** Includes health checks and resource limits, exposed internally via `ClusterIP`.
5.  **Advanced Ingress (`ingress.yaml`):** Provides a production-grade "Front Door." It uses **Hostname Routing** (`semantic-analysis.local`) to direct `/api` traffic to the backend and `/` traffic to the frontend UI.
6.  **Automated CI/CD:** The updated **`JenkinsFile`** now handles the full deployment lifecycle, automatically applying these manifests to the cluster and performing a `rollout restart` to update running pods with new images.

### Observability & Centralized Logging (ELK Stack)
To ensure operational visibility and debugging support, the system incorporates the **ELK Stack** (plus Filebeat) for centralized logging and monitoring. Think of it as a factory pipeline that turns raw text logs into actionable insights.

1.  **Filebeat (The Courier)**: It runs as a background agent on our Kubernetes nodes. As soon as our **FastAPI Backend** or **Frontend** writes a log line, Filebeat immediately "harvests" it and ships it off to Logstash. It ensures that no logs are lost even if a container restarts.
2.  **Logstash (The Processor)**: This is the "brain" of the pipeline. It takes raw data and transforms it. We use it to parse the **JSON logs** coming from the backend. It extracts the `similarity_score`, `is_paraphrase`, and `response_time_ms` fields so they become searchable variables rather than just lines of text.
3.  **Elasticsearch (The Search Engine)**: It is a powerful, distributed database designed specifically for searching through massive amounts of data. It stores and indexes all our processed logs. Because it's an "indexed" storage, you can search for a specific error or a specific similarity score across millions of logs in milliseconds.
4.  **Kibana (The Visual Dashboard)**: The web interface that sits on top of Elasticsearch. It allows you to create **real-time dashboards** with pie charts (e.g., "Paraphrase vs. Non-Paraphrase distribution") or line graphs (e.g., "API Latency over the last hour").

**Why is this important?**
Without this stack, if your application crashes in a cluster of 50 containers, you would have to manually log into each container to find the error. With the ELK stack, you just go to one URL (Kibana) and see everything in a single, beautiful dashboard.

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

## 4. End-to-End Deployment Journey (The "How it Works" Flow)

When we start the deployment process, the system follows a precise sequence to ensure everything from the network layer to the machine learning model is perfectly synchronized.

### Step 1: Local Environment Preparation (`./startup.sh`)
Everything begins with a single command. The `startup.sh` script:
1.  **Orchestrator Check**: It verifies if **Minikube** is running. If not, it initializes the cluster and enables the **NGINX Ingress Addon**, which acts as the "front door" for our cluster.
2.  **DNS & Routing**: It securely maps `semantic-analysis.test` to your local IP address in `/etc/hosts`. This allows you to use a real domain name instead of a cryptic IP address.
3.  **Cross-Platform Image Building**: It detects your hardware (macOS ARM64 vs. Ubuntu AMD64) and builds optimized Docker images directly inside the Minikube environment. This ensures the ML libraries (like Torch and Transformers) are compiled correctly for your specific chip.

### Step 2: Orchestration & Self-Healing (Kubernetes)
Once images are ready, the Kubernetes manifests are applied:
1.  **Deployment**: K8s spins up the Backend and Frontend pods. 
2.  **Health Checks**: The **Readiness Probes** ensure the UI (Frontend) doesn't start sending requests to the Backend until the large MPNet model is fully loaded into RAM.
3.  **Ingress Routing**: The Ingress Controller detects the new services and routes `http://semantic-analysis.test/api` to the Backend and `http://semantic-analysis.test/` to the React UI.

### Step 3: Accessing the UI
At this point, you run `minikube tunnel`. This creates a network bridge between your computer and the isolated K8s cluster. You can then open your browser and see the **ParaSense AI** interface ready for input.

---

## 5. The Automated Jenkins CI/CD Pipeline (Production Lifecycle)

While `startup.sh` is for local setup, the **Jenkins Pipeline** is our production-grade automation engine. It follows a "Quality-First" approach defined in the `JenkinsFile`.

### Stage 1: The Quality Gate (Smoke Testing)
Unlike simple pipelines, we don't deploy unless the model passes a specific accuracy threshold:
-   **Temporary Environment**: Jenkins builds a temporary backend container.
-   **Validation Suite**: It runs `tests/test50.py`, which contains 50 complex semantic scenarios (negation traps, adversarial swaps).
-   **The Gate**: If the pass percentage is **below 60%**, the pipeline **self-destructs** (fails) and prevents the deployment. This ensures that a "broken" or "dumb" model never reaches the users.

### Stage 2: Containerization & Versioning
Once the tests pass:
-   **Docker Hub Push**: The images are tagged and pushed to **Docker Hub**. This creates a centralized "Golden Image" that any server in the world can pull.
-   **Secure Credentials**: Jenkins uses encrypted credentials (`DockerHubCred`) to log in, ensuring our registry remains secure.

### Stage 3: Rolling Update & Monitoring
The final act is the deployment to the live cluster:
-   **Zero-Downtime Rollout**: Jenkins executes `kubectl rollout restart`. Kubernetes performs a "Rolling Update," where it starts new pods before killing the old ones, ensuring users never see a "Service Unavailable" page.
-   **Automated Monitoring**: Finally, the pipeline applies the ELK manifests (`k8s/monitoring/`). It doesn't just deploy the app; it also ensures the **Elasticsearch, Logstash, and Kibana** infrastructure is up and running to watch the new deployment.

---

## 6. Future Improvement Plan
As outlined in `IMPROVEMENT_PLAN.md`, the next phase involves migrating to a Multi-Task Training approach utilizing Knowledge Distillation to achieve >85% universal accuracy across highly diverse domains.

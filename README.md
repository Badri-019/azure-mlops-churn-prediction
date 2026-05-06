# Azure MLOps — Customer Churn Prediction

A production-grade MLOps pipeline for customer churn prediction, built on Azure. This project demonstrates end-to-end MLOps practices including model versioning, containerization, Kubernetes-based model serving, and GitOps-based continuous deployment.

---

## What does this model do?

Predicts whether a customer is likely to cancel their subscription based on behavioral and billing data.

**Example input:**

```json
{
  "age": 45,
  "tenure_months": 24,
  "monthly_charges": 79.99,
  "total_charges": 1920.00,
  "num_support_calls": 3
}
```

**Example output:**

```json
{
  "churn": 1,
  "churn_probability": 0.73
}
```

The model looks for patterns like high monthly charges, frequent support calls, and low tenure — all signals that a customer may leave. Businesses can use this to proactively intervene with targeted offers or support before the customer churns.

---

## Architecture

```
Developer (git push)
        │
        ▼
GitHub repository (azure-mlops-cicd branch)
        │
        ├──► GitHub Actions CI Pipeline
        │         ├── Train model (generate_data.py + train.py)
        │         ├── DVC push → Azure Blob Storage (dvc-store/)
        │         ├── Upload model → Azure Blob Storage (kserve-model/)
        │         ├── Docker build + push → Azure Container Registry
        │         └── Update k8s/inference.yaml → git commit
        │
        ├──► ArgoCD (watches k8s/ folder)
        │         └── Auto-sync → AKS cluster
        │
        └──► Azure Services
                  ├── Azure Blob Storage  — model versioning (DVC) + KServe serving
                  ├── Azure Container Registry — Docker images
                  ├── Managed Identity + Workload Identity — zero-secret auth
                  └── AKS cluster
                            ├── cert-manager
                            ├── KServe (RawDeployment mode)
                            │     └── InferenceService: churn-predictor
                            │           └── storage-initializer pulls model from Blob
                            └── ArgoCD
```

---

## Tech stack

| Component | Technology |
|---|---|
| Model training | Python, scikit-learn |
| API server | FastAPI |
| Model versioning | DVC |
| Model storage | Azure Blob Storage |
| Container registry | Azure Container Registry (ACR) |
| Container orchestration | Azure Kubernetes Service (AKS) |
| Model serving | KServe (RawDeployment mode) |
| CI pipeline | GitHub Actions |
| CD pipeline | ArgoCD (GitOps) |
| Auth (no secrets) | Azure Workload Identity + Managed Identity |

---

## Project structure

```
azure-mlops/
├── api.py                          # FastAPI inference server
├── train.py                        # Model training script
├── generate_data.py                # Synthetic dataset generator
├── requirements.txt                # Python dependencies
├── Dockerfile                      # Container image
├── .dvc/
│   └── config                      # DVC remote configuration (Azure Blob)
├── models/
│   └── churn_model.pkl.dvc         # DVC metadata for model
├── data/
│   └── churn_data.csv.dvc          # DVC metadata for dataset
├── k8s/
│   ├── serviceaccount.yaml         # Kubernetes ServiceAccount (Workload Identity)
│   └── inference.yaml              # KServe InferenceService manifest
└── .github/
    └── workflows/
        └── mlops-pipeline.yaml     # GitHub Actions CI pipeline
```

---

## Local setup

### Prerequisites

- Python 3.10+
- Docker

### Run locally

```bash
# Clone the repo
git clone https://github.com/Badri-019/azure-mlops-churn-prediction.git
cd azure-mlops-churn-prediction
git checkout azure-mlops-cicd

# Create virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Generate synthetic dataset
python generate_data.py

# Train the model
python train.py

# Start the FastAPI server
python api.py
# Visit http://localhost:8000/docs
```

### Test the API locally

```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "age": 45,
    "tenure_months": 24,
    "monthly_charges": 79.99,
    "total_charges": 1920.00,
    "num_support_calls": 3
  }'
```

---

## Azure infrastructure setup

### Prerequisites

- Azure account with an active subscription
- Azure Cloud Shell (CLI, kubectl, and helm pre-installed)

### Step 1 — Resource group and storage

```bash
az group create --name mlops-churn-rg --location centralindia

az storage account create \
  --name mlopschurnbadri019 \
  --resource-group mlops-churn-rg \
  --location centralindia \
  --sku Standard_LRS

az storage container create \
  --name churn-model \
  --account-name mlopschurnbadri019 \
  --auth-mode login
```

### Step 2 — Managed Identity and Workload Identity

```bash
az identity create \
  --name mlops-churn-identity \
  --resource-group mlops-churn-rg \
  --location centralindia

# Assign Storage Blob Data Contributor role
PRINCIPAL_ID=$(az identity show \
  --name mlops-churn-identity \
  --resource-group mlops-churn-rg \
  --query principalId -o tsv)

STORAGE_ID=$(az storage account show \
  --name mlopschurnbadri019 \
  --resource-group mlops-churn-rg \
  --query id -o tsv)

az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope $STORAGE_ID
```

### Step 3 — AKS cluster

```bash
az aks create \
  --resource-group mlops-churn-rg \
  --name mlops-churn-aks \
  --location centralindia \
  --node-count 2 \
  --node-vm-size Standard_B4ms \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --generate-ssh-keys

az aks get-credentials \
  --resource-group mlops-churn-rg \
  --name mlops-churn-aks
```

### Step 4 — Federated credential (Workload Identity bridge)

```bash
OIDC_ISSUER=$(az aks show \
  --name mlops-churn-aks \
  --resource-group mlops-churn-rg \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)

az identity federated-credential create \
  --name mlops-churn-federated \
  --identity-name mlops-churn-identity \
  --resource-group mlops-churn-rg \
  --issuer $OIDC_ISSUER \
  --subject "system:serviceaccount:churn-model:churn-sa" \
  --audience api://AzureADTokenExchange
```

### Step 5 — KServe installation

```bash
# cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl wait --for=condition=Ready pods --all -n cert-manager --timeout=120s

# KServe
kubectl apply -f https://github.com/kserve/kserve/releases/download/v0.14.0/kserve.yaml

# Patch to RawDeployment mode (no Knative required)
kubectl patch configmap inferenceservice-config \
  -n kserve \
  --type merge \
  -p '{"data": {"deploy": "{\"defaultDeploymentMode\": \"RawDeployment\"}"}}'

kubectl rollout restart deployment/kserve-controller-manager -n kserve
```

### Step 6 — Deploy to Kubernetes

```bash
kubectl create namespace churn-model
kubectl apply -f k8s/serviceaccount.yaml
kubectl apply -f k8s/inference.yaml
```

### Step 7 — Test KServe endpoint

```bash
kubectl port-forward -n churn-model svc/churn-predictor-predictor 8080:80 &

curl -X POST http://localhost:8080/v1/models/churn-predictor:predict \
  -H "Content-Type: application/json" \
  -d '{"instances": [[45, 24, 79.99, 1920.00, 3]]}'
```

Expected response:

```json
{"predictions": [1]}
```

---

## CI/CD pipeline

### GitHub Actions CI (`.github/workflows/mlops-pipeline.yaml`)

Triggers on every push to the `azure-mlops-cicd` branch and performs:

1. Train a fresh model using `generate_data.py` + `train.py`
2. Push model to Azure Blob Storage via DVC (`dvc-store/` for versioning)
3. Upload model directly to `kserve-model/model.pkl` for KServe serving
4. Build Docker image and push to Azure Container Registry
5. Update `k8s/inference.yaml` with the new image tag (commit SHA)
6. Commit and push the updated manifest back to GitHub

### Required GitHub secrets

| Secret | Description |
|---|---|
| `AZURE_CREDENTIALS` | Service principal JSON for Azure login |
| `ACR_LOGIN_SERVER` | ACR login server (e.g. `mlopschurnacr.azurecr.io`) |
| `ACR_USERNAME` | Service principal client ID |
| `ACR_PASSWORD` | Service principal client secret |
| `AZURE_STORAGE_ACCOUNT` | Storage account name |
| `AZURE_STORAGE_KEY` | Storage account key |

### ArgoCD GitOps CD

ArgoCD watches the `k8s/` folder in the `azure-mlops-cicd` branch. When GitHub Actions commits an updated `inference.yaml`, ArgoCD detects the change and automatically syncs the new InferenceService to AKS — deploying the new model version without any manual intervention.

```
code push → CI trains + builds → manifest updated → ArgoCD syncs → new model live
```

---

## Workload Identity — zero secrets in Kubernetes

Instead of storing Azure credentials as Kubernetes secrets, this project uses AKS Workload Identity:

- The `churn-sa` ServiceAccount is annotated with the Managed Identity client ID
- AKS automatically injects a short-lived OIDC token into the pod
- KServe's storage-initializer exchanges the token for Azure AD credentials at runtime
- The Managed Identity has `Storage Blob Data Contributor` role on the storage account
- No storage keys or connection strings exist anywhere in the cluster

---

## Cost management

This project uses `Standard_B4ms` nodes (4 vCPU, 16 GB RAM) which is the minimum comfortable size for KServe + cert-manager. To avoid ongoing charges when not in use:

```bash
# Stop the cluster (keeps config, stops billing for VMs)
az aks stop --name mlops-churn-aks --resource-group mlops-churn-rg

# Start it again
az aks start --name mlops-churn-aks --resource-group mlops-churn-rg

# Or delete everything
az group delete --name mlops-churn-rg --yes
```

---

## Acknowledgements

This project is based on the MLOps Zero to Hero course by [Abhishek Veeramalla](https://github.com/iam-veeramalla), adapted from the original AWS/KIND setup to a full Azure-native stack with AKS, Azure Blob Storage, ACR, and Workload Identity.

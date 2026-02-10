# ESO Demo - External Secrets Operator with Infisical

A demo application showing how to use the External Secrets Operator (ESO) to sync secrets from Infisical into Kubernetes using Kubernetes Auth.

## Architecture

```
Infisical (source of truth)
    |
    |  Kubernetes Auth (TokenReview)
    v
SecretStore --> ExternalSecret --> K8s Secret "eso-demo-secret" --> Pod (env vars)
```

## Project Structure

```
.
├── app.py                              # Flask app - reads env vars and displays them
├── templates/
│   └── index.html                      # Dark-themed UI showing secret values
├── Dockerfile                          # Container image (python:3.12-slim)
├── requirements.txt                    # Flask dependency
└── Demo-files/
    ├── rbac-setup.yaml                 # ServiceAccount + ClusterRoleBinding for token review
    ├── infisical-identity.yaml         # Machine Identity ID reference
    ├── secretstore.yaml                # ESO connection config (Infisical + Kubernetes Auth)
    ├── externalsecret.yaml             # Secret mappings (what to fetch from Infisical)
    ├── deployment.yaml                 # App deployment (pulls from ECR)
    └── service.yaml                    # LoadBalancer service
```

## Prerequisites

- AWS account with CLI configured (`aws configure`)
- `kubectl`, `helm`, `eksctl` installed
- An Infisical account with a project containing your secrets
- A Machine Identity in Infisical with Kubernetes Auth enabled

## Quick Start

### 1. Create EKS cluster

```bash
eksctl create cluster --name eso-demo --region us-east-1 --nodes 2 --node-type t3.medium
```

### 2. Build and push the image (first time only)

```bash
aws ecr get-login-password | docker login --username AWS --password-stdin 162856940288.dkr.ecr.us-east-1.amazonaws.com
docker build --platform linux/amd64 -t 162856940288.dkr.ecr.us-east-1.amazonaws.com/eso-demo:latest .
docker push 162856940288.dkr.ecr.us-east-1.amazonaws.com/eso-demo:latest
```

### 3. Deploy the app

```bash
kubectl apply -f Demo-files/deployment.yaml
kubectl apply -f Demo-files/service.yaml
```

### 4. Install ESO

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets --create-namespace --set installCRDs=true
```

### 5. Set up RBAC and get the token reviewer JWT

```bash
kubectl apply -f Demo-files/rbac-setup.yaml
kubectl get secret infisical-token-reviewer-token -o=jsonpath='{.data.token}' | base64 --decode
```

Update the Token Reviewer JWT and Kubernetes Host (`kubectl cluster-info`) in Infisical's Machine Identity Kubernetes Auth config.

### 6. Apply ESO manifests

```bash
kubectl apply -f Demo-files/infisical-identity.yaml
kubectl apply -f Demo-files/secretstore.yaml
kubectl apply -f Demo-files/externalsecret.yaml
```

### 7. Verify and restart

```bash
kubectl get externalsecret eso-demo-external
kubectl rollout restart deployment eso-demo
kubectl get svc eso-demo
```

Open `http://<EXTERNAL-IP>:5000` in your browser.

## Teardown

```bash
eksctl delete cluster --name eso-demo --region us-east-1
```

ECR repo is cheap to keep. To remove it too:

```bash
aws ecr delete-repository --repository-name eso-demo --force
```

## Notes

- Each new EKS cluster generates a new token reviewer JWT and API server URL. Update both in Infisical's Machine Identity config after cluster creation.
- The identity ID in `infisical-identity.yaml` is not a sensitive credential -- it's a reference ID that is useless without the Kubernetes Auth flow.
- ESO syncs are all-or-nothing: if any key in the ExternalSecret fails to fetch, none of them sync.
- The app displays 5 env vars: `DB_HOST`, `DB_PORT`, `DB_USERNAME`, `DB_PASSWORD`, `API_KEY`.

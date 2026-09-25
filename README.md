# Release - Kubernetes Deployment Infrastructure

This repository contains the Kubernetes deployment configuration and GitHub Actions workflows used to deploy services to the production Kubernetes cluster.

The repository follows a **Kustomize-based Kubernetes deployment architecture** with separate configurations for each service.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Repository Structure](#repository-structure)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Deployment Workflow](#deployment-workflow)
- [How to Deploy a New Service](#how-to-deploy-a-new-service)
- [Configure GitHub Actions](#configure-github-actions)
- [Configure Ingress](#configure-ingress)

---

# Overview

The repository provides infrastructure for deploying backend services to Kubernetes.

The main technologies used are:

- Kubernetes
- Kustomize
- GitHub Actions
- GitHub Container Registry (GHCR)
- Oracle Kubernetes Engine (OKE)
- NGINX Ingress
- cert-manager
- Let's Encrypt

The deployment process is automated through GitHub Actions.

A typical deployment flow looks like:

```text
Developer
    |
    | Push code
    v
GitHub Repository
    |
    v
Build Docker Image
    |
    v
GitHub Container Registry
    |
    v
GitHub Actions
    |
    v
Kubernetes / OKE
    |
    v
Kubernetes Deployment
    |
    v
Service
    |
    v
NGINX Ingress
    |
    v
Application
```

# Repository Structure
```
.
├── .github/
│   ├── actions/
│   │   └── generate-secrets/
│   │       └── action.yml
│   │
│   └── workflows/
│       ├── dispatch-deploy.yml
│       └── infra-deploy.yml
│
├── apps/
│   └── auth-service/
│       ├── base/
│       │   ├── configmap.yml
│       │   ├── deployment.yaml
│       │   ├── kustomization.yml
│       │   └── service.yaml
│       │
│       └── overlays/
│           └── prod/
│               ├── kustomization.yaml
│               └── secret.env
│
├── ingress.yaml
├── issuer.yaml
└── README.md
```

# Architecture

Each `application/service` is maintained independently under:

`apps/<service-name>/`

Each service contains two main Kubernetes configuration layers:

```
base/
overlays/
```

- The `base` directory contains the common Kubernetes resources.

- The `overlays/prod` directory contains production-specific configuration.

### Example
```
apps/
└── auth-service/
    ├── base/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── configmap.yml
    │   └── kustomization.yml
    │
    └── overlays/
        └── prod/
            ├── kustomization.yaml
            └── secret.env
```

# How to Deploy a New Service

Suppose we want to add a new service:

`order-service`

The recommended structure is:
```
apps/
└── order-service/
    ├── base/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── configmap.yml
    │   └── kustomization.yml
    │
    └── overlays/
        └── prod/
            ├── kustomization.yaml
```

### Follow the steps below.
- Make sure you added all the required files with service specific changes Ex: `Package Image`

# Configure Github Actions

### step-1
Inside `.github/workflows/infra-deploy.yml`
Line number 17 & 45 Add
```
filters: |
            auth-service: 'apps/auth-service/**'
            order-service: 'apps/order-service/**' # your service
```

Line number 30 add service specific `ENV`
```
env:
      AUTH_SERVICE_SECRETS: ${{ secrets.AUTH_SERVICE_SECRETS }}
      ORDER_SERVICE_SECRETS: ${{secrets.ORDER_SERVICE_SECRETS}} # Deploy service env
```

### Step-2
Inside `.github/workflows/dispatch-deploy.yml` file add env

```
env:
      AUTH_SERVICE_SECRETS: ${{ secrets.AUTH_SERVICE_SECRETS }}
      ORDER_SERVICE_SECRETS: ${{secrets.ORDER_SERVICE_SECRETS}} # Deploy service env
```

### Step-3 Configure Secrets
Add ENV to list
```
env:
        SERVICE: ${{ inputs.service }}
        AUTH_SECRETS: ${{ env.AUTH_SERVICE_SECRETS }}
        ORDER_SECRETS: ${{ env.ORDER_SERVICE_SECRETS }} # Add to store the secrets
```

Add service Name to list
```
case "$SERVICE" in
          auth-service)
            echo "$AUTH_SECRETS" > "$TARGET_FILE"
            ;;
          order-service) # should match to service repo name
            echo "$ORDER_SECRETS" > "$TARGET_FILE"
            ;;
          *)
            echo "Unknown service: $SERVICE"
            exit 1
            ;;
        esac
```

# Configure Ingress
Add your service path to NGINX Ingress server. Make sure the Indentation is properly configured
```
rules:
    - host: backend.stud-hub.me
      http:
        paths:
          - path: /auth(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: auth-service
                port:
                  number: 80
        

        <----- 👇👇👇 Add your deployed service here 👇👇👇 ---->
          - path: /order(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: order-service
                port:
                  number: 80

        <---- ☝️☝️☝️ Above ☝️☝️☝️ ----->


          - path: /query
            pathType: Prefix
            backend:
              service:
                name: auth-service
                port:
                  number: 80
```

# Merge to Main
Git add and merge to main
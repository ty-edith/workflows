# Getting Started Guide

This guide will help you set up and use the GitHub Actions workflows for deploying applications to Google Cloud Run.

## Quick Start

1. [Prerequisites](#prerequisites)
2. [Setup Google Cloud](#setup-google-cloud)
3. [Configure GitHub](#configure-github)
4. [Create Templates](#create-templates)
5. [Create Your First Workflow](#create-your-first-workflow)
6. [Deploy](#deploy)

## Prerequisites

Before you begin, ensure you have:

- A Google Cloud Platform account with billing enabled
- A GitHub repository with a containerized application
- A `Dockerfile` in your repository root
- Administrative access to your GitHub repository

## Setup Google Cloud

### 1. Create a GCP Project

```bash
# Create a new project (or use an existing one)
gcloud projects create my-app-project --name="My App"

# Set the project as default
gcloud config set project my-app-project
```

### 2. Enable Required APIs

```bash
# Enable necessary APIs
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  iam.googleapis.com \
  cloudresourcemanager.googleapis.com
```

### 3. Create Artifact Registry Repository

```bash
# Create Docker repository
gcloud artifacts repositories create cloud-run-source-deploy \
  --repository-format=docker \
  --location=us-central1 \
  --description="Docker repository for Cloud Run deployments"
```

### 4. Create Service Account

```bash
# Create service account
gcloud iam service-accounts create github-actions \
  --description="Service account for GitHub Actions" \
  --display-name="GitHub Actions"

# Grant necessary roles
gcloud projects add-iam-policy-binding my-app-project \
  --member="serviceAccount:github-actions@my-app-project.iam.gserviceaccount.com" \
  --role="roles/run.admin"

gcloud projects add-iam-policy-binding my-app-project \
  --member="serviceAccount:github-actions@my-app-project.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

gcloud projects add-iam-policy-binding my-app-project \
  --member="serviceAccount:github-actions@my-app-project.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"
```

### 5. Setup Workload Identity Federation

```bash
# Create workload identity pool
gcloud iam workload-identity-pools create github-pool \
  --location=global \
  --description="Pool for GitHub Actions"

# Create workload identity provider
gcloud iam workload-identity-pools providers create-oidc github-provider \
  --location=global \
  --workload-identity-pool=github-pool \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository=='YOUR_GITHUB_ORG/YOUR_REPO'"

# Allow GitHub to impersonate the service account
gcloud iam service-accounts add-iam-policy-binding \
  github-actions@my-app-project.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/attribute.repository/YOUR_GITHUB_ORG/YOUR_REPO"
```

**Important**: Replace `YOUR_GITHUB_ORG/YOUR_REPO` with your actual GitHub organization and repository name, and `PROJECT_NUMBER` with your GCP project number.

### 6. Get Project Information

```bash
# Get project number
gcloud projects describe my-app-project --format="value(projectNumber)"

# Get the workload identity provider resource name
gcloud iam workload-identity-pools providers describe github-provider \
  --location=global \
  --workload-identity-pool=github-pool \
  --format="value(name)"
```

## Configure GitHub

### 1. Add Repository Secrets

Go to your GitHub repository → Settings → Secrets and variables → Actions, and add these secrets:

| Secret Name | Value | Example |
|-------------|-------|---------|
| `GCP_PROJECT_ID` | Your GCP project ID | `my-app-project` |
| `GCP_SERVICE_ACCOUNT` | Service account email | `github-actions@my-app-project.iam.gserviceaccount.com` |
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | Full provider resource name | `projects/123456789/locations/global/workloadIdentityPools/github-pool/providers/github-provider` |
| `GCP_REGION` | GCP region | `us-central1` |

### 2. Add the Workflows Repository

Add this workflows repository as a submodule or reference it directly in your workflows.

## Create Templates

Create the following directory structure in your repository:

```
.github/
└── resources/
    ├── values.yaml
    ├── service.yaml
    ├── migration.job.yaml  # Optional, for database migrations
    └── env/
        ├── test.values.yaml
        └── production.values.yaml
```

### 1. Base Values (`values.yaml`)

```yaml
#@data/values
---
service_name: my-app
memory: 512Mi
cpu: 1
min_instances: 0
max_instances: 10
port: 8080
timeout: 300
```

### 2. Cloud Run Service Template (`service.yaml`)

```yaml
#@ load("@ytt:data", "data")
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: #@ data.values.service_name
  annotations:
    run.googleapis.com/ingress: all
spec:
  template:
    metadata:
      annotations:
        run.googleapis.com/service-account: #@ data.values.service_account
        autoscaling.knative.dev/minScale: #@ str(data.values.min_instances)
        autoscaling.knative.dev/maxScale: #@ str(data.values.max_instances)
        run.googleapis.com/execution-environment: gen2
    spec:
      containerConcurrency: 80
      timeoutSeconds: #@ data.values.timeout
      containers:
      - image: #@ data.values.image_url
        ports:
        - containerPort: #@ data.values.port
        resources:
          limits:
            memory: #@ data.values.memory
            cpu: #@ data.values.cpu
        env:
        - name: PORT
          value: #@ str(data.values.port)
        - name: COMMIT_SHA
          value: #@ data.values.commit_sha
```

### 3. Environment-Specific Values

**Test Environment** (`env/test.values.yaml`):

```yaml
#@data/values
---
service_name: my-app-test
memory: 512Mi
cpu: 1
min_instances: 0
max_instances: 5
```

**Production Environment** (`env/production.values.yaml`):

```yaml
#@data/values
---
service_name: my-app
memory: 1Gi
cpu: 2
min_instances: 1
max_instances: 50
timeout: 600
```

## Create Your First Workflow

Create `.github/workflows/deploy.yml` in your repository:

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    if: github.event_name == 'push'
    uses: ty-edith/workflows/.github/workflows/build-and-push.yml@v1
    with:
      project_id: ${{ secrets.GCP_PROJECT_ID }}
      service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
      workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      region: ${{ secrets.GCP_REGION }}

  deploy-test:
    if: github.event_name == 'push' && github.ref != 'refs/heads/main'
    needs: build
    uses: ty-edith/workflows/.github/workflows/release.yml@v1
    with:
      image_url: ${{ needs.build.outputs.image-url }}
      environment: "test"
      project_id: ${{ secrets.GCP_PROJECT_ID }}
      service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
      workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      region: ${{ secrets.GCP_REGION }}

  deploy-production:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: build
    uses: ty-edith/workflows/.github/workflows/release.yml@v1
    with:
      image_url: ${{ needs.build.outputs.image-url }}
      environment: "production"
      project_id: ${{ secrets.GCP_PROJECT_ID }}
      service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
      workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      region: ${{ secrets.GCP_REGION }}
```

## Deploy

### 1. Test Your Setup

1. Commit and push your changes to a feature branch
2. Create a pull request to trigger the workflow
3. Check the Actions tab in GitHub to see the workflow run

### 2. Deploy to Production

1. Merge your pull request to the main branch
2. The workflow will automatically build and deploy to production
3. Check the Cloud Run console to see your deployed service

### 3. Access Your Application

After deployment, you can find your service URL in:

- The GitHub Actions workflow output
- Google Cloud Console → Cloud Run → Your Service

## Next Steps

### Add Database Migrations

If your application uses a database, create a migration job template:

```yaml
#@ load("@ytt:data", "data")
apiVersion: run.googleapis.com/v1
kind: Job
metadata:
  name: #@ data.values.service_name + "-migration"
spec:
  template:
    spec:
      template:
        metadata:
          annotations:
            run.googleapis.com/service-account: #@ data.values.service_account
        spec:
          restartPolicy: Never
          containers:
          - image: #@ data.values.image_url
            command: ["python", "manage.py", "migrate"]  # Adjust for your framework
            resources:
              limits:
                memory: 512Mi
                cpu: 1
```

Then add `migrate: true` to your release workflow calls.

### Configure Custom Domains

1. Set up domain mapping in Cloud Run
2. Add domain configuration to your service template
3. Configure DNS records

### Add Monitoring

1. Set up Cloud Monitoring alerts
2. Configure structured logging
3. Add health check endpoints to your application

## Troubleshooting

### Common Issues

1. **Authentication Failed**: Double-check Workload Identity Federation setup
2. **Permission Denied**: Verify service account has necessary IAM roles
3. **Template Errors**: Validate YAML templates with ytt locally
4. **Build Failures**: Check Dockerfile and dependencies

### Getting Help

- Check GitHub Actions workflow logs
- Review Google Cloud Console for resource status
- Validate templates locally with ytt
- Test Docker builds locally

### Useful Commands

```bash
# Test ytt templates locally
ytt -f .github/resources/service.yaml \
    --data-values-file .github/resources/values.yaml \
    --data-values-file .github/resources/env/test.values.yaml \
    --data-value image_url=test-image

# Test Docker build locally
docker build -t my-app .

# Check Cloud Run services
gcloud run services list

# View service logs
gcloud logs read --service=my-app
```

## Security Best Practices

1. **Least Privilege**: Grant minimal necessary permissions
2. **Environment Separation**: Use different projects for test/prod
3. **Secret Management**: Store sensitive data in Secret Manager
4. **Regular Audits**: Review IAM permissions regularly
5. **Monitoring**: Set up alerts for unusual activity

Congratulations! You now have a complete CI/CD pipeline for deploying to Google Cloud Run. Your applications will be automatically built and deployed whenever you push changes to your repository.

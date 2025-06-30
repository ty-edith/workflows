# GitHub Actions Workflows

This repository contains reusable GitHub Actions workflows for building, deploying, and managing applications on Google Cloud Platform (GCP) using Cloud Run.

## Overview

The workflows in this repository provide a complete CI/CD pipeline for containerized applications:

1. **Build and Push** - Builds Docker images and pushes them to Google Artifact Registry
2. **Release** - Deploys applications to Cloud Run with optional database migrations

## Workflows

### 🏗️ Build and Push (`build-and-push.yml`)

Builds a Docker image from your repository and pushes it to Google Artifact Registry.

**Features:**

- Builds Docker images using the Dockerfile in your repository root
- Pushes to Google Artifact Registry
- Supports custom tags or automatic SHA-based tagging
- Configurable repository name

**Outputs:**

- `image-url`: The full URL of the built Docker image

[View detailed documentation →](./docs/build-and-push.md)

### 🚀 Release (`release.yml`)

Deploys applications to Google Cloud Run with support for database migrations.

**Features:**

- Deploys to Cloud Run using declarative YAML templates
- Optional database migration jobs
- Environment-specific configurations
- Supports both test and production deployments
- Uses `ytt` for YAML templating

[View detailed documentation →](./docs/release.md)

## Prerequisites

### Google Cloud Setup

1. **GCP Project**: You need a Google Cloud Project with the following APIs enabled:
   - Cloud Run API
   - Artifact Registry API
   - Identity and Access Management (IAM) API

2. **Service Account**: Create a service account with the following roles:
   - Cloud Run Admin
   - Artifact Registry Writer
   - Service Account User

3. **Workload Identity Federation**: Set up Workload Identity Federation to allow GitHub Actions to authenticate with Google Cloud without storing service account keys.

### Repository Setup

1. **Dockerfile**: Your repository must contain a `Dockerfile` in the root directory

2. **YAML Templates** (for release workflow):
   - `.github/resources/service.yaml` - Cloud Run service template
   - `.github/resources/migration.job.yaml` - Migration job template (if using migrations)
   - `.github/resources/values.yaml` - Base configuration values
   - `.github/resources/env/{environment}.values.yaml` - Environment-specific values

### GitHub Secrets

Configure the following secrets in your repository:

- `GCP_PROJECT_ID`: Your Google Cloud Project ID
- `GCP_SERVICE_ACCOUNT`: Service account email
- `GCP_WORKLOAD_IDENTITY_PROVIDER`: Workload Identity Provider resource name
- `GCP_REGION`: Google Cloud region (e.g., `us-central1`)

## Usage Examples

### Basic Build and Deploy

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]

jobs:
  build:
    uses: ty-edith/workflows/.github/workflows/build-and-push.yml@v1
    with:
      project_id: ${{ secrets.GCP_PROJECT_ID }}
      service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
      workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      region: ${{ secrets.GCP_REGION }}

  deploy:
    needs: build
    uses: ty-edith/workflows/.github/workflows/release.yml@v1
    with:
      image_url: ${{ needs.build.outputs.image-url }}
      environment: production
      project_id: ${{ secrets.GCP_PROJECT_ID }}
      service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
      workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      region: ${{ secrets.GCP_REGION }}
```

### Deploy with Database Migration

```yaml
deploy-with-migration:
  needs: build
  uses: ty-edith/workflows/.github/workflows/release.yml@v1
  with:
    image_url: ${{ needs.build.outputs.image-url }}
    environment: production
    migrate: true
    project_id: ${{ secrets.GCP_PROJECT_ID }}
    service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
    workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
    region: ${{ secrets.GCP_REGION }}
```

## Best Practices

1. **Environment Separation**: Use different GCP projects or service accounts for test and production environments

2. **Version Tagging**: Use semantic versioning for production deployments:

   ```yaml
   with:
     tag: "v1.2.3"
   ```

3. **Security**:

   - Never store GCP credentials in repository secrets
   - Use Workload Identity Federation for secure authentication
   - Regularly rotate service account keys if using them

4. **Monitoring**:

   - Set up Cloud Monitoring alerts for your deployments
   - Use structured logging in your applications
   - Monitor resource usage and costs

## Troubleshooting

### Common Issues

1. **Authentication Errors**: Verify Workload Identity Federation is correctly configured
2. **Permission Denied**: Ensure service account has necessary IAM roles
3. **Image Not Found**: Check that the Artifact Registry repository exists
4. **Migration Failures**: Verify migration job templates and database connectivity

### Getting Help

- Check the GitHub Actions logs for detailed error messages
- Review Google Cloud Console for resource status
- Ensure all required APIs are enabled in your GCP project

## Contributing

When contributing to these workflows:

1. Test changes in a development environment first
2. Update documentation for any new features or parameters
3. Follow semantic versioning for releases
4. Add appropriate error handling and logging

## License

See [LICENSE](./LICENSE) file for details.

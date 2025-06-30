# Build and Push Workflow

The `build-and-push.yml` workflow builds Docker images from your repository and pushes them to Google Artifact Registry.

## Overview

This workflow is designed to be used as a reusable workflow that can be called from other workflows. It handles the complete process of building a Docker image and pushing it to Google Artifact Registry for later deployment.

## Inputs

| Parameter | Required | Type | Default | Description |
|-----------|----------|------|---------|-------------|
| `project_id` | ✅ | string | - | Google Cloud Project ID |
| `service_account` | ✅ | string | - | GCP service account email for authentication |
| `workload_identity_provider` | ✅ | string | - | Full provider resource name (e.g., `projects/302.../providers/github`) |
| `region` | ✅ | string | - | GCP region (e.g., `us-central1`) |
| `tag` | ❌ | string | SHA-based | Custom tag for the Docker image. If not provided, uses SHA |
| `repository` | ❌ | string | `cloud-run-source-deploy` | Artifact Registry repository name |

## Outputs

| Output | Description |
|--------|-------------|
| `image-url` | The full URL of the built Docker image in Artifact Registry |

## Prerequisites

### Repository Requirements

1. **Dockerfile**: Your repository must contain a `Dockerfile` in the root directory
2. **Buildable Application**: The Docker build process should complete successfully

### Google Cloud Setup

1. **Artifact Registry Repository**: Create an Artifact Registry repository in your GCP project:
   ```bash
   gcloud artifacts repositories create cloud-run-source-deploy \
     --repository-format=docker \
     --location=us-central1 \
     --description="Docker repository for Cloud Run deployments"
   ```

2. **Service Account Permissions**: The service account needs these IAM roles:
   - `roles/artifactregistry.writer`
   - `roles/iam.serviceAccountTokenCreator` (for Workload Identity)

3. **Workload Identity Federation**: Configure a workload identity pool and provider:
   ```bash
   # Create workload identity pool
   gcloud iam workload-identity-pools create "github-pool" \
     --location="global" \
     --description="Pool for GitHub Actions"

   # Create workload identity provider
   gcloud iam workload-identity-pools providers create-oidc "github-provider" \
     --location="global" \
     --workload-identity-pool="github-pool" \
     --issuer-uri="https://token.actions.githubusercontent.com" \
     --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository"
   ```

## Usage Examples

### Basic Usage

```yaml
jobs:
  build:
    uses: ty-edith/workflows/.github/workflows/build-and-push.yml@v1
    with:
      project_id: "my-gcp-project"
      service_account: "github-actions@my-gcp-project.iam.gserviceaccount.com"
      workload_identity_provider: "projects/123456789/locations/global/workloadIdentityPools/github-pool/providers/github-provider"
      region: "us-central1"
```

### With Custom Tag

```yaml
jobs:
  build:
    uses: ty-edith/workflows/.github/workflows/build-and-push.yml@v1
    with:
      project_id: "my-gcp-project"
      service_account: "github-actions@my-gcp-project.iam.gserviceaccount.com"
      workload_identity_provider: "projects/123456789/locations/global/workloadIdentityPools/github-pool/providers/github-provider"
      region: "us-central1"
      tag: "v1.2.3"
```

### With Custom Repository

```yaml
jobs:
  build:
    uses: ty-edith/workflows/.github/workflows/build-and-push.yml@v1
    with:
      project_id: "my-gcp-project"
      service_account: "github-actions@my-gcp-project.iam.gserviceaccount.com"
      workload_identity_provider: "projects/123456789/locations/global/workloadIdentityPools/github-pool/providers/github-provider"
      region: "us-central1"
      repository: "my-custom-repo"
```

### Complete CI/CD Pipeline

```yaml
name: CI/CD Pipeline

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
      tag: ${{ github.ref_name == 'main' && 'latest' || github.sha }}

  deploy-test:
    if: github.event_name == 'push' && github.ref_name == 'main'
    needs: build
    uses: ty-edith/workflows/.github/workflows/release.yml@v1
    with:
      image_url: ${{ needs.build.outputs.image-url }}
      environment: "test"
      project_id: ${{ secrets.GCP_PROJECT_ID }}
      service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
      workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      region: ${{ secrets.GCP_REGION }}
```

## Image URL Format

The generated image URL follows this pattern:

- **With custom tag**: `{region}-docker.pkg.dev/{project_id}/{repository}/{owner}/{repo_name}:{tag}`
- **With SHA**: `{region}-docker.pkg.dev/{project_id}/{repository}/{owner}/{repo_name}/sha:{github_sha}`

Examples:
- `us-central1-docker.pkg.dev/my-project/cloud-run-source-deploy/myorg/myapp:v1.2.3`
- `us-central1-docker.pkg.dev/my-project/cloud-run-source-deploy/myorg/myapp/sha:abc123def456`

## Workflow Steps

1. **Checkout code**: Uses `actions/checkout@v4` to get repository content
2. **Authenticate to Google Cloud**: Uses Workload Identity Federation for secure authentication
3. **Set up Cloud SDK**: Configures gcloud CLI with the project
4. **Configure Docker**: Sets up Docker to use gcloud as credential helper for Artifact Registry
5. **Set output**: Determines the image URL based on inputs
6. **Build Docker image**: Builds the image using the repository's Dockerfile
7. **Push Docker image**: Pushes the built image to Artifact Registry

## Troubleshooting

### Common Issues

1. **Authentication Failed**
   - Verify Workload Identity Federation is correctly configured
   - Check that the service account has necessary permissions
   - Ensure the provider resource name is correct

2. **Permission Denied (Artifact Registry)**
   - Verify the service account has `roles/artifactregistry.writer` role
   - Check that the Artifact Registry repository exists
   - Ensure the repository location matches the region parameter

3. **Docker Build Failed**
   - Check that your Dockerfile is valid and in the repository root
   - Verify all dependencies are available during build
   - Check for sufficient runner resources

4. **Push Failed**
   - Verify Docker is configured correctly for Artifact Registry
   - Check network connectivity to Artifact Registry
   - Ensure the image was built successfully

### Debugging Tips

1. **Enable Debug Logging**: Add this to your workflow:
   ```yaml
   env:
     ACTIONS_STEP_DEBUG: true
   ```

2. **Check Image URL**: The workflow outputs the image URL, verify it matches your expectations

3. **Test Locally**: You can test the Docker build locally:
   ```bash
   docker build -t test-image .
   ```

## Security Considerations

1. **Use Workload Identity Federation**: Never store service account keys in GitHub secrets
2. **Minimal Permissions**: Grant only necessary IAM roles to the service account
3. **Repository Access**: Limit which repositories can use this workflow
4. **Image Scanning**: Consider enabling vulnerability scanning in Artifact Registry

## Performance Tips

1. **Multi-stage Builds**: Use multi-stage Dockerfiles to reduce image size
2. **Docker Layer Caching**: GitHub Actions provides Docker layer caching
3. **Parallel Builds**: If you have multiple services, build them in parallel jobs
4. **Registry Cleanup**: Regularly clean up old images to reduce storage costs

## Related Documentation

- [Google Cloud Artifact Registry](https://cloud.google.com/artifact-registry/docs)
- [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation)
- [GitHub Actions Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)

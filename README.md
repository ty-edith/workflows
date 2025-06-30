# Build and Push Docker Image GitHub Action

A reusable GitHub Action workflow for building and pushing Docker images to Google Cloud Artifact Registry.

## Features

- 🐳 Build Docker images from your repository
- 🚀 Push images to Google Cloud Artifact Registry
- 🔐 Secure authentication using Workload Identity Federation
- 🏷️ Flexible tagging with custom tags or automatic SHA-based tags
- 📦 Configurable Artifact Registry repository

## Prerequisites

Before using this workflow, ensure you have:

1. **Google Cloud Project** with Artifact Registry enabled
2. **Artifact Registry Repository** created in your GCP project
3. **Workload Identity Pool and Provider** configured for GitHub Actions
4. **Service Account** with appropriate permissions:
   - `roles/artifactregistry.writer`
   - `roles/storage.admin` (if using Cloud Storage for layers)

## Setup

### 1. Configure Workload Identity Federation

Follow [Google's documentation](https://cloud.google.com/blog/products/identity-security/enabling-keyless-authentication-from-github-actions) to set up Workload Identity Federation for GitHub Actions.

### 2. Create Artifact Registry Repository

```bash
gcloud artifacts repositories create YOUR_REPOSITORY_NAME \
    --repository-format=docker \
    --location=YOUR_REGION \
    --project=YOUR_PROJECT_ID
```

## Usage

### Basic Usage

Create a workflow in your repository that calls this reusable workflow:

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-image:
    uses: your-org/build-and-push-image/.github/workflows/build-and-push-docker.yml@main
    with:
      project_id: "your-gcp-project-id"
      service_account: "github-actions@your-project.iam.gserviceaccount.com"
      workload_identity_provider: "projects/123456789/locations/global/workloadIdentityPools/github/providers/github"
      region: "us-central1"
```

### Advanced Usage with Custom Repository and Tag

```yaml
name: Build and Deploy

on:
  push:
    tags: ['v*']

jobs:
  build-image:
    uses: your-org/build-and-push-image/.github/workflows/build-and-push-docker.yml@main
    with:
      project_id: "your-gcp-project-id"
      service_account: "github-actions@your-project.iam.gserviceaccount.com"
      workload_identity_provider: "projects/123456789/locations/global/workloadIdentityPools/github/providers/github"
      region: "us-central1"
      repository: "my-custom-repository"
      tag: ${{ github.ref_name }}
  
  deploy:
    needs: build-image
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Cloud Run
        run: |
          echo "Deploying image: ${{ needs.build-image.outputs.image-url }}"
          # Add your deployment logic here
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `project_id` | GCP project ID | ✅ | - |
| `service_account` | GCP service account email for authentication | ✅ | - |
| `workload_identity_provider` | Full provider resource name | ✅ | - |
| `region` | GCP region for Artifact Registry | ✅ | - |
| `tag` | Custom tag for the Docker image | ❌ | SHA-based tag |
| `repository` | Artifact Registry repository name | ❌ | `cloud-run-source-deploy` |

## Outputs

| Output | Description |
|--------|-------------|
| `image-url` | The full URL of the built Docker image |

## Image Naming Convention

The workflow generates image URLs using the following pattern:

### With Custom Tag

```text
{region}-docker.pkg.dev/{project_id}/{repository}/{github.repository_owner}/{github.event.repository.name}:{tag}
```

### With SHA Tag (default)

```text
{region}-docker.pkg.dev/{project_id}/{repository}/{github.repository_owner}/{github.event.repository.name}/sha:{github.sha}
```

## Examples

### Example 1: Simple Build on Push

```yaml
name: Build on Push

on:
  push:
    branches: [main, develop]

jobs:
  build:
    uses: your-org/build-and-push-image/.github/workflows/build-and-push-docker.yml@main
    with:
      project_id: "my-project-123"
      service_account: "github-actions@my-project-123.iam.gserviceaccount.com"
      workload_identity_provider: "projects/123456789/locations/global/workloadIdentityPools/github/providers/github"
      region: "us-west1"
```

### Example 2: Release Build with Version Tag

```yaml
name: Release Build

on:
  release:
    types: [published]

jobs:
  build-and-push:
    uses: your-org/build-and-push-image/.github/workflows/build-and-push-docker.yml@main
    with:
      project_id: "my-project-123"
      service_account: "github-actions@my-project-123.iam.gserviceaccount.com"
      workload_identity_provider: "projects/123456789/locations/global/workloadIdentityPools/github/providers/github"
      region: "europe-west1"
      repository: "production-images"
      tag: ${{ github.event.release.tag_name }}
```

### Example 3: Multi-Environment Setup

```yaml
name: Multi-Environment Build

on:
  push:
    branches: [main, staging, develop]

jobs:
  build:
    uses: your-org/build-and-push-image/.github/workflows/build-and-push-docker.yml@main
    with:
      project_id: ${{ github.ref == 'refs/heads/main' && 'prod-project' || 'dev-project' }}
      service_account: ${{ github.ref == 'refs/heads/main' && 'prod-sa@prod-project.iam.gserviceaccount.com' || 'dev-sa@dev-project.iam.gserviceaccount.com' }}
      workload_identity_provider: "projects/123456789/locations/global/workloadIdentityPools/github/providers/github"
      region: "us-central1"
      repository: ${{ github.ref == 'refs/heads/main' && 'production' || 'development' }}
      tag: ${{ github.ref_name }}
```

## Troubleshooting

### Common Issues

1. **Authentication Failed**
   - Verify your Workload Identity Federation setup
   - Ensure the service account has the correct permissions
   - Check that the provider resource name is correct

2. **Permission Denied**
   - Verify the service account has `roles/artifactregistry.writer`
   - Ensure the Artifact Registry repository exists
   - Check that the repository is in the correct region

3. **Docker Build Failed**
   - Ensure your repository contains a valid `Dockerfile`
   - Check that all required build files are committed to the repository

4. **Invalid Image URL**
   - Verify the project ID, region, and repository name are correct
   - Check that special characters in repository names are properly handled

### Getting Help

- Check the [Google Cloud Artifact Registry documentation](https://cloud.google.com/artifact-registry/docs)
- Review [GitHub Actions Workload Identity documentation](https://cloud.google.com/blog/products/identity-security/enabling-keyless-authentication-from-github-actions)
- Examine the workflow run logs for detailed error messages

## Contributing

1. Fork this repository
2. Create a feature branch
3. Make your changes
4. Test with a sample project
5. Submit a pull request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

# Release Workflow

The `release.yml` workflow deploys applications to Google Cloud Run with support for database migrations and environment-specific configurations.

## Overview

This workflow provides a complete deployment solution for Cloud Run applications. It supports:

- Deploying Cloud Run services using declarative YAML templates
- Running database migration jobs before deployment
- Environment-specific configurations (test, production, etc.)
- YAML templating with `ytt` for flexible configuration management

## Inputs

| Parameter | Required | Type | Default | Description |
|-----------|----------|------|---------|-------------|
| `image_url` | ✅ | string | - | The Docker image URL to deploy |
| `environment` | ✅ | string | - | Deployment environment (test, production) |
| `migrate` | ❌ | boolean | `false` | Whether to run database migrations |
| `project_id` | ✅ | string | - | GCP project ID |
| `service_account` | ✅ | string | - | GCP service account for authentication |
| `workload_identity_provider` | ✅ | string | - | Full provider resource name |
| `region` | ✅ | string | - | GCP region |

## Prerequisites

### Repository Structure

Your repository must include the following YAML template files:

```
.github/
└── resources/
    ├── values.yaml                    # Base configuration values
    ├── service.yaml                   # Cloud Run service template
    ├── migration.job.yaml             # Migration job template (optional)
    └── env/
        ├── test.values.yaml          # Test environment values
        └── production.values.yaml    # Production environment values
```

### Template Files

#### 1. Base Values (`values.yaml`)

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

#### 2. Cloud Run Service Template (`service.yaml`)

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

#### 3. Migration Job Template (`migration.job.yaml`)

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
            command: ["python", "manage.py", "migrate"]
            resources:
              limits:
                memory: 512Mi
                cpu: 1
            env:
            - name: COMMIT_SHA
              value: #@ data.values.commit_sha
```

#### 4. Environment Values (`env/production.values.yaml`)

```yaml
#@data/values
---
service_name: my-app
memory: 1Gi
cpu: 2
min_instances: 1
max_instances: 50
timeout: 600
database_url: postgresql://prod-db-url
```

### Google Cloud Setup

1. **Service Account Permissions**: The service account needs these IAM roles:
   - `roles/run.admin`
   - `roles/iam.serviceAccountUser`

2. **APIs**: Enable the following APIs:
   - Cloud Run API
   - Cloud Resource Manager API

## Usage Examples

### Basic Deployment

```yaml
jobs:
  deploy:
    uses: ty-edith/workflows/.github/workflows/release.yml@v1
    with:
      image_url: "us-central1-docker.pkg.dev/my-project/repo/app:latest"
      environment: "production"
      project_id: ${{ secrets.GCP_PROJECT_ID }}
      service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
      workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      region: ${{ secrets.GCP_REGION }}
```

### Deployment with Migration

```yaml
jobs:
  deploy:
    uses: ty-edith/workflows/.github/workflows/release.yml@v1
    with:
      image_url: "us-central1-docker.pkg.dev/my-project/repo/app:latest"
      environment: "production"
      migrate: true
      project_id: ${{ secrets.GCP_PROJECT_ID }}
      service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
      workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      region: ${{ secrets.GCP_REGION }}
```

### Complete Pipeline with Build and Deploy

```yaml
name: Deploy to Production

on:
  release:
    types: [published]

jobs:
  build:
    uses: ty-edith/workflows/.github/workflows/build-and-push.yml@v1
    with:
      project_id: ${{ secrets.GCP_PROJECT_ID }}
      service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
      workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      region: ${{ secrets.GCP_REGION }}
      tag: ${{ github.event.release.tag_name }}

  deploy:
    needs: build
    uses: ty-edith/workflows/.github/workflows/release.yml@v1
    with:
      image_url: ${{ needs.build.outputs.image-url }}
      environment: "production"
      migrate: true
      project_id: ${{ secrets.GCP_PROJECT_ID }}
      service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
      workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      region: ${{ secrets.GCP_REGION }}
```

### Multi-Environment Deployment

```yaml
name: Deploy to Multiple Environments

on:
  push:
    branches: [main, develop]

jobs:
  build:
    uses: ty-edith/workflows/.github/workflows/build-and-push.yml@v1
    with:
      project_id: ${{ secrets.GCP_PROJECT_ID }}
      service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
      workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      region: ${{ secrets.GCP_REGION }}

  deploy-test:
    if: github.ref == 'refs/heads/develop'
    needs: build
    uses: ty-edith/workflows/.github/workflows/release.yml@v1
    with:
      image_url: ${{ needs.build.outputs.image-url }}
      environment: "test"
      migrate: true
      project_id: ${{ secrets.GCP_PROJECT_ID }}
      service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
      workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      region: ${{ secrets.GCP_REGION }}

  deploy-production:
    if: github.ref == 'refs/heads/main'
    needs: build
    uses: ty-edith/workflows/.github/workflows/release.yml@v1
    with:
      image_url: ${{ needs.build.outputs.image-url }}
      environment: "production"
      migrate: true
      project_id: ${{ secrets.GCP_PROJECT_ID }}
      service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
      workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
      region: ${{ secrets.GCP_REGION }}
```

## Workflow Steps

### Migration Steps (if `migrate: true`)

1. **Verify environment values file**: Checks that the environment-specific values file exists
2. **Install ytt**: Downloads and installs the `ytt` YAML templating tool
3. **Render migration job YAML**: Uses `ytt` to generate the migration job configuration
4. **Deploy migration job**: Replaces the existing migration job with the new configuration
5. **Execute migration job**: Runs the migration job and waits for completion

### Service Deployment Steps

1. **Render Cloud Run service YAML**: Uses `ytt` to generate the service configuration
2. **Deploy Cloud Run service**: Replaces the existing service with the new configuration
3. **Deployment summary**: Displays deployment information including service URL

## YAML Templating with ytt

The workflow uses [ytt](https://carvel.dev/ytt/) for YAML templating, which provides:

- **Data values**: Inject configuration values into templates
- **Functions**: Use built-in functions for string manipulation, conditionals, etc.
- **Overlays**: Modify existing YAML structures
- **Schema validation**: Validate configuration values

### ytt Features Used

- `#@ load("@ytt:data", "data")`: Load data values
- `#@ data.values.field_name`: Access configuration values
- `#@ str(value)`: Convert values to strings
- `#@data/values`: Mark files as data value sources

### Environment-Specific Configuration

The workflow merges configuration in this order:

1. Base values (`values.yaml`)
2. Environment-specific values (`env/{environment}.values.yaml`)
3. Runtime values (image_url, commit_sha, service_account)

This allows you to override base configuration for specific environments while keeping common settings in one place.

## Troubleshooting

### Common Issues

1. **Missing Environment Values File**
   ```
   Error: Missing .github/resources/env/production.values.yaml
   ```
   - Solution: Create the environment-specific values file

2. **ytt Template Errors**
   ```
   Error: ytt: Error: Evaluating library...
   ```
   - Solution: Check YAML template syntax and data value references

3. **Migration Job Failed**
   ```
   Error: Job execution failed
   ```
   - Solution: Check migration job logs in Cloud Console
   - Verify database connectivity and migration scripts

4. **Service Deployment Failed**
   ```
   Error: The request has errors
   ```
   - Solution: Check the rendered YAML for syntax errors
   - Verify service account permissions

### Debugging Tips

1. **Check Rendered YAML**: The workflow saves rendered YAML to `/tmp/`. You can add a step to output it:
   ```yaml
   - name: Debug rendered YAML
     run: cat /tmp/service.yaml
   ```

2. **Validate Templates Locally**: Install ytt locally and test templates:
   ```bash
   ytt -f .github/resources/service.yaml \
       --data-values-file .github/resources/values.yaml \
       --data-value image_url=test
   ```

3. **Check Cloud Run Logs**: View application logs in Cloud Console for runtime issues

## Security Considerations

1. **Service Account Permissions**: Use minimal required permissions
2. **Environment Separation**: Use different projects or service accounts for different environments
3. **Secret Management**: Store sensitive configuration in Cloud Secret Manager
4. **Network Security**: Configure VPC connectors and ingress controls as needed

## Best Practices

1. **Environment Parity**: Keep test and production configurations as similar as possible
2. **Configuration Management**: Use environment variables for runtime configuration
3. **Health Checks**: Implement proper health check endpoints
4. **Monitoring**: Set up logging and monitoring for your services
5. **Rollback Strategy**: Keep previous revisions for quick rollbacks

## Advanced Configuration

### Custom Health Checks

Add health checks to your service template:

```yaml
spec:
  template:
    spec:
      containers:
      - image: #@ data.values.image_url
        livenessProbe:
          httpGet:
            path: /health
            port: #@ data.values.port
          initialDelaySeconds: 30
          periodSeconds: 10
```

### VPC Connector

Add VPC connectivity for database access:

```yaml
metadata:
  annotations:
    run.googleapis.com/vpc-access-connector: projects/PROJECT/locations/REGION/connectors/CONNECTOR
```

### Custom Domains

Configure custom domains:

```yaml
metadata:
  annotations:
    run.googleapis.com/ingress: all
    run.googleapis.com/custom-audiences: "https://my-domain.com"
```

## Related Documentation

- [Google Cloud Run](https://cloud.google.com/run/docs)
- [ytt YAML Templating](https://carvel.dev/ytt/)
- [Cloud Run YAML Reference](https://cloud.google.com/run/docs/reference/yaml/v1)
- [GitHub Actions Reusable Workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows)

# Azure DevOps Build Containers

This directory contains Dockerfiles for Azure DevOps pipeline build containers.

## Dockerfile_azure_build

A build container with all tools pre-installed for building erwin-DM-Cloud backend services.

### Included Tools

| Tool | Version | Purpose |
|------|---------|---------|
| Java (Eclipse Temurin) | 21 | Compile Spring Boot services |
| Maven | 3.9.9 | Build automation |
| Docker CLI | Latest | Build and push container images |
| Azure CLI | Latest | ACR authentication, AKS deployments |
| kubectl | 1.31.x | Kubernetes deployments |
| Helm | Latest | Kubernetes package management |
| Git | Latest | Source control |

### Build the Image

#### Option 1: GitHub Actions (Automated)

1. Create a new GitHub repository from this folder
2. Add secrets in **Settings > Secrets and variables > Actions**:
   - `DOCKERHUB_USERNAME` - Your Docker Hub username
   - `DOCKERHUB_TOKEN` - Docker Hub access token ([create here](https://hub.docker.com/settings/security))
3. Push to `main` branch or run the workflow manually

The image will be pushed to: `<your-username>/azure-build:java21`

#### Option 2: Build Locally

```bash
# From repository root
cd azure-devops/docker

# Build locally
docker build -t azure-build:java21 -f Dockerfile_azure_build .

# Build and push to ACR
docker build -t <your-acr>.azurecr.io/azure-build:java21 -f Dockerfile_azure_build .
az acr login --name <your-acr>
docker push <your-acr>.azurecr.io/azure-build:java21
```

### Pipeline Usage

Reference the image in your pipeline parameters:

```yaml
parameters:
  - name: containerImage
    displayName: 'Container Image for Build'
    type: string
    default: '<your-acr>.azurecr.io/azure-build:java21'
```

Or when running the pipeline manually, specify the full image path:
- `questpmerdacr.azurecr.io/azure-build:java21`
- `questerdacr.azurecr.io/azure-build:java21`

### Why a Custom Build Container?

1. **Java 21 Requirement**: erwin-DM-Cloud requires Java 21. Public images often have older Java versions.
2. **No Runtime Installation**: Installing tools during pipeline runs is slow and can fail due to permissions.
3. **Consistency**: Same environment for all builds across all agents.
4. **Speed**: No `apt-get` calls during builds = faster pipelines.

### Troubleshooting

**Permission denied when running apt-get**
- This was the original issue. The solution is to use this pre-built image instead of trying to install tools at runtime.

**Docker commands fail**
- Ensure `mapDockerSocket: true` is set in the container configuration.
- The container uses the host's Docker daemon via socket mapping.

**Image not found**
- Verify the image is pushed to your ACR
- Verify the service connection has AcrPull permissions
- Use fully qualified image name: `<acr>.azurecr.io/azure-build:java21`

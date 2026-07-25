# Docker Images

This repository contains Dockerfiles organized in per-image folders.
A GitHub Actions workflow automatically builds and pushes an image to Docker Hub
**whenever its folder changes** on the `main` branch.

---

## Repository Structure

```
docker_images/
├── .github/
│   └── workflows/
│       └── docker-build-push.yml   ← automated CI/CD
├── azure-build/
│   └── Dockerfile                  ← Java 21 + Maven + Azure CLI + kubectl + Helm
├── <next-image>/
│   └── Dockerfile                  ← add new images here
└── README.md
```

### Naming convention

| Folder name | Docker Hub image |
|-------------|-----------------|
| `azure-build` | `<DOCKERHUB_USERNAME>/azure-build` |
| `node-build`  | `<DOCKERHUB_USERNAME>/node-build`  |
| `python-build`| `<DOCKERHUB_USERNAME>/python-build`|

The folder name becomes the Docker Hub repository name automatically.

---

## How the Workflow Works

1. You push a change to `main` that modifies one or more `Dockerfile` files.
2. The `detect-changes` job diffs `HEAD~1..HEAD` to find which **folders** contain changed Dockerfiles.
3. A parallel matrix job runs **only** for those folders — builds the image and pushes two tags:
   - `:latest`
   - `:<short-sha>` (e.g. `:a1b2c3d`) for traceability

Images in **unchanged** folders are not rebuilt.

---

## One-time Setup

### 1 — Add GitHub Secrets

In your GitHub repository go to **Settings → Secrets and variables → Actions** and add:

| Secret | Value |
|--------|-------|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token ([create here](https://hub.docker.com/settings/security)) |

### 2 — Push to `main`

Any push that modifies a `Dockerfile` inside a first-level folder triggers the workflow automatically.

### 3 — Manual trigger (optional)

Run the workflow from **Actions → Build and Push Docker Images → Run workflow**.

- Leave *folder* empty to build **all** images.
- Enter a folder name (e.g. `azure-build`) to build only that image.

---

## Adding a New Image

1. Create a new folder: `mkdir my-new-image`
2. Add a `Dockerfile` inside it.
3. Commit and push to `main`.

The workflow picks it up automatically — no changes to the workflow file needed.

---

## Images

### `azure-build`

Azure DevOps build container for erwin-DM-Cloud backend services.

| Tool | Version |
|------|---------|
| Java (Eclipse Temurin) | 21 |
| Maven | 3.9.9 |
| Docker CLI | Latest |
| Azure CLI | Latest |
| kubectl | 1.31.x |
| Helm | Latest |
| Git | Latest |

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

# Docker Images

This repository contains Dockerfiles organized by runner OS.
A GitHub Actions workflow automatically builds and pushes an image to Docker Hub
**whenever its folder changes** on the `main` branch.

---

## Repository Structure

```
docker_images/
├── .github/
│   └── workflows/
│       └── docker-build-push.yml        ← automated CI/CD
├── linux/                               ← built on ubuntu-latest
│   └── azure-build/
│       └── Dockerfile                   ← Java 21 + Maven + Azure CLI + kubectl + Helm
├── windows/                             ← built on windows-2025
│   └── azure-windows-build/
│       └── Dockerfile                   ← Java 21 + Maven + Azure CLI + kubectl + Helm (Windows)
└── README.md
```

### Runner mapping

| Top-level folder | GitHub Actions runner |
|------------------|-----------------------|
| `linux/`         | `ubuntu-latest`       |
| `windows/`       | `windows-2025`        |

### Docker Hub image naming

The **second-level folder name** becomes the Docker Hub repository name.

| Folder path | Docker Hub image |
|-------------|-----------------|
| `linux/azure-build` | `<DOCKERHUB_USERNAME>/azure-build` |
| `windows/azure-windows-build` | `<DOCKERHUB_USERNAME>/azure-windows-build` |

---

## How the Workflow Works

1. You push a change to `main` that modifies a `Dockerfile` under `linux/` or `windows/`.
2. The `detect-changes` job diffs `HEAD~1..HEAD`, extracts changed folders, and builds a matrix of `{ folder, runner, image }` entries.
3. A parallel matrix job runs **only** for those folders on the correct runner — builds the image and pushes two tags:
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

Any push that modifies a `Dockerfile` inside `linux/` or `windows/` triggers the workflow automatically.

### 3 — Manual trigger (optional)

Run the workflow from **Actions → Build and Push Docker Images → Run workflow**.

- Leave *folder* empty to build **all** images.
- Enter a folder path (e.g. `linux/azure-build` or `windows/azure-windows-build`) to build only that image.

---

## Adding a New Image

**Linux image:**
```
mkdir -p linux/my-new-image
# add linux/my-new-image/Dockerfile
git add . && git commit -m "add my-new-image" && git push
```

**Windows image:**
```
mkdir -p windows/my-new-image
# add windows/my-new-image/Dockerfile
git add . && git commit -m "add my-new-image" && git push
```

The workflow picks it up automatically — no changes to the workflow file needed.

---

## Images

### `linux/azure-build`

Azure DevOps build container for erwin-DM-Cloud backend services (Linux).

| Tool | Version |
|------|---------|
| Java (Eclipse Temurin) | 21 |
| Maven | 3.9.9 |
| Docker CLI | Latest |
| Azure CLI | Latest |
| kubectl | 1.31.x |
| Helm | Latest |
| Git | Latest |

### `windows/azure-windows-build`

Azure DevOps build container for erwin-DM-Cloud backend services (Windows Server Core ltsc2025).

| Tool | Version |
|------|---------|
| Java (Eclipse Temurin) | 21 |
| Maven | 3.9.9 |
| Docker CLI | Latest |
| Azure CLI | Latest |
| kubectl | 1.31.x |
| Helm | Latest |
| Git | Latest |
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

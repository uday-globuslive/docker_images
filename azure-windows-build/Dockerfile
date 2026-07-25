# Azure DevOps Build Container for erwin-DM-Cloud (Windows)
# Pre-installed: Java 21, Maven 3.9.x, Docker CLI, Azure CLI, Git, Chrome, kubectl, Helm
# Base: Windows Server Core ltsc2025 (matches process-isolation host requirement)
#
# Build:  docker build -t <your-registry>/azure-build:java21-win -f Dockerfile_azure_build_windows .
# Push:   docker push <your-registry>/azure-build:java21-win
#
# Pipeline usage:
#   pool: name: 'OnPrem-Windows'
#   container: '<your-registry>/azure-build:java21-win'
#
# NOTE: Build this on a host already matched to ltsc2025 (or the image
# won't run under process isolation on a different Windows build).

FROM mcr.microsoft.com/windows/servercore:ltsc2025

LABEL maintainer="erwin-DM-Cloud Team"
LABEL description="Azure DevOps build container with Java 21, Maven, Docker CLI, Azure CLI (Windows)"

SHELL ["powershell", "-NoProfile", "-ExecutionPolicy", "Bypass", "-Command"]

# ── Install Chocolatey (package manager, stands in for apt) ─────────────
RUN Set-ExecutionPolicy Bypass -Scope Process -Force; \
    [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; \
    Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1')); \
    choco --version

# ── Install Java 21 (Eclipse Temurin) ────────────────────────────────────
RUN choco install -y temurin21

# Temurin installs into a versioned folder (e.g. jdk-21.0.9.10-hotspot) that
# drifts with every patch update, so resolve it and pin a stable junction.
RUN $jdkPath = (Get-ChildItem 'C:\Program Files\Eclipse Adoptium' -Directory | Where-Object { $_.Name -like 'jdk-21*' } | Select-Object -First 1).FullName ; \
    cmd /c mklink /J "C:\java21" "$jdkPath"

ENV JAVA_HOME="C:\java21"
RUN [Environment]::SetEnvironmentVariable('PATH', $env:PATH + ';' + $env:JAVA_HOME + '\bin', 'Machine')

# ── Install Maven 3.9.x ───────────────────────────────────────────────────
ARG MAVEN_VERSION=3.9.9
RUN Invoke-WebRequest -Uri "https://archive.apache.org/dist/maven/maven-3/$env:MAVEN_VERSION/binaries/apache-maven-$env:MAVEN_VERSION-bin.zip" -OutFile C:\maven.zip ; \
    Expand-Archive -Path C:\maven.zip -DestinationPath 'C:\opt' ; \
    Rename-Item -Path "C:\opt\apache-maven-$env:MAVEN_VERSION" -NewName 'maven' ; \
    Remove-Item C:\maven.zip

ENV MAVEN_HOME="C:\opt\maven"
RUN [Environment]::SetEnvironmentVariable('PATH', $env:PATH + ';' + $env:MAVEN_HOME + '\bin', 'Machine')

# ── Install Git ────────────────────────────────────────────────────────
RUN choco install -y git --params "/GitAndUnixToolsOnPath"

# ── Install Docker CLI (for Docker-in-Docker via named pipe mapping) ────
RUN choco install -y docker-cli

# ── Install Azure CLI ─────────────────────────────────────────────────
RUN $ProgressPreference = 'SilentlyContinue'; \
    Invoke-WebRequest -Uri https://aka.ms/installazurecliwindows -OutFile .\AzureCLI.msi ; \
    Start-Process msiexec.exe -Wait -ArgumentList '/I AzureCLI.msi /quiet' ; \
    Remove-Item .\AzureCLI.msi

# ── Install Google Chrome (for Selenium) ──────────────────────────────
RUN choco install -y googlechrome --ignore-checksums

# ── Install kubectl ────────────────────────────────────────────────────
RUN $stable = (Invoke-WebRequest -Uri "https://dl.k8s.io/release/stable.txt" -UseBasicParsing).Content.Trim() ; \
    Invoke-WebRequest -Uri "https://dl.k8s.io/release/$stable/bin/windows/amd64/kubectl.exe" -OutFile 'C:\Windows\System32\kubectl.exe'

# ── Install Helm ───────────────────────────────────────────────────────
RUN choco install -y kubernetes-helm

# ── Install jq, unzip equivalents (7zip covers archive handling) ───────
RUN choco install -y jq 7zip

WORKDIR C:\\azp

# ── Verify all tools are installed correctly ─────────────────────────
RUN Write-Host '=== Build Container Tool Versions ===' ; \
    Write-Host 'Java:'; & "$env:JAVA_HOME\bin\java.exe" -version ; \
    Write-Host ''; Write-Host 'Maven:'; & "$env:MAVEN_HOME\bin\mvn.cmd" -version ; \
    Write-Host ''; Write-Host 'Docker CLI:'; docker --version ; \
    Write-Host ''; Write-Host 'Azure CLI:'; az version ; \
    Write-Host ''; Write-Host 'Google Chrome:'; & 'C:\Program Files\Google\Chrome\Application\chrome.exe' --version ; \
    Write-Host ''; Write-Host 'kubectl:'; kubectl version --client ; \
    Write-Host ''; Write-Host 'Helm:'; helm version ; \
    Write-Host ''; Write-Host 'Git:'; git --version

# Windows containers run as ContainerAdministrator by default; switch to the
# lower-privilege built-in account for parity with the Linux non-root setup
USER ContainerUser

# Default command (Azure DevOps overrides this)
CMD ["powershell"]
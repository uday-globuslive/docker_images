# Azure DevOps Build Container for erwin-DM-Cloud
# Pre-installed: Java 21, Maven 3.9.x, Docker CLI, Azure CLI, Git
# Usage: Build and push to your container registry, then reference in pipeline
#
# Build:  docker build -t <your-registry>/azure-build:java21 -f Dockerfile_azure_build .
# Push:   docker push <your-registry>/azure-build:java21
#
# Pipeline usage:
#   containerImage: '<your-registry>/azure-build:java21'

FROM eclipse-temurin:21-jdk-jammy

LABEL maintainer="erwin-DM-Cloud Team"
LABEL description="Azure DevOps build container with Java 21, Maven, Docker CLI, Azure CLI"

# Avoid prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive

# Switch to more reliable HTTPS mirrors and install dependencies
# Using mirrors.ubuntu.com which redirects to geographically close mirrors
RUN set -eux; \
    # Backup original sources.list
    cp /etc/apt/sources.list /etc/apt/sources.list.bak || true; \
    # Replace with HTTPS mirrors (more reliable in cloud environments)
    echo "deb https://mirrors.edge.kernel.org/ubuntu/ jammy main restricted universe multiverse" > /etc/apt/sources.list; \
    echo "deb https://mirrors.edge.kernel.org/ubuntu/ jammy-updates main restricted universe multiverse" >> /etc/apt/sources.list; \
    echo "deb https://mirrors.edge.kernel.org/ubuntu/ jammy-backports main restricted universe multiverse" >> /etc/apt/sources.list; \
    echo "deb https://mirrors.edge.kernel.org/ubuntu/ jammy-security main restricted universe multiverse" >> /etc/apt/sources.list; \
    # Clean and update with retries
    rm -rf /var/lib/apt/lists/*; \
    apt-get clean; \
    for i in 1 2 3; do \
        apt-get update && break || { echo "Retry $i..."; sleep 5; }; \
    done; \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        wget \
        gnupg \
        git \
        jq \
        unzip \
        zip \
    && rm -rf /var/lib/apt/lists/*

# Install Maven 3.9.x
ARG MAVEN_VERSION=3.9.9
ARG MAVEN_SHA512=a555254d6b53d267965a3404ecb14e53c3827c09c3b94b5678835887ab404556bfaf78dcfe03ba76fa2508649dca8531c74bca4d5846513522404d48e8c4ac8b
ENV MAVEN_HOME=/opt/maven
ENV PATH="${MAVEN_HOME}/bin:${PATH}"

RUN mkdir -p ${MAVEN_HOME} \
    && curl -fsSL https://archive.apache.org/dist/maven/maven-3/${MAVEN_VERSION}/binaries/apache-maven-${MAVEN_VERSION}-bin.tar.gz \
       -o /tmp/maven.tar.gz \
    && echo "${MAVEN_SHA512}  /tmp/maven.tar.gz" | sha512sum -c - \
    && tar -xzf /tmp/maven.tar.gz -C ${MAVEN_HOME} --strip-components=1 \
    && rm /tmp/maven.tar.gz \
    && mvn --version

# Install Docker CLI (for Docker-in-Docker via socket mapping)
RUN set -eux; \
    install -m 0755 -d /etc/apt/keyrings; \
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /tmp/docker.gpg; \
    gpg --dearmor -o /etc/apt/keyrings/docker.gpg < /tmp/docker.gpg; \
    rm /tmp/docker.gpg; \
    chmod a+r /etc/apt/keyrings/docker.gpg; \
    echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu jammy stable" > /etc/apt/sources.list.d/docker.list; \
    for i in 1 2 3; do apt-get update && break || { echo "Retry $i..."; sleep 5; }; done; \
    apt-get install -y --no-install-recommends docker-ce-cli docker-buildx-plugin; \
    rm -rf /var/lib/apt/lists/*; \
    docker --version

# Install Azure CLI (download and install directly, bypassing apt mirrors)
RUN set -eux; \
    # Install Azure CLI using Microsoft's install script with apt retries
    for i in 1 2 3; do \
        curl -sL https://aka.ms/InstallAzureCLIDeb | bash && break || { \
            echo "Azure CLI install attempt $i failed, retrying..."; \
            rm -rf /var/lib/apt/lists/*; \
            sleep 10; \
        }; \
    done; \
    rm -rf /var/lib/apt/lists/*; \
    az version

# Install Google Chrome (official Google repository)
RUN set -eux; \
    install -m 0755 -d /etc/apt/keyrings; \
    curl -fsSL https://dl.google.com/linux/linux_signing_key.pub -o /tmp/google-linux-signing-key.pub; \
    gpg --dearmor -o /etc/apt/keyrings/google-chrome.gpg < /tmp/google-linux-signing-key.pub; \
    rm /tmp/google-linux-signing-key.pub; \
    chmod a+r /etc/apt/keyrings/google-chrome.gpg; \
    echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/google-chrome.gpg] https://dl.google.com/linux/chrome/deb/ stable main" > /etc/apt/sources.list.d/google-chrome.list; \
    for i in 1 2 3; do apt-get update && break || { echo "Retry $i..."; sleep 5; }; done; \
    apt-get install -y --no-install-recommends google-chrome-stable; \
    rm -rf /var/lib/apt/lists/*; \
    google-chrome --version

# Install kubectl for AKS deployments (using direct binary download for reliability)
RUN curl -fsSL "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" -o /usr/local/bin/kubectl \
    && chmod +x /usr/local/bin/kubectl \
    && kubectl version --client

# Install Helm for Kubernetes package management
RUN curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash \
    && helm version

# Set JAVA_HOME environment variable
ENV JAVA_HOME=/opt/java/openjdk
ENV PATH="${JAVA_HOME}/bin:${PATH}"

# Create non-root user for Azure DevOps agent compatibility
# Azure DevOps container jobs run as root by default, but we ensure flexibility
RUN groupadd -r azureagent && useradd -r -g azureagent -G docker azureagent 2>/dev/null || true

# Set working directory
WORKDIR /azp

# Verify all tools are installed correctly
RUN echo "=== Build Container Tool Versions ===" \
    && echo "Java:" && java -version \
    && echo "" && echo "Maven:" && mvn -version \
    && echo "" && echo "Docker CLI:" && docker --version \
    && echo "" && echo "Azure CLI:" && az version \
    && echo "" && echo "Google Chrome:" && google-chrome --version \
    && echo "" && echo "kubectl:" && kubectl version --client \
    && echo "" && echo "Helm:" && helm version \
    && echo "" && echo "Git:" && git --version

# Default command (Azure DevOps overrides this)
CMD ["/bin/bash"]

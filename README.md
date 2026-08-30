# HTML Docker Deployment Test

A minimal static HTML project for validating a self-hosted CI/CD pipeline using Jenkins, Harbor, Docker Compose, and a homelab-style infrastructure layout.

---

## Overview

This project is used to test the following deployment flow:

```text
GitHub -> Jenkins -> Docker Build -> Harbor -> Deployment Server -> Docker Compose
```

The application itself is intentionally simple: a static HTML page served by Nginx inside a Docker container.

---

## Infrastructure Architecture

```mermaid
flowchart TB
    subgraph EDGE[Edge Layer]
        INTERNET[Internet]
        FW[Firewall / Router]
        RP[Reverse Proxy]
    end

    subgraph VIRTUALIZATION[Virtualization Layer]
        PROXMOX[Proxmox Host]
    end

    subgraph CI[CI/CD Layer]
        GH[GitHub]
        JENKINS[Jenkins]
        HARBOR[Harbor Registry]
        IMAGE[Docker Image]
    end

    subgraph COMPUTE[Compute Layer]
        DEPLOY[Deployment Server]
        CONTAINERS[Docker Containers]
    end

    subgraph STORAGE[Storage Layer]
        NAS[NAS]
        NFS[NFS Mounts]
    end

    INTERNET --> FW
    FW --> RP
    RP --> DEPLOY

    PROXMOX --> JENKINS
    PROXMOX --> HARBOR
    PROXMOX --> DEPLOY

    GH -->|source code| JENKINS
    JENKINS -->|build image| IMAGE
    IMAGE -->|push image| HARBOR
    JENKINS -->|SSH deploy| DEPLOY

    HARBOR -->|store artifacts| NFS
    NFS --> NAS

    DEPLOY -->|pull image| HARBOR
    DEPLOY -->|run| CONTAINERS
    CONTAINERS -->|persistent data| NFS

    classDef edge fill:#fee2e2,stroke:#dc2626,stroke-width:2px,color:#111827;
    classDef virt fill:#e0e7ff,stroke:#4f46e5,stroke-width:2px,color:#111827;
    classDef cicd fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#111827;
    classDef compute fill:#dbeafe,stroke:#2563eb,stroke-width:2px,color:#111827;
    classDef storage fill:#fef3c7,stroke:#d97706,stroke-width:2px,color:#111827;

    class INTERNET,FW,RP edge;
    class PROXMOX virt;
    class GH,JENKINS,HARBOR,IMAGE cicd;
    class DEPLOY,CONTAINERS compute;
    class NAS,NFS storage;

    %% Edge / public traffic: red
    linkStyle 0 stroke:#dc2626,stroke-width:3px;
    linkStyle 1 stroke:#dc2626,stroke-width:3px;
    linkStyle 2 stroke:#dc2626,stroke-width:3px;

    %% Proxmox / virtualization placement: purple
    linkStyle 3 stroke:#7c3aed,stroke-width:2px,stroke-dasharray: 5 5;
    linkStyle 4 stroke:#7c3aed,stroke-width:2px,stroke-dasharray: 5 5;
    linkStyle 5 stroke:#7c3aed,stroke-width:2px,stroke-dasharray: 5 5;

    %% CI/CD flow: green
    linkStyle 6 stroke:#16a34a,stroke-width:3px;
    linkStyle 7 stroke:#16a34a,stroke-width:3px;
    linkStyle 8 stroke:#16a34a,stroke-width:3px;
    linkStyle 9 stroke:#16a34a,stroke-width:3px;

    %% Storage flow: orange
    linkStyle 10 stroke:#d97706,stroke-width:3px;
    linkStyle 11 stroke:#d97706,stroke-width:3px;

    %% Runtime / deployment flow: blue
    linkStyle 12 stroke:#2563eb,stroke-width:3px;
    linkStyle 13 stroke:#2563eb,stroke-width:3px;

    %% Persistent data flow: orange dashed
    linkStyle 14 stroke:#d97706,stroke-width:3px,stroke-dasharray: 5 5;
```

### Legend

| Color | Meaning |
| --- | --- |
| Red | Public / edge traffic |
| Purple dashed | Proxmox virtualization placement |
| Green | CI/CD build and deployment commands |
| Blue | Runtime container deployment |
| Orange | NAS / NFS storage flow |
| Orange dashed | Optional persistent application data |

---

## Security & CI/CD Architecture

```mermaid
flowchart LR
    INTERNET[Internet] --> FW[Firewall]
    GH[GitHub] -. source checkout .-> JENKINS[Jenkins]

    FW -->|allow 80/443 only| RP[Reverse Proxy with WAF]

    subgraph INTERNAL[Internal LAN]
        RP
        JENKINS
        HARBOR[Harbor Registry]
        DEPLOY[Deployment Server]
    end

    RP -->|proxy pass| DEPLOY

    JENKINS -->|build & push image| HARBOR
    JENKINS -->|SSH deploy| DEPLOY
    DEPLOY -->|pull image| HARBOR
```

---

## CI/CD Flow

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub
    participant Jenkins as Jenkins Build Server
    participant Harbor as Harbor Registry
    participant Prod as Deployment Server
    participant Docker as Docker Compose

    Dev->>GH: Push source code
    Jenkins->>GH: Checkout repository
    Jenkins->>Jenkins: Build Docker image
    Jenkins->>Harbor: Push image tags
    Jenkins->>Prod: SSH deploy command
    Prod->>Harbor: Pull selected image tag
    Prod->>Docker: docker compose up -d
```

---

## Image

```text
reg.yangdongi.com/library/test:<build-number>
reg.yangdongi.com/library/test:latest
```

The build-number tag is used for traceable deployments and rollback.  
The `latest` tag points to the most recent successful build.

---

## Deployment

The service is deployed with Docker Compose:

```yaml
services:
  test:
    image: reg.yangdongi.com/library/test:${IMAGE_TAG:-latest}
    container_name: test
    restart: unless-stopped
    ports:
      - "${APP_PORT:-8080}:80"
```

Manual deployment example:

```bash
IMAGE_TAG=latest APP_PORT=8080 docker compose pull
IMAGE_TAG=latest APP_PORT=8080 docker compose up -d
```

---

## Pipeline Steps

1. Jenkins checks out the repository from GitHub.
2. Jenkins builds the Docker image.
3. Jenkins tags the image with both the Jenkins build number and `latest`.
4. Jenkins pushes the image to Harbor.
5. Jenkins connects to the deployment server over SSH.
6. The deployment server pulls the selected image tag.
7. Docker Compose recreates the running container.

---

## Notes

- Only the reverse proxy is exposed through the firewall.
- Jenkins, Harbor, deployment servers, and NAS remain inside the internal LAN.
- Harbor stores image artifacts on NAS-backed NFS storage.
- Deployment servers pull images from Harbor and run services with Docker Compose.
- Jenkins URLs, credentials, SSH keys, private IPs, and internal server details are intentionally not exposed in this repository.
- Harbor credentials and SSH keys are managed outside of the repository.
- Build-number tags are used for rollback-friendly deployments.
- This repository is intended as a lightweight CI/CD pipeline validation example.
combined-infra-readme.md
7KB

# HTML Docker Deployment Test

A minimal static HTML project for validating a self-hosted CI/CD pipeline using Jenkins, Harbor, and Docker Compose.

## Pipeline

```text
GitHub -> Jenkins -> Docker Build -> Harbor -> Deployment Server -> Docker Compose
```

## Image

```text
reg.yangdongi.com/library/test:<build-number>
reg.yangdongi.com/library/test:latest
```

The build-number tag is used for traceable deployments and rollback.  
The `latest` tag points to the most recent successful build.

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

## CI/CD Flow

1. Jenkins checks out the repository from GitHub.
2. Jenkins builds the Docker image.
3. Jenkins tags the image with both the Jenkins build number and `latest`.
4. Jenkins pushes the image to Harbor.
5. Jenkins connects to the deployment server over SSH.
6. The deployment server pulls the selected image tag.
7. Docker Compose recreates the running container.

## Notes

- Jenkins URLs, credentials, and server details are intentionally not exposed in this repository.
- Harbor credentials and SSH keys are managed outside of the repository.
- Build-number tags are used for rollback-friendly deployments.
- This repository is intended as a lightweight CI/CD pipeline validation example.

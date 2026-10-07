# Submission — Movie Picture CI/CD Pipelines

## Overview

This project implements four GitHub Actions workflows that automate the full lifecycle of the Movie Picture application — a React frontend and Flask backend. The pipelines enforce code quality on every pull request and handle building, publishing, and deploying container images to an Amazon EKS cluster on every merge to main.

## Workflow Files

| Filename | Workflow name | Trigger |
|---|---|---|
| `frontend-ci.yaml` | Frontend PR Pipeline | PRs to `main` affecting `starter/frontend/**`, plus manual |
| `backend-ci.yaml` | Backend PR Pipeline | PRs to `main` affecting `starter/backend/**`, plus manual |
| `frontend-cd.yaml` | Frontend Release Pipeline | Pushes to `main` affecting `starter/frontend/**`, plus manual |
| `backend-cd.yaml` | Backend Release Pipeline | Pushes to `main` affecting `starter/backend/**`, plus manual |

## Pipeline Structure

### CI (Pull Request) Pipelines

Both CI pipelines follow the same pattern:

1. **`code-lint`** and **`unit-tests`** execute concurrently — ESLint + Jest for the frontend, flake8 + pytest for the backend. Running them in parallel cuts total wait time.
2. **`docker-verify`** only runs if both gates pass. It builds the Docker image locally without pushing, confirming the Dockerfile and application compile correctly.

### CD (Release) Pipelines

The CD pipelines extend the CI checks with two additional stages:

3. **`container-publish`** — Builds the Docker image, tags it with the git commit SHA (`${{ github.sha }}`), and pushes it to the corresponding ECR repository. For the frontend, this step first queries the EKS cluster to discover the backend's LoadBalancer URL and injects it via the `REACT_APP_MOVIE_API_URL` build arg.
4. **`k8s-release`** — Uses kustomize to patch the Kubernetes manifests with the newly pushed image URI, applies them to the EKS cluster, and waits for the rollout to succeed.

Each CD pipeline ends with reviewer-requested verification steps that log `kubectl get all`, `kubectl describe deploy`, and the latest ECR image metadata.

## Design Decisions

- **Parallel quality gates**: Lint and tests have no `needs` dependency on each other, so they run simultaneously.
- **Sequential gating**: The Docker build/push job declares `needs: [code-lint, unit-tests]`, preventing any image work if either check fails.
- **Commit SHA tagging**: Every Docker image is tagged `<ecr-registry>/<repo>:<git-sha>`, creating a direct, auditable link between a deployed container and the exact source commit.
- **Backend-first deployment order**: The frontend CD discovers the backend service URL during its build step, so the backend must be deployed before the frontend.
- **Secrets management**: AWS credentials are stored in GitHub repository secrets — `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and optionally `AWS_SESSION_TOKEN` — and are never hardcoded.

## Infrastructure

| Resource | Details |
|---|---|
| Container Registry | Amazon ECR — two repos (`frontend`, `backend`) |
| Orchestration | Amazon EKS cluster in `us-east-1` |
| IaC | Terraform configs under `setup/terraform/` |
| Manifest tooling | kustomize patches the image tag per deploy |

## Screenshots

Screenshots documenting successful workflow runs, ECR images, and the live application are in the `screenshots/` directory.

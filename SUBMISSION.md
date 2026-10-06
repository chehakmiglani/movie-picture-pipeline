# Submission — Movie Picture Pipeline

## What I Built

For this project, I set up four GitHub Actions pipelines that handle the full lifecycle of a two-part web app (React frontend + Flask backend). The goal was to go from code change → automated checks → container image → live deployment on Kubernetes, all without any manual steps once code lands on `main`.

## Pipeline Breakdown

### CI Pipelines (Pull Request Gates)

Each service has its own CI workflow that kicks in whenever a PR targets `main` and touches files under its directory. They can also be started by hand from the Actions tab.

- **Frontend CI** (`frontend-ci.yaml`): Spins up ESLint and Jest in parallel. If both are clean, a Docker build runs as a sanity check (nothing gets pushed anywhere).
- **Backend CI** (`backend-ci.yaml`): Runs flake8 for style and pytest for correctness, again in parallel. Same idea — Docker build only happens after both pass.

The parallel setup saves a noticeable amount of time compared to running things one after another.

### CD Pipelines (Deploy on Merge)

When code actually merges into `main`, the CD workflows take over. They repeat the same quality checks (because you can't trust that the PR checks ran against the final merged state), then go further:

1. Build a Docker image tagged with the exact commit SHA
2. Push it to the matching ECR repository
3. Use kustomize to patch the Kubernetes manifests with the new image tag
4. Apply the updated manifests to the EKS cluster via kubectl

The frontend CD has one extra step: before building the image, it queries the cluster to find the backend service's load balancer URL and passes it as `REACT_APP_MOVIE_API_URL` so the React app knows where to send API requests.

## Secrets & Security

All AWS credentials are stored as GitHub repository secrets — nothing is hardcoded in the workflow files. The workflows reference three secrets:

| Secret | Purpose |
|--------|---------|
| `AWS_ACCESS_KEY_ID` | IAM access key for ECR/EKS operations |
| `AWS_SECRET_ACCESS_KEY` | Corresponding secret key |
| `AWS_SESSION_TOKEN` | Required for temporary/session-based credentials |

## Infrastructure Stack

- **Container registry**: Amazon ECR (separate repos for `frontend` and `backend`)
- **Orchestration**: Amazon EKS cluster in `us-east-1`
- **Provisioning**: Terraform (configs live in `setup/terraform/`)
- **Manifest management**: kustomize for patching image tags at deploy time

## Architectural Decisions

1. **Commit SHA as image tag** — Every deployed image maps directly to a Git commit, which makes rollbacks and debugging straightforward.
2. **Gated builds** — Docker work is expensive, so I made it depend on lint+test passing first. No point building an image from broken code.
3. **Separate CI and CD files** — Keeps the PR feedback loop fast (no AWS credentials needed) while the deploy pipeline handles the heavier lifting.
4. **kustomize over manual sed** — Using `kustomize edit set image` is cleaner and less error-prone than string replacement in YAML files.

## Screenshots

| # | Filename | Description |
|---|----------|-------------|
| 1 | `01-frontend-movie-list.jpeg` | Frontend rendering movie data from the backend |
| 2 | `02-backend-movies-json.jpeg` | Raw JSON from the `/movies` API endpoint |
| 3 | `03-all-workflows-green.png` | All four workflows showing successful runs |
| 4 | `04-ecr-backend-image.jpeg` | Backend image in ECR with commit SHA tag |
| 5 | `05-ecr-frontend-image.jpeg` | Frontend image in ECR with commit SHA tag |
| 6 | `06-failing-test-blocks-build.jpeg` | Broken test correctly preventing the build job |

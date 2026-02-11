# cicd-template-nodejs-gcr

# Reusable CI/CD Workflows Template (GitHub Actions → Docker Hub → Google Cloud Run)

Use this file as a reference when you want to set up the same CI/CD pattern in another Node.js project. Copy the workflows below into the new repo under `.github/workflows/` and adjust the variables/secrets as described.

---

## 1. CI Workflow (Pull Requests to `staging` and `main`)

Save as: `.github/workflows/ci-pipeline.yml`

```yaml
name: CI Pipeline to staging/production environment

on:
  pull_request:
    branches:
      - staging
      - main

jobs:
  test:
    runs-on: ubuntu-latest
    name: Setup, test, and build project
    env:
      PORT: 5001 # change if your app uses a different port
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Install dependencies
        run: npm ci

      - name: Test application
        run: npm test

      - name: Build application
        run: |
          echo "Run command to build the application if present"
          npm run build --if-present
```

---

## 2. CD Workflow (Deploy to Google Cloud Run: Staging + Production)

Save as: `.github/workflows/cd-pipeline.yml`

```yaml
name: CD Pipeline to Google Cloud Run (staging and production)

on:
  push:
    branches:
      - staging
  workflow_dispatch: {}
  release:
    types: published

env:
  PORT: 5001
  # vars.IMAGE must be set in GitHub repo variables, e.g. "dockerhub-username/app-name"
  IMAGE: ${{ vars.IMAGE }}:${{ github.sha }}

jobs:
  test:
    runs-on: ubuntu-latest
    name: Setup, test, and build project
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Install dependencies
        run: npm ci

      - name: Test application
        run: npm test

  build:
    needs: test
    runs-on: ubuntu-latest
    name: Setup project, Authorize GitHub Actions to GCP and Docker Hub, and deploy
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Authenticate for GCP
        id: gcp-auth
        uses: google-github-actions/auth@v0
        with:
          credentials_json: ${{ secrets.GCP_SERVICE_ACCOUNT }}

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v0

      - name: Authenticate for Docker Hub
        id: docker-auth
        env:
          D_USER: ${{ secrets.DOCKER_USER }}
          D_PASS: ${{ secrets.DOCKER_PASSWORD }}
        run: |
          docker login -u "$D_USER" -p "$D_PASS"

      - name: Build and tag Image
        run: |
          docker build -t "${{ env.IMAGE }}" .

      - name: Push the image to Docker hub
        run: |
          docker push "${{ env.IMAGE }}"

      - name: Enable the Billing API
        run: |
          gcloud services enable cloudbilling.googleapis.com --project="${{ secrets.GCP_PROJECT_ID }}"

      - name: Deploy to GCP Run - Production environment (If a new release was published from the main branch)
        if: github.event_name == 'release' && github.event.action == 'published' && github.event.release.target_commitish == 'main'
        run: |
          gcloud run deploy "${{ vars.GCR_PROJECT_NAME }}" \
            --region "${{ vars.GCR_REGION }}" \
            --image "${{ env.IMAGE }}" \
            --platform "managed" \
            --allow-unauthenticated \
            --tag production

      - name: Deploy to GCP Run - Staging environment
        if: github.ref != 'refs/heads/main'
        run: |
          echo "Deploying to staging environment"
          gcloud run deploy "${{ vars.GCR_STAGING_PROJECT_NAME }}" \
            --region "${{ vars.GCR_REGION }}" \
            --image "${{ env.IMAGE }}" \
            --platform "managed" \
            --allow-unauthenticated \
            --tag staging
```

---

## 3. GitHub Configuration Checklist (for any new project)

In each new repository that uses these workflows, configure:

### Repository Variables (Settings → Actions → Variables)

- `IMAGE` → Docker Hub image name, e.g. `dockerhub-username/app-name`.
- `GCR_REGION` → Cloud Run region, e.g. `us-central1`.
- `GCR_PROJECT_NAME` → Cloud Run service name for production.
- `GCR_STAGING_PROJECT_NAME` → Cloud Run service name for staging.

### Repository Secrets (Settings → Actions → Secrets)

- `GCP_SERVICE_ACCOUNT` → JSON key of a GCP service account with:
  - Cloud Run Admin
  - Service Account User
  - Artifact/Container Registry write access (if needed)
  - Permission to enable or call Billing API (if you keep that step).
- `GCP_PROJECT_ID` → Your Google Cloud project ID.
- `DOCKER_USER` → Docker Hub username.
- `DOCKER_PASSWORD` → Docker Hub access token/password.

---

## 4. Google Cloud Configuration Checklist

For the GCP project referenced by `GCP_PROJECT_ID`:

- Enable APIs:
  - Cloud Run API
  - Cloud Build API (if used)
  - Artifact Registry or Container Registry (if storing images there)
  - Cloud Billing API (if you keep the explicit enable step)

- Create a service account for GitHub Actions (for example, `github-actions-deployer`) and grant it roles such as:
  - `roles/run.admin`
  - `roles/iam.serviceAccountUser`
  - `roles/artifactregistry.writer` or equivalent (if pushing to GCP registry)

- Decide and document:
  - Region (`GCR_REGION`), e.g. `us-central1`.
  - Cloud Run service name for production (`GCR_PROJECT_NAME`).
  - Cloud Run service name for staging (`GCR_STAGING_PROJECT_NAME`).

With this file, you can quickly copy the YAML blocks into a new project and then follow the checklists to wire GitHub and Google Cloud correctly.

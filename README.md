# CI/CD Templates

Reusable GitHub Actions workflows for self-hosted runners.

## Available Workflows

### 1. Flutter + Fastlane (`flutter-fastlane.yml`)
iOS → TestFlight + Android → Google Play via Fastlane on self-hosted macOS runner.

### 2. NestJS + AWS EB (`nestjs-aws-eb.yml`)
Lint → Test → Build → Deploy to AWS Elastic Beanstalk.

---

## Usage

### Flutter + Fastlane

Create `.github/workflows/ci.yml` in your Flutter project:

```yaml
name: CI/CD

on:
  push:
    branches: [main, develop]
  workflow_dispatch:
    inputs:
      platform:
        type: choice
        options: [all, ios, android]
        default: all
      flavor:
        type: choice
        options: [all, dev, prod]
        default: all

jobs:
  deploy:
    uses: MonsterDKing/ci-templates/.github/workflows/flutter-fastlane.yml@main
    with:
      platform: ${{ github.event.inputs.platform || 'all' }}
      flavor: ${{ github.event.inputs.flavor || 'all' }}
    secrets: inherit
```

**Required secrets** (set in repo Settings → Secrets):
| Secret | Description |
|--------|-------------|
| `APP_STORE_CONNECT_API_KEY_P8` | App Store Connect API key (.p8 content) |
| `ASC_KEY_ID` | API Key ID |
| `ASC_ISSUER_ID` | Issuer ID |
| `ANDROID_KEYSTORE_BASE64` | `base64 -i android/key.jks` |
| `ANDROID_KEY_PROPERTIES` | Content of `android/key.properties` |
| `GOOGLE_PLAY_JSON_KEY` | Google Play service account JSON |

**Required Fastlane lanes** in your project:
- iOS: `deploy_dev_testflight`, `deploy_prod_testflight`
- Android: `ci_cd_dev`, `ci_cd_prod`

---

### NestJS + AWS Elastic Beanstalk

Create `.github/workflows/ci.yml` in your NestJS project:

```yaml
name: CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  ci-cd:
    uses: MonsterDKing/ci-templates/.github/workflows/nestjs-aws-eb.yml@main
    with:
      node-version: "20.x"
      run-e2e: true
      deploy-staging: ${{ github.ref == 'refs/heads/develop' }}
      deploy-production: ${{ github.ref == 'refs/heads/main' }}
    secrets: inherit
```

**Required secrets:**
| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | AWS access key |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key |
| `EB_APPLICATION_NAME` | Elastic Beanstalk app name |
| `EB_STAGING_ENV_NAME` | EB staging environment |
| `EB_PRODUCTION_ENV_NAME` | EB production environment |
| `S3_DEPLOY_BUCKET` | S3 bucket for deploy artifacts |
| `DATABASE_URL` | Production DB URL |
| `STAGING_DATABASE_URL` | Staging DB URL |

---

## Self-Hosted Runner Setup

Both workflows expect a macOS self-hosted runner. Setup:

```bash
# Download runner
mkdir ~/actions-runner && cd ~/actions-runner
curl -o runner.tar.gz -L https://github.com/actions/runner/releases/latest/download/actions-runner-osx-arm64-2.332.0.tar.gz
tar xzf runner.tar.gz

# Configure (per repo)
./config.sh --url https://github.com/USER/REPO --token TOKEN

# Run
nohup ./run.sh > runner.log 2>&1 &
```

## Fastlane Templates

See `fastlane/` directory for starter Fastfile templates compatible with these workflows.

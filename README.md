# CI/CD Templates

Reusable GitHub Actions workflows for self-hosted runners.

## Available Workflows

### 1. Flutter Deploy (`flutter-deploy.yml`) ⭐ Recommended
Stateless iOS + Android deployment. Creates temporary keychain per build — zero dependency on host machine state.

### 2. Flutter + Fastlane (`flutter-fastlane.yml`) — Legacy
Older approach that depends on host keychain and pre-installed profiles.

### 3. NestJS + AWS EB (`nestjs-aws-eb.yml`)
Lint → Test → Build → Deploy to AWS Elastic Beanstalk.

---

## Flutter Deploy (Recommended)

### How it works

```
1. Creates temporary keychain (no user password needed)
2. Imports P12 certificate from secrets
3. Installs provisioning profile from secrets
4. flutter build ipa / appbundle
5. Uploads to TestFlight / Google Play via API
6. Deletes temporary keychain and all signing files
```

### Usage

Create `.github/workflows/deploy.yml` in your Flutter project:

```yaml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    uses: MonsterDKing/ci-templates/.github/workflows/flutter-deploy.yml@main
    with:
      ios_bundle_id: "com.example.app"
      android_package_name: "com.example.app"
      # flavor: "prod"                    # Optional
      # entry_point: "lib/main_prod.dart" # Optional
      # android_track: "internal"         # internal, alpha, beta, production
      # build_number_floor: 70            # Minimum build number
    secrets:
      APPLE_CERTIFICATE_P12: ${{ secrets.APPLE_CERTIFICATE_P12 }}
      APPLE_CERTIFICATE_PASSWORD: ${{ secrets.APPLE_CERTIFICATE_PASSWORD }}
      PROVISIONING_PROFILE: ${{ secrets.PROVISIONING_PROFILE }}
      ASC_KEY_P8: ${{ secrets.ASC_KEY_P8 }}
      ASC_KEY_ID: ${{ secrets.ASC_KEY_ID }}
      ASC_ISSUER_ID: ${{ secrets.ASC_ISSUER_ID }}
      ANDROID_KEYSTORE_BASE64: ${{ secrets.ANDROID_KEYSTORE_BASE64 }}
      ANDROID_KEY_PROPERTIES: ${{ secrets.ANDROID_KEY_PROPERTIES }}
      GOOGLE_PLAY_JSON_KEY: ${{ secrets.GOOGLE_PLAY_JSON_KEY }}
```

### Required Secrets

| Secret | How to get it |
|--------|---------------|
| `APPLE_CERTIFICATE_P12` | Keychain Access → Export distribution cert as .p12 → `base64 -i cert.p12` |
| `APPLE_CERTIFICATE_PASSWORD` | Password you set when exporting the .p12 |
| `PROVISIONING_PROFILE` | Apple Developer Portal → Download .mobileprovision → `base64 -i profile.mobileprovision` |
| `ASC_KEY_P8` | App Store Connect → Users & Access → Keys → Download .p8 |
| `ASC_KEY_ID` | Shown when creating the API key |
| `ASC_ISSUER_ID` | Shown at top of Keys page |
| `ANDROID_KEYSTORE_BASE64` | `base64 -i release-key.jks` |
| `ANDROID_KEY_PROPERTIES` | Content of key.properties file |
| `GOOGLE_PLAY_JSON_KEY` | Google Cloud → Service Account → JSON key |

### Required Fastlane files in your project

The template needs `next_build_number` lanes. Minimal setup:

**`ios/Gemfile`** and **`android/Gemfile`**:
```ruby
source "https://rubygems.org"
gem "fastlane"
```

**`ios/fastlane/Fastfile`**:
```ruby
default_platform(:ios)
platform :ios do
  lane :next_build_number do
    api_key = app_store_connect_api_key(
      key_id: ENV["ASC_KEY_ID"],
      issuer_id: ENV["ASC_ISSUER_ID"],
      key_filepath: ENV["ASC_KEY_FILEPATH"]
    )
    current = latest_testflight_build_number(api_key: api_key, app_identifier: "YOUR_BUNDLE_ID")
    UI.message("Current: #{current} → Next: #{current + 1}")
    current + 1
  end
end
```

**`android/fastlane/Fastfile`**:
```ruby
default_platform(:android)
platform :android do
  lane :next_build_number do
    codes = google_play_track_version_codes(track: 'internal', json_key: ENV["SUPPLY_JSON_KEY"], package_name: "YOUR_PACKAGE")
    current = codes.max || 0
    UI.message("Current: #{current} → Next: #{current + 1}")
    current + 1
  end
end
```

### Inputs Reference

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `flutter_version` | string | `stable` | FVM version or channel |
| `ios_bundle_id` | string | **required** | iOS bundle identifier |
| `android_package_name` | string | **required** | Android package name |
| `flavor` | string | `""` | App flavor (empty = no flavor) |
| `entry_point` | string | `""` | Dart entry point (empty = lib/main.dart) |
| `deploy_ios` | boolean | `true` | Deploy to TestFlight |
| `deploy_android` | boolean | `true` | Deploy to Google Play |
| `android_track` | string | `internal` | Play Store track |
| `timeout_minutes` | number | `90` | Job timeout |
| `build_number_floor` | number | `1` | Minimum build number |

---

## Self-Hosted Runner Setup

See full documentation at `/Volumes/NVME_FAST/dev/runners-docs/README.md`

Quick setup:
```bash
mkdir -p ~/actions-runner-PROYECTO && cd ~/actions-runner-PROYECTO
curl -o r.tar.gz -L https://github.com/actions/runner/releases/download/v2.323.0/actions-runner-osx-arm64-2.323.0.tar.gz
tar xzf r.tar.gz && rm r.tar.gz
TOKEN=$(gh api repos/OWNER/REPO/actions/runners/registration-token -X POST --jq '.token')
./config.sh --url https://github.com/OWNER/REPO --token $TOKEN --name mac-PROYECTO --unattended
./svc.sh install && ./svc.sh start
```

---

## NestJS + AWS Elastic Beanstalk

```yaml
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

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | AWS access key |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key |
| `EB_APPLICATION_NAME` | Elastic Beanstalk app name |
| `EB_STAGING_ENV_NAME` | EB staging environment |
| `EB_PRODUCTION_ENV_NAME` | EB production environment |
| `S3_DEPLOY_BUCKET` | S3 bucket for deploy artifacts |

# golyv-actions

Shared composite actions and reusable workflows for the Golyv organisation.

This repository is **public** so that private repos across the org can consume it
on any GitHub plan. It contains build and deploy glue only — no secrets, no
proprietary code. Never commit credentials here.

## Layout

```
actions/                      composite actions
  ecr-build-push/             build + push an image to ECR (OIDC auth)
  bump-chart-values/          write the new tag into golyv-charts
  validate-branch-flow/       enforce which branches may merge where
  sonarqube-scan/             sonar-scanner, non-blocking
  trivy-scan/                 image vulnerability scan, non-blocking
  php-sast/                   composer audit + PHPStan, non-blocking
.github/workflows/            reusable workflows
  build-deploy.yml            the standard app pipeline
  validate-pr.yml             the standard PR gate
```

## Versioning

Consumers pin the `v1` tag. Move `v1` forward for backwards-compatible changes;
cut `v2` for anything that changes an input contract.

```bash
git tag -f v1 && git push -f origin v1
```

## Typical consumer

```yaml
# .github/workflows/dev.yml
name: Deploy dev
on:
  push:
    branches: [dev]
permissions:
  id-token: write   # required: build-deploy.yml federates to AWS via OIDC,
  contents: read    # and a called workflow can only narrow the caller's token
jobs:
  deploy:
    if: vars.GOLYV_CI_ENABLED == 'true'
    uses: Golyv-LLC/golyv-actions/.github/workflows/build-deploy.yml@v1
    with:
      image-name: hospital-backend-dev
      dockerfile: manifests/docker_file/Dockerfile-dev
      aws-role: ${{ vars.AWS_OIDC_ROLE_ARN }}
      chart-files: |
        values/hospital-staging-values.yaml
      chart-set: |
        .backend.image = strenv(IMAGE)
        .backend.tag = strenv(TAG)
    secrets:
      GOLYV_CI_APP_ID: ${{ secrets.GOLYV_CI_APP_ID }}
      GOLYV_CI_APP_PRIVATE_KEY: ${{ secrets.GOLYV_CI_APP_PRIVATE_KEY }}
```

## Required org configuration

| Name | Kind | Purpose |
|---|---|---|
| `GOLYV_CI_ENABLED` | variable | Master cutover switch. Every consuming job is gated on `vars.GOLYV_CI_ENABLED == 'true'`, so workflows sit inert until this is set. Setting it org-wide turns the whole estate on; unsetting it is the rollback. |
| `AWS_OIDC_ROLE_ARN` | variable | IAM role assumed via OIDC to push to ECR |
| `GOLYV_CI_APP_ID` | secret | `golyv-ci` GitHub App id |
| `GOLYV_CI_APP_PRIVATE_KEY` | secret | `golyv-ci` GitHub App private key |
| `SONAR_TOKEN` | secret | SonarQube token |
| `SONAR_HOST_URL` | secret | SonarQube server URL |

## Conventions carried over from GitLab CI

These are deliberate and load-bearing — do not "clean them up" without checking
what deploys from them:

- **Image tag is the 8-character short commit SHA**, identical to GitLab's
  `echo $CI_COMMIT_SHA | cut -c1-8`. ArgoCD is running on tags produced this way,
  so the scheme must not change.
- **Quality gates never block.** SonarQube, Trivy and PHP SAST all mirror
  `allow_failure: true`; they report and move on.
- **Charts are always bumped on the `dev` branch** of `golyv-charts`, regardless
  of which branch triggered the build. ArgoCD tracks `dev`.
- **`hospital-frontend` writes only `.frontend.tag`**, never `.frontend.image`.
- **`hospital-backend` production bumps every `values/hospital-*.yaml` except
  `values/hospital-staging-values.yaml`.**
- **`ondemand-backend` builds on every branch but deploys from only two**, and
  the `feat/media-spaces-migration` branch overrides both its image name and its
  target values file.

## Local testing

The two actions with real logic are pure bash and can be exercised outside CI by
extracting the `run:` block:

```bash
python3 -c "
import yaml,pathlib
a=yaml.safe_load(open('actions/bump-chart-values/action.yml'))
s=[x for x in a['runs']['steps'] if x.get('id')=='bump'][0]
pathlib.Path('/tmp/bump.sh').write_text('#!/usr/bin/env bash\n'+s['run'])"
```

then run it against a scratch clone of `golyv-charts` with a local bare remote.

## Gotchas

**Callers must declare `permissions`.** A reusable workflow can only *reduce* the
caller's token scopes, never widen them. `build-deploy.yml` needs
`id-token: write` for OIDC, so any workflow calling it must grant that itself —
otherwise the run dies at `startup_failure` before a single step executes, with
no log to explain why.

**Everything is gated on `GOLYV_CI_ENABLED`.** Workflows were installed while
GitLab CI was still the live system, so every job carries
`if: vars.GOLYV_CI_ENABLED == 'true'` and skips until that org variable is set.

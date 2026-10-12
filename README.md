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
  ecr-promote/                add a release tag to an existing image (v2)
.github/workflows/            reusable workflows
  build-deploy.yml            build, push, bump staging values
  release.yml                 promote a tested image to prod, no rebuild (v2)
  validate-pr.yml             PR gate: branch flow, SAST, Sonar, image build
```

## Versioning

| Tag | For | Notes |
|---|---|---|
| `v1` | existing pipelines (hospital, ondemand, whatsapp, …) | frozen; keeps working unchanged |
| `v2` | new pipelines, starting with rumbl-backend | adds release promotion, `push-latest`, PR SAST/image build |

Move a tag forward only for backwards-compatible changes. Anything that changes
an input contract gets a new major tag.

```bash
git tag -f v2 && git push -f origin v2
```

## Build once, promote to production (v2)

Staging builds the image once, tagged with the 8-character commit SHA. A release
does **not** rebuild. Pushing tag `v1.4.0` on a commit that staging already ran
adds `v1.4.0` to that same image (`ecr-promote`), then writes it into the
production instances' values. Production therefore runs the exact bytes staging
tested.

- Releasing a commit staging never built fails with a clear error. That is
  deliberate: it is the "was this tested?" check.
- Re-running a release is safe. If the tag already points at the same image,
  nothing happens; if it points at a different one, the release refuses.
- Images are OCI indexes (image + provenance attestation). `ecr-promote` passes
  the manifest media type through, which `put-image` requires.
- Use `push-latest: false` on repositories with **immutable** tags, or the second
  build fails trying to move `latest`.

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

## Required configuration

The org is on **GitHub Team**, so these are set **once at org level** with
visibility `all` and reach every repository, including private ones. Don't
duplicate them per repo.

> Historical note: while the org was on Free, org-level values did not reach
> private repos (they resolved to empty and gated jobs silently skipped), so
> some older repos also carry repo-level copies. Those still work and take
> precedence; remove them when convenient.

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

# Reusable CI templates

Reusable GitHub Actions workflows for building container images with Kaniko and
updating image tags in an infrastructure repository. Configuration is supplied
by the calling workflow through inputs and build secrets.

## Build and deploy example

Replace `YOUR_ORG`, `YOUR_APP`, and the workflow ref with your own values. Examples
use `new_templates`; use a published commit SHA to pin the workflow version.

```yaml
name: Build and deploy

on:
  push:
    branches:
      - main

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    uses: YOUR_ORG/github-templates/.github/workflows/build.yaml@new_templates
    with:
      kaniko_image: ${{ vars.KANIKO_IMAGE }}
      # Required: full image path without a tag.
      image: "${{ vars.REGISTRY_URL }}/YOUR_APP"
      # Source repository/ref/SHA are assembled by the caller.
      context: "${{ github.repositoryUrl }}#${{ github.ref }}#${{ github.sha }}"
      image_tag: ${{ github.sha }}
      git_username: ${{ github.actor }}
      # Select the runner in the calling repository.
      runner: ${{ vars.RUNNER_NAME }}
      registry: ${{ vars.REGISTRY }}
      dockerfile: ./Dockerfile
      # One image; use '["api","worker"]' for a service matrix.
      services: '[""]'
      cache: true
      push_retry: 5
      # Optional extra Kaniko flags; values must not contain whitespace.
      args: >-
        --build-arg=NODE_ENV=production
        --verbosity=debug
    secrets:
      source_token: ${{ secrets.GITHUB_TOKEN }}
      # Optional: omit when private npm packages are not needed.
      npm_token: ${{ secrets.NPM_TOKEN }}

  update-chart:
    needs: build
    uses: YOUR_ORG/github-templates/.github/workflows/update-image-tag.yaml@new_templates
    with:
      ci_image: ${{ vars.CI_IMAGE }}
      # Select the runner in the calling repository.
      runner: ${{ vars.RUNNER_NAME }}
      infra_repo_url: ${{ vars.INFRA_REPO_URL }}
      infra_branch: ${{ vars.INFRA_BRANCH }}
      infra_repo_workdir: infrastructure
      # Required: path relative to the infrastructure repository root.
      values_file: ${{ vars.VALUES_FILE }}
      image_tag: ${{ github.sha }}
      git_author_name: ${{ vars.GIT_AUTHOR_NAME }}
      git_author_email: ${{ vars.GIT_AUTHOR_EMAIL }}
      tag_path: ${{ vars.TAG_PATH }}
```

Every enabled branch runs the same build and infrastructure update. Add caller
job conditions if different branches require different deployment behavior.

## Build workflow

[build.yaml](.github/workflows/build.yaml) uses the maintained
[osscontainertools/kaniko fork](https://github.com/osscontainertools/kaniko), with
a version and digest supplied by the caller. The workflow runs Kaniko directly using the supplied inputs; no Docker daemon is
required. For registry hostnames ending in `.pkg.dev`, it configures
`docker-credential-gcr`. Other registries require credentials configured externally.

The caller supplies the build context and image tag. Each image is pushed with
the provided tag and `latest`.

### Inputs

| Input | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `image` | string | Yes | — | Full destination image path without a tag. |
| `kaniko_image` | string | Yes | — | Container image that provides `/kaniko/executor`. |
| `context` | string | Yes | — | Kaniko source context, usually `repository-url#ref#commit`. |
| `image_tag` | string | Yes | — | Tag identifying the source revision. |
| `git_username` | string | No | Empty | Username for private source repository access. |
| `registry` | string | Yes | — | Registry hostname. The Google credential helper is configured only for `*.pkg.dev`. |
| `runner` | string | Yes | — | Runner label or scale set name selected by the caller. |
| `dockerfile` | string | No | `./Dockerfile` | Dockerfile path in the Git build context. |
| `services` | string | No | `[""]` | JSON array of service names for parallel builds. |
| `cache` | boolean | No | `true` | Enable Kaniko layer caching. |
| `push_retry` | number | No | `5` | Number of retries for pushing an image. |
| `args` | string | No | Empty string | Additional whitespace-separated Kaniko arguments. |

### Secrets

| Secret | Required | Description |
| --- | --- | --- |
| `source_token` | No | Password/token used by Kaniko to read a private build context. |
| `npm_token` | No | Private npm registry token, forwarded as the `NPM_TOKEN` build argument, including when empty. The Dockerfile and package manager configuration must consume it. |

### All build options

```yaml
jobs:
  build:
    uses: YOUR_ORG/github-templates/.github/workflows/build.yaml@new_templates
    with:
      kaniko_image: ${{ vars.KANIKO_IMAGE }}
      image: europe-west2-docker.pkg.dev/YOUR_PROJECT/YOUR_REPOSITORY/YOUR_APP
      context: "${{ github.repositoryUrl }}#${{ github.ref }}#${{ github.sha }}"
      image_tag: ${{ github.sha }}
      git_username: ${{ github.actor }}
      registry: ${{ vars.REGISTRY }}
      runner: YOUR_RUNNER
      dockerfile: ./Dockerfile
      services: '["api","worker"]'
      cache: true
      push_retry: 5
      args: >-
        --build-arg=NODE_ENV=production
        --verbosity=debug
    secrets:
      source_token: ${{ secrets.GITHUB_TOKEN }}
      npm_token: ${{ secrets.NPM_TOKEN }}
```

A nonempty service adds `-<service>` to the image name and is passed as the
`SERVICE` build argument. The example builds `YOUR_APP-api` and `YOUR_APP-worker`,
each with the supplied `image_tag` and `latest` tags. Omitting `services` builds one image without a
suffix and with an empty `SERVICE` build argument. An empty array does not represent a single build;
use the default `[""]` for that behavior.

`SERVICE` and `NPM_TOKEN` are always passed as build arguments, including empty
values. Empty values override any defaults for these arguments in the Dockerfile.

Matrix builds use `fail-fast: false`: one failure does not cancel the other
service builds. A downstream job with `needs: build` runs only when all builds
succeed. Images already pushed by successful matrix jobs are not rolled back
when another service fails.

`args` is split on whitespace with filename expansion disabled (`set -f`). Do
not add shell continuation backslashes. Values containing whitespace are not
supported, and embedded quotes are not treated as shell quoting. For example,
`--build-arg=GIT_SHA=${{ github.sha }}` is supported. Avoid repeating flags already
managed by explicit inputs, and pass npm credentials through `secrets.npm_token`
instead of `args`.

## Update image tag workflow

[update-image-tag.yaml](.github/workflows/update-image-tag.yaml) runs in
`${{ inputs.ci_image }}`, which must provide Git, OpenSSH, `yq`, and a shell. It uses the infrastructure URL, commit tag, and author supplied by the caller.
The runner must provide an SSH key, trusted host keys, and `GIT_SSH_COMMAND`.
The workflow writes the tag as a YAML string and pushes the
change. An unchanged values file does not create an empty commit. Deployment
reconciliation is handled by the infrastructure tooling.

### Inputs

| Input | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `ci_image` | string | Yes | — | Container image providing Git and `yq`. |
| `values_file` | string | Yes | — | Values file path relative to the infrastructure repository root. |
| `infra_repo_url` | string | Yes | — | SSH clone URL for the infrastructure repository. |
| `image_tag` | string | Yes | — | Image tag to write to the values file. |
| `git_author_name` | string | Yes | — | Commit author name. |
| `git_author_email` | string | Yes | — | Commit author email. |
| `infra_branch` | string | No | `master` | Branch checked out, rebased, and pushed. |
| `infra_repo_workdir` | string | No | `infrastructure` | Checkout directory under the runner workspace. This does not change the base for `values_file`. |
| `tag_path` | string | No | `.global.image.tag` | Assignment target expressed as a `yq` path. |
| `runner` | string | Yes | — | Runner label or scale set name selected by the caller. |

### All update options

```yaml
jobs:
  update-chart:
    needs: build
    uses: YOUR_ORG/github-templates/.github/workflows/update-image-tag.yaml@new_templates
    with:
      ci_image: registry.example.com/ci/ci-deploy:v0.1.0
      infra_repo_url: ${{ vars.INFRA_REPO_URL }}
      infra_branch: master
      infra_repo_workdir: infrastructure
      values_file: environments/production/YOUR_APP/values.yaml
      image_tag: ${{ github.sha }}
      git_author_name: ${{ vars.GIT_AUTHOR_NAME }}
      git_author_email: ${{ vars.GIT_AUTHOR_EMAIL }}
      tag_path: .global.image.tag
      runner: YOUR_RUNNER
```

Concurrent updates from other workflows can cause rebase or push conflicts.
Inspect and resolve conflicts before rerunning the failed job; the template
does not implement a push retry loop for Git.

## Caller variables and access

Templates consume neutral inputs, not environment-specific repository variables.
The following names are used only by the application example above; callers may
use different names or pass literal values:

| Caller variable or secret | Maps to |
| --- | --- |
| `vars.REGISTRY_URL` | Base path used to form the build input `image`. |
| `vars.REGISTRY` | Artifact Registry host passed as the build workflow's `registry` input. |
| `vars.KANIKO_IMAGE` | Pinned Kaniko image passed as the `kaniko_image` input. |
| `vars.INFRA_REPO_URL` | Infrastructure SSH URL passed as `infra_repo_url`. |
| `vars.GIT_AUTHOR_NAME` | Commit author name passed as `git_author_name`. |
| `vars.GIT_AUTHOR_EMAIL` | Commit author email passed as `git_author_email`. |
| `vars.CI_IMAGE` | Full deploy image reference, for example `<registry>/ci-deploy:v0.1.0`. |
| `vars.RUNNER_NAME` | Required `runner` input in both workflows. |
| `vars.INFRA_BRANCH` | Update input `infra_branch`. |
| `vars.VALUES_FILE` | Update input `values_file`. |
| `vars.TAG_PATH` | Update input `tag_path`. |
| `secrets.NPM_TOKEN` | Optional build secret `npm_token`. |

An optional input's default applies when it is omitted. Explicitly passing an
empty repository variable does not select the default: configure the variable
or omit that input. Boolean and number inputs should use their native YAML
types, such as `cache: false` and `push_retry: 3`.

For GitHub Actions, set `permissions: contents: read` when mapping
`github.token` to `source_token`. Infrastructure writes use SSH credentials configured on the runner. Store secrets in the calling repository or
expose them through organization secrets with the appropriate repository access.

For private templates, enable access from the calling repositories in the
template repository's **Settings → Actions → General → Access**. Caller Actions
policies must allow the workflows and actions they use.

The self-hosted runner must support container jobs. For Artifact
Registry, Google credentials available to the build pod must permit image
and cache pushes; `docker-credential-gcr` configures their use. This workflow
does not submit Google Cloud Build jobs. Overriding `registry` alone does not
provision credentials for an unrelated registry provider.

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
      cache: true
      push_retry: 5
      # Optional extra Kaniko flags; values must not contain whitespace.
      args: >-
        --build-arg=NODE_ENV=production
        --verbosity=debug
    secrets:
      source_token: ${{ secrets.GITHUB_TOKEN }}
      # Optional: omit when private npm packages are not needed.
      secret_build_args: |
        NPM_TOKEN=${{ secrets.NPM_TOKEN }}

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
      tag_path: ${{ vars.TAG_PATH }}
```

Every enabled branch runs the same build and infrastructure update. Add caller
job conditions if different branches require different deployment behavior.

## Build workflow

[build.yaml](.github/workflows/build.yaml) uses the maintained
[osscontainertools/kaniko fork](https://github.com/osscontainertools/kaniko), with
a version and digest supplied by the caller. The workflow runs Kaniko directly
using the supplied inputs; no Docker daemon is required. For registry hostnames ending in `.pkg.dev`, it configures
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
| `cache` | boolean | No | `true` | Enable Kaniko layer caching. |
| `push_retry` | number | No | `5` | Number of retries for pushing an image. |
| `args` | string | No | Empty string | Additional whitespace-separated Kaniko arguments. |

### Secrets

| Secret | Required | Description |
| --- | --- | --- |
| `source_token` | No | Password/token used by Kaniko to read a private build context. |
| `secret_build_args` | No | Secret build arguments as `NAME=value` lines. Omitted or empty means no secret arguments. |

`secret_build_args` accepts arbitrary names as one `NAME=value` argument per line.
The template has no application-specific secret names:

```yaml
secrets:
  secret_build_args: |
    NPM_TOKEN=${{ secrets.NPM_TOKEN }}
    OTHER_TOKEN=${{ secrets.OTHER_TOKEN }}
```

Blank lines are ignored. Spaces, quotes, backslashes, and additional `=` characters
inside values are preserved literally; values must not contain newlines.
`toJSON` is not needed. Omit `secret_build_args` to pass no secret arguments.
A repository secret containing the entire list can also be passed directly
as `${{ secrets.BUILD_SECRETS }}`. The Dockerfile must avoid persisting secret
values in the resulting image.

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
      cache: true
      push_retry: 5
      args: >-
        --build-arg=NODE_ENV=production
        --verbosity=debug
    secrets:
      source_token: ${{ secrets.GITHUB_TOKEN }}
      secret_build_args: |
        NPM_TOKEN=${{ secrets.NPM_TOKEN }}
```

The template builds one image per call. Define a matrix in the calling application
and pass the full image name and service build argument explicitly:

```yaml
jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        service: [api, worker]
    uses: YOUR_ORG/github-templates/.github/workflows/build.yaml@new_templates
    with:
      registry: europe-west2-docker.pkg.dev
      kaniko_image: ${{ vars.KANIKO_IMAGE }}
      runner: ${{ vars.RUNNER_NAME }}
      image: "${{ vars.REGISTRY_URL }}/YOUR_APP-${{ matrix.service }}"
      context: "${{ github.repositoryUrl }}#${{ github.ref }}#${{ github.sha }}"
      image_tag: ${{ github.sha }}
      git_username: ${{ github.actor }}
      args: --build-arg=SERVICE=${{ matrix.service }}
    secrets:
      source_token: ${{ secrets.GITHUB_TOKEN }}
```

Service names must not contain whitespace. The template does not automatically
add image suffixes or a `SERVICE` build argument. Secret arguments are passed
only for entries in `secret_build_args`; empty values override Dockerfile defaults.

The matrix example uses `fail-fast: false`: one failure does not cancel the other
service builds. A downstream job with `needs: build` runs only when all builds
succeed. Images already pushed by successful matrix jobs are not rolled back
when another service fails.

`args` is split on whitespace with filename expansion disabled (`set -f`). Do
not add shell continuation backslashes. Values containing whitespace are not
supported, and embedded quotes are not treated as shell quoting. For example,
`--build-arg=GIT_SHA=${{ github.sha }}` is supported. Avoid repeating flags already
managed by explicit inputs, and pass secret arguments through
`secrets.secret_build_args` instead of `args`.

## Update image tag workflow

[update-image-tag.yaml](.github/workflows/update-image-tag.yaml) runs in
`${{ inputs.ci_image }}`, which must provide `ci-update-image-tag.sh`, Git, OpenSSH,
`yq`, and a shell. The executable is packaged in the `ci_deploy_img` image from
`bazel-lib/deps/images/ci`. The workflow passes the caller-provided infrastructure
URL and image tag to the executable through environment variables.
Commits use author and committer `ci <ci@svetoch.dev>`, as configured in the
workflow environment.
The runner must provide an SSH key, trusted host keys, and `GIT_SSH_COMMAND`.
The script writes the tag as a YAML string, commits the change, rebases on the
remote branch, and pushes. An unchanged values file does not create an empty commit. Deployment
reconciliation is handled by the infrastructure tooling.

### Inputs

| Input | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `ci_image` | string | Yes | — | Container image providing `ci-update-image-tag.sh`, Git, OpenSSH, and `yq`. |
| `values_file` | string | Yes | — | Values file path relative to the infrastructure repository root. |
| `infra_repo_url` | string | Yes | — | SSH clone URL for the infrastructure repository. |
| `image_tag` | string | Yes | — | Image tag to write to the values file. |
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
      ci_image: ${{ vars.CI_IMAGE }}
      infra_repo_url: ${{ vars.INFRA_REPO_URL }}
      infra_branch: master
      infra_repo_workdir: infrastructure
      values_file: environments/production/YOUR_APP/values.yaml
      image_tag: ${{ github.sha }}
      tag_path: .global.image.tag
      runner: YOUR_RUNNER
```

After moving or changing the script, rebuild and publish the CI deploy image.
`CI_IMAGE` must reference an image containing `/usr/local/bin/ci-update-image-tag.sh`;
an older image without the executable cannot run this workflow. Updating the
workflow alone does not update the script inside an already published image.

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
| `vars.CI_IMAGE` | Full deploy image reference, for example `<registry>/ci-deploy:v0.1.0`. |
| `vars.RUNNER_NAME` | Required `runner` input in both workflows. |
| `vars.INFRA_BRANCH` | Update input `infra_branch`. |
| `vars.VALUES_FILE` | Update input `values_file`. |
| `vars.TAG_PATH` | Update input `tag_path`. |
| `secrets.NPM_TOKEN` | Example value passed through `secret_build_args` by the caller. |

An optional input's default applies when it is omitted. Explicitly passing an
empty repository variable does not select the default: configure the variable
or omit that input. Boolean and number inputs should use their native YAML
types, such as `cache: false` and `push_retry: 3`.

For GitHub Actions, set `permissions: contents: read` when mapping
`github.token` to `source_token`. Infrastructure writes use SSH credentials configured on the runner. Store build
secrets in the calling repository or expose them through organization secrets
with the appropriate repository access.

For private templates, enable access from the calling repositories in the
template repository's **Settings → Actions → General → Access**. Caller Actions
policies must allow the workflows and actions they use.

The self-hosted runner must support container jobs. For Artifact
Registry, Google credentials available to the build pod must permit image
and cache pushes; `docker-credential-gcr` configures their use. This workflow
does not submit Google Cloud Build jobs. Overriding `registry` alone does not
provision credentials for an unrelated registry provider.

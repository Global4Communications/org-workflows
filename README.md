# org-workflows

Reusable GitHub Actions workflows shared across G4C projects. Call them with
`uses:` rather than copying the steps, so a fix lands everywhere at once.

## php-ci.yml

Runs PHPCS, PHPStan, Behat and PHPUnit on a pull request, skipping whatever the
repo has no config file for. PHPUnit is run via `vendor/bin/phpunit`, so the
repository's Composer constraints decide the version under test.

```yaml
name: CI

on:
  pull_request:
  workflow_dispatch:

jobs:
  php:
    uses: Global4Communications/org-workflows/.github/workflows/php-ci.yml@main
    secrets:
      webservices_pat: ${{ secrets.WEBSERVICES_PAT }}
```

## docker-build-push.yml

Builds an image and pushes it to GHCR. Copy this whole file into
`.github/workflows/build-docker-image.yml` and uncomment what you need.

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
    paths-ignore: ['**.md']
  pull_request:
    branches: [main]
    paths-ignore: ['**.md']
  workflow_dispatch:

jobs:
  build:
    uses: Global4Communications/org-workflows/.github/workflows/docker-build-push.yml@main
    secrets:
      webservices_pat: ${{ secrets.WEBSERVICES_PAT }}
    ## Everything below is optional and shows the defaults. Drop one "# " to
    ## enable a line; the ## notes stay comments.
    # with:
    #   ## The image is <registry>/<owner>/<name>, so an image is named after
    #   ## its repo. Override any one of the three on its own.
    #   registry: ghcr.io
    #   owner: global4communications
    #   name: ${{ github.event.repository.name }}
    #
    #   ## Tag family, for a second image built from the same repo. 'staging'
    #   ## gives staging, staging-<commit> and staging-pr-<n>.
    #   variant: ''
    #
    #   ## Prefer moving the file to the repo root over setting this.
    #   dockerfile: Dockerfile
    #
    #   ## One KEY=value per line.
    #   build_args: |
    #     ASSET_URL=https://www.hometelecom.co.uk/shop
```

### Tags

| Trigger | Tags pushed |
| --- | --- |
| push to main | `latest`, `<commit>` |
| pull request | `pr-<number>`, `<commit>` |

A PR only ever moves its own `pr-<number>` tag, so it cannot overwrite what
consumers track. The commit tag is immutable — pin a deployment to it, or roll
back to it. `sha_tag` and `digest` are exposed as outputs so a following job can
deploy exactly the build that just ran.

### Build metadata

`BUILD_COMMIT`, `BUILD_TIMESTAMP` and `BUILD_RUN` are always passed as build args
and set as OCI labels, so any image can be identified with
`docker buildx imagetools inspect` without pulling it. Dockerfiles that don't
declare the args are unaffected.

Web apps should also serve them at `/version.txt`, so the running build can be
checked over HTTP. This has to live in the Dockerfile — only the app knows where
its document root is. Put it in the **last** layer, or the changing values
invalidate the `composer install` layer every build:

```dockerfile
ARG BUILD_COMMIT=unknown
ARG BUILD_TIMESTAMP=unknown
ARG BUILD_RUN=unknown
RUN printf 'commit: %s\nbuilt:  %s\nrun:    %s\n' \
        "$BUILD_COMMIT" "$BUILD_TIMESTAMP" "$BUILD_RUN" \
        > /var/www/html/public/version.txt \
    && chown phpapp:phpapp /var/www/html/public/version.txt
```

Deployment is deliberately not part of this workflow — it differs too much
between apps (terraform pin, `az` sitecontainer update, restart-to-pull) to be
worth a pile of conditional inputs.

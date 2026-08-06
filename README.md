# org-workflows

Reusable GitHub Actions workflows shared across G4C projects.  Call them with
`uses:` rather than copying the steps into each repo, so a fix lands everywhere
at once.

## `php-ci.yml`

Runs PHPCS, PHPStan, Behat and PHPUnit on a pull request, skipping whatever the
repo has no config file for.

```yaml
jobs:
  php:
    uses: Global4Communications/org-workflows/.github/workflows/php-ci.yml@main
    secrets:
      webservices_pat: ${{ secrets.WEBSERVICES_PAT }}
```

## `docker-build-push.yml`

Builds an image and pushes it to GHCR with a consistent tag scheme.

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
```

### Naming

**An image is named after its repo.** `home-switcher` builds
`ghcr.io/global4communications/home-switcher`, and no `with:` block is needed.

`image` takes the whole reference, so it can be overridden as far as you need —
a different name in the usual place, or a different registry entirely:

```yaml
    with:
      image: ghcr.io/global4communications/giacom-api
```

```yaml
    with:
      image: someregistry.azurecr.io/team/thing
```

The workflow logs in to whichever registry the reference names, so the second
case pushes rather than failing at the end of a build — though `webservices_pat`
has to be a credential that registry accepts.

Do not include a tag; tags are added for you, and the build fails if you pass one
so a typo can't quietly publish `:latest:v2`. Uppercase is rejected for the same
reason.

Likewise the Dockerfile is expected at the repo root. Override with
`dockerfile:` if it lives elsewhere, but prefer moving the file.

### Tags

| Trigger | Tags pushed |
| --- | --- |
| push to main | `latest`, `<commit>` |
| pull request | `pr-<number>`, `<commit>` |

A PR only ever moves its own `pr-<number>` tag, so it cannot overwrite what
consumers track.  The commit tag is immutable — pin a deployment to it, or roll
back to it.

Set `variant` to build a tag family instead: `variant: staging` gives `staging`,
`staging-<commit>` and `staging-pr-<n>`.  Use it for a second image built from
the same repo, or for a matrix of base images (`php82`, `php81`, …).

### Build metadata

Three build args are always passed, whether or not the Dockerfile declares them:

| Arg | Value |
| --- | --- |
| `BUILD_COMMIT` | commit the image was built from |
| `BUILD_TIMESTAMP` | UTC build time, ISO 8601 |
| `BUILD_RUN` | workflow run number |

The same values go on as OCI labels, so any image can be identified without
pulling it:

```
docker buildx imagetools inspect ghcr.io/global4communications/<image>:latest
```

Web apps should also write them into the document root, so the running build can
be checked over HTTP at `/version.txt`.  This part has to live in the Dockerfile
— only the app knows where its document root is, and the base image cannot write
into a directory the app has not copied in yet.  Put it in the **last** layer, or
the changing values invalidate the `composer install` layer on every build:

```dockerfile
ARG BUILD_COMMIT=unknown
ARG BUILD_TIMESTAMP=unknown
ARG BUILD_RUN=unknown
RUN printf 'commit: %s\nbuilt:  %s\nrun:    %s\n' \
        "$BUILD_COMMIT" "$BUILD_TIMESTAMP" "$BUILD_RUN" \
        > /var/www/html/public/version.txt \
    && chown phpapp:phpapp /var/www/html/public/version.txt
```

### Outputs

`sha_tag` and `digest` are exposed so a follow-on job can deploy exactly the
build that just ran, rather than resolving a moving tag again:

```yaml
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "deploying ${{ needs.build.outputs.digest }}"
```

Deployment itself is deliberately not part of this workflow — it differs too much
between apps (terraform pin, `az` sitecontainer update, restart-to-pull) to be
worth a pile of conditional inputs.

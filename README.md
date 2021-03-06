[![GitHub release](https://img.shields.io/github/release/crazy-max/ghaction-docker-meta.svg?style=flat-square)](https://github.com/crazy-max/ghaction-docker-meta/releases/latest)
[![GitHub marketplace](https://img.shields.io/badge/marketplace-docker--meta-blue?logo=github&style=flat-square)](https://github.com/marketplace/actions/docker-meta)
[![Test workflow](https://img.shields.io/github/workflow/status/crazy-max/ghaction-docker-meta/test?label=test&logo=github&style=flat-square)](https://github.com/crazy-max/ghaction-docker-meta/actions?workflow=test)
[![Codecov](https://img.shields.io/codecov/c/github/crazy-max/ghaction-docker-meta?logo=codecov&style=flat-square)](https://codecov.io/gh/crazy-max/ghaction-docker-meta)
[![Become a sponsor](https://img.shields.io/badge/sponsor-crazy--max-181717.svg?logo=github&style=flat-square)](https://github.com/sponsors/crazy-max)
[![Paypal Donate](https://img.shields.io/badge/donate-paypal-00457c.svg?logo=paypal&style=flat-square)](https://www.paypal.me/crazyws)

## About

GitHub Action to extract metadata (tags, labels) for Docker. This action is particularly useful if used with
[Docker Build Push](https://github.com/docker/build-push-action) action.

If you are interested, [check out](https://git.io/Je09Y) my other :octocat: GitHub Actions!

![Screenshot](.github/ghaction-docker-meta.png)

___

* [Usage](#usage)
  * [Basic](#basic)
  * [Semver](#semver)
  * [Bake definition](#bake-definition)
* [Customizing](#customizing)
  * [inputs](#inputs)
  * [outputs](#outputs)
* [Notes](#notes)
  * [Latest tag](#latest-tag)
  * [Handle semver tag](#handle-semver-tag)
  * [`tag-match` examples](#tag-match-examples)
  * [Schedule tag](#schedule-tag)
  * [Overwrite labels](#overwrite-labels)
* [Keep up-to-date with GitHub Dependabot](#keep-up-to-date-with-github-dependabot)
* [Contributing](#contributing)
* [License](#license)

## Usage

### Basic

| Event           | Ref                           | Commit SHA | Docker Tags                         |
|-----------------|-------------------------------|------------|-------------------------------------|
| `pull_request`  | `refs/pull/2/merge`           | `a123b57`  | `pr-2`                              |
| `push`          | `refs/heads/master`           | `cf20257`  | `master`                            |
| `push`          | `refs/heads/my/branch`        | `a5df687`  | `my-branch`                         |
| `push tag`      | `refs/tags/v1.2.3`            | `ad132f5`  | `v1.2.3`, `latest`                  |
| `push tag`      | `refs/tags/v2.0.8-beta.67`    | `fc89efd`  | `v2.0.8-beta.67`, `latest`          |

```yaml
name: ci

on:
  push:
    branches:
      - '**'
    tags:
      - 'v*'
  pull_request:

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      -
        name: Checkout
        uses: actions/checkout@v2
      -
        name: Docker meta
        id: docker_meta
        uses: crazy-max/ghaction-docker-meta@v1
        with:
          images: name/app
      -
        name: Set up QEMU
        uses: docker/setup-qemu-action@v1
      -
        name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1
      -
        name: Login to DockerHub
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      -
        name: Build and push
        uses: docker/build-push-action@v2
        with:
          context: .
          file: ./Dockerfile
          platforms: linux/amd64,linux/arm64,linux/386
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.docker_meta.outputs.tags }}
          labels: ${{ steps.docker_meta.outputs.labels }}
```

### Semver

| Event           | Ref                           | Commit SHA | Docker Tags                         |
|-----------------|-------------------------------|------------|-------------------------------------|
| `pull_request`  | `refs/pull/2/merge`           | `a123b57`  | `pr-2`                              |
| `push`          | `refs/heads/master`           | `cf20257`  | `master`                            |
| `push`          | `refs/heads/my/branch`        | `a5df687`  | `my-branch`                         |
| `push tag`      | `refs/tags/v1.2.3`            | `ad132f5`  | `1.2.3`, `1.2`, `latest`            |
| `push tag`      | `refs/tags/v2.0.8-beta.67`    | `fc89efd`  | `2.0.8-beta.67`                     |

```yaml
name: ci

on:
  push:
    branches:
      - '**'
    tags:
      - 'v*'
  pull_request:

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      -
        name: Checkout
        uses: actions/checkout@v2
      -
        name: Docker meta
        id: docker_meta
        uses: crazy-max/ghaction-docker-meta@v1
        with:
          images: name/app
          tag-semver: |
            {{version}}
            {{major}}.{{minor}}
      -
        name: Set up QEMU
        uses: docker/setup-qemu-action@v1
      -
        name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1
      -
        name: Login to DockerHub
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      -
        name: Build and push
        uses: docker/build-push-action@v2
        with:
          context: .
          file: ./Dockerfile
          platforms: linux/amd64,linux/arm64,linux/386
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.docker_meta.outputs.tags }}
          labels: ${{ steps.docker_meta.outputs.labels }}
```

### Bake definition

This action also handles a bake definition file that can be used with the
[Docker Buildx Bake action](https://github.com/crazy-max/ghaction-docker-buildx-bake). You just have to declare an empty
target named `ghaction-docker-meta` and inherit from it.

```hcl
// docker-bake.hcl

target "ghaction-docker-meta" {}

target "build" {
  inherits = ["ghaction-docker-meta"]
  context = "./"
  dockerfile = "Dockerfile"
  platforms = ["linux/amd64", "linux/arm/v6", "linux/arm/v7", "linux/arm64", "linux/386", "linux/ppc64le"]
}
```

```yaml
name: ci

on:
  push:
    branches:
      - '**'
    tags:
      - 'v*'
  pull_request:

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      -
        name: Checkout
        uses: actions/checkout@v2
      -
        name: Docker meta
        id: docker_meta
        uses: crazy-max/ghaction-docker-meta@v1
        with:
          images: name/app
          tag-sha: true
          tag-semver: |
            {{version}}
            {{major}}.{{minor}}
      -
        name: Set up QEMU
        uses: docker/setup-qemu-action@v1
      -
        name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1
      -
        name: Build
        uses: crazy-max/ghaction-docker-buildx-bake@v1
        with:
          files: |
            ./docker-bake.hcl
            ${{ steps.docker_meta.outputs.bake-file }}
          targets: |
            build
```

Content of `${{ steps.docker_meta.outputs.bake-file }}` file will look like this:

```json
{
  "target": {
    "ghaction-docker-meta": {
      "tags": [
        "name/app:1.1.1",
        "name/app:1.1",
        "name/app:latest"
      ],
      "labels": {
        "org.opencontainers.image.title": "Hello-World",
        "org.opencontainers.image.description": "This your first repo!",
        "org.opencontainers.image.url": "https://github.com/octocat/Hello-World",
        "org.opencontainers.image.source": "https://github.com/octocat/Hello-World",
        "org.opencontainers.image.version": "1.1.1",
        "org.opencontainers.image.created": "2020-01-10T00:30:00.000Z",
        "org.opencontainers.image.revision": "90dd6032fac8bda1b6c4436a2e65de27961ed071",
        "org.opencontainers.image.licenses": "MIT"
      },
      "args": {
        "DOCKER_META_IMAGES": "name/app",
        "DOCKER_META_VERSION": "1.1.1"
      }
    }
  }
}
```

## Customizing

### inputs

Following inputs can be used as `step.with` keys

> `List` type is a newline-delimited string
> ```yaml
> label-custom: |
>   org.opencontainers.image.title=MyCustomTitle
>   org.opencontainers.image.description=Another description
>   org.opencontainers.image.vendor=MyCompany
> ```

> `CSV` type is a comma-delimited string
> ```yaml
> images: name/app,ghcr.io/name/app
> ```

| Name                | Type     | Description                        |
|---------------------|----------|------------------------------------|
| `images`            | List/CSV | List of Docker images to use as base name for tags |
| `tag-sha`           | Bool     | Add git short commit as Docker tag (default `false`) |
| `tag-edge`          | Bool     | Enable edge branch tagging (default `false`) |
| `tag-edge-branch`   | String   | Branch that will be tagged as edge (default `repo.default_branch`) |
| `tag-semver`        | List/CSV | Handle Git tag as semver [template](#handle-semver-tag) if possible |
| `tag-match`         | String   | RegExp to match against a Git tag and use first match as Docker tag |
| `tag-match-group`   | Number   | Group to get if `tag-match` matches (default `0`) |
| `tag-latest`        | Bool     | Set `latest` Docker tag if `tag-semver`, `tag-match` or Git tag event occurs (default `true`) |
| `tag-schedule`      | String   | [Template](#schedule-tag) to apply to schedule tag (default `nightly`) |
| `tag-custom`        | List/CSV | List of custom tags |
| `tag-custom-only`   | Bool     | Only use `tag-custom` as Docker tags |
| `label-custom`      | List     | List of custom labels |
| `sep-tags`          | String   | Separator to use for tags output (default `\n`) |
| `sep-labels`        | String   | Separator to use for labels output (default `\n`) |

> `tag-semver` and `tag-match` are mutually exclusive

### outputs

Following outputs are available

| Name          | Type    | Description                           |
|---------------|---------|---------------------------------------|
| `version`     | String  | Docker image version |
| `tags`        | String  | Docker tags |
| `labels`      | String  | Docker labels |
| `bake-file`   | File    | [Bake definition file](https://github.com/docker/buildx#file-definition) path |

## Notes

### Latest tag

Latest Docker tag will be generated by default on `push tag` event. If for example you push the `v1.2.3` Git tag,
you will have at the output of this action the Docker tags `v1.2.3` and `latest`. But you can allow the latest tag to be
generated only if `tag-semver` is a valid [semver](https://semver.org/) or if Git tag matches a regular expression
with the [`tag-match` input](#tag-match-examples). Can be disabled if `tag-latest` is `false`.

### Handle semver tag

If Git tag is a valid [semver](https://semver.org/) you can handle it to output multi Docker tags at once.
`tag-semver` supports multi-line [Handlebars template](https://handlebarsjs.com/guide/) with the following inputs:

| Git tag            | `tag-semver`                                             | Valid              | Output tags                | Output version               |
|--------------------|----------------------------------------------------------|--------------------|----------------------------|------------------------------|
| `v1.2.3`           | `{{raw}}`                                                | :white_check_mark: | `v1.2.3`, `latest`         | `v1.2.3`                     |
| `v1.2.3`           | `{{version}}`                                            | :white_check_mark: | `1.2.3`, `latest`          | `1.2.3`                      |
| `v1.2.3`           | `{{major}}.{{minor}}`                                    | :white_check_mark: | `1.2`, `latest`            | `1.2`                        |
| `v1.2.3`           | `v{{major}}`                                             | :white_check_mark: | `v1`, `latest`             | `v1`                         |
| `v1.2.3`           | `{{minor}}`                                              | :white_check_mark: | `2`, `latest`              | `2`                          |
| `v1.2.3`           | `{{patch}}`                                              | :white_check_mark: | `3`, `latest`              | `3`                          |
| `v1.2.3`           | `{{major}}.{{minor}}`<br>`{{major}}.{{minor}}.{{patch}}` | :white_check_mark: | `1.2`, `1.2.3`, `latest`   | `1.2`*                       |
| `v2.0.8-beta.67`   | `{{raw}}`                                                | :white_check_mark: | `2.0.8-beta.67`**          | `2.0.8-beta.67`              |
| `v2.0.8-beta.67`   | `{{version}}`                                            | :white_check_mark: | `2.0.8-beta.67`            | `2.0.8-beta.67`              |
| `v2.0.8-beta.67`   | `{{major}}.{{minor}}`                                    | :white_check_mark: | `2.0.8-beta.67`**          | `2.0.8-beta.67`              |
| `release1`         | `{{raw}}`                                                | :x:                | `release1`                 | `release1`                   |

> *First occurrence of `tag-semver` will be taken as `output.version`

> **Pre-release (rc, beta, alpha) will only extend `{{version}}` as tag because they are updated frequently,
> and contain many breaking changes that are (by the author's design) not yet fit for public consumption.

### `tag-match` examples

| Git tag                 | `tag-match`                        | `tag-match-group` | Match                | Output tags               | Output version               |
|-------------------------|------------------------------------|-------------------|----------------------|---------------------------|------------------------------|
| `v1.2.3`                | `\d{1,3}.\d{1,3}.\d{1,3}`          | `0`               | :white_check_mark:   | `1.2.3`, `latest`         | `1.2.3`                      |
| `v2.0.8-beta.67`        | `v(.*)`                            | `1`               | :white_check_mark:   | `2.0.8-beta.67`, `latest` | `2.0.8-beta.67`              |
| `v2.0.8-beta.67`        | `v(\d.\d)`                         | `1`               | :white_check_mark:   | `2.0`, `latest`           | `2.0`                        |
| `release1`              | `\d{1,3}.\d{1,3}`                  | `0`               | :x:                  | `release1`                | `release1`                   |
| `20200110-RC2`          | `\d+`                              | `0`               | :white_check_mark:   | `20200110`, `latest`      | `20200110`                   |

### Schedule tag

`tag-schedule` is specially crafted input to support [Handlebars template](https://handlebarsjs.com/guide/) with
the following expressions:

| Expression              | Example                                   | Description                              |
|-------------------------|-------------------------------------------|------------------------------------------|
| `{{date 'format'}}`     | `{{date 'YYYYMMDD'}}` > `20200110`        | Render date by its [moment format](https://momentjs.com/docs/#/displaying/format/)

You can find more examples in the [CI workflow](.github/workflows/ci.yml).

### Overwrite labels

If some of the [OCI Image Format Specification](https://github.com/opencontainers/image-spec/blob/master/annotations.md)
labels generated are not suitable, you can overwrite them like this:

```yaml
      -
        name: Docker meta
        id: docker_meta
        uses: crazy-max/ghaction-docker-meta@v1
        with:
          images: name/app
          label-custom: |
            maintainer=CrazyMax
            org.opencontainers.image.title=MyCustomTitle
            org.opencontainers.image.description=Another description
            org.opencontainers.image.vendor=MyCompany
```

## Keep up-to-date with GitHub Dependabot

Since [Dependabot](https://docs.github.com/en/github/administering-a-repository/keeping-your-actions-up-to-date-with-github-dependabot)
has [native GitHub Actions support](https://docs.github.com/en/github/administering-a-repository/configuration-options-for-dependency-updates#package-ecosystem),
to enable it on your GitHub repo all you need to do is add the `.github/dependabot.yml` file:

```yaml
version: 2
updates:
  # Maintain dependencies for GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "daily"
```

## Contributing

Want to contribute? Awesome! The most basic way to show your support is to star :star2: the project,
or to raise issues :speech_balloon:. If you want to open a pull request, please read the
[contributing guidelines](.github/CONTRIBUTING.md).

You can also support this project by [**becoming a sponsor on GitHub**](https://github.com/sponsors/crazy-max) or by
making a [Paypal donation](https://www.paypal.me/crazyws) to ensure this journey continues indefinitely!

Thanks again for your support, it is much appreciated! :pray:

## License

MIT. See `LICENSE` for more details.

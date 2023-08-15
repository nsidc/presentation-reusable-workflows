---
title: Re-usable GitHub Actions workflows
format:
  revealjs: default
---

## Things you already know about code re-use

* Enables maintenance of a single thing to improve many things
* Enables abstraction of unimportant details
  * "Which container registry or registries should I publish to?"


## An example workflow

Modified from [source](https://github.com/nsidc/usaon-vta-survey/blob/v0.2.0/.github/workflows/release.yml)

```yaml
name: "Build and release container image"

# Triggers
on:
  push:
    branches:
      - "main"
  release:
    types:
      - "published"


# A group of steps that occurs in the same run environment
jobs:

  build-and-release-image:
    name: "Build and release container image"
    runs-on: "ubuntu-latest"
    env:
      IMAGE_NAME: "nsidc/usaon-vta-survey"
      # GitHub Actions expressions don't have great conditional support, so
      # writing a ternary expression looks a lot like bash. In Python, this
      # would read as:
      #     github.event.release.tag_name if github.event_name == 'release' else 'latest'
      #     https://docs.github.com/en/actions/learn-github-actions/expressions
      IMAGE_TAG: "${{ github.event_name == 'release' && github.event.release.tag_name || 'latest' }}"
    steps:
      - name: "Check out repository"
        uses: "actions/checkout@v3"

      - name: "Build container image"
        run: |
          docker build -t "${IMAGE_NAME}:${IMAGE_TAG}" .

      - name: "DockerHub login"
        uses: "docker/login-action@v2"
        with:
          username: "${{secrets.DOCKER_USER}}"
          password: "${{secrets.DOCKER_PASS}}"

      - name: "GHCR login"
        uses: "docker/login-action@v2"
        with:
          registry: "ghcr.io"
          username: "${{ github.repository_owner }}"
          password: "${{ secrets.GITHUB_TOKEN }}"

      - name: "Push to DockerHub and GHCR"
        run: |
          docker push "${IMAGE_NAME}:${IMAGE_TAG}"

          docker tag "${IMAGE_NAME}:${IMAGE_TAG}" "ghcr.io/${IMAGE_NAME}:${IMAGE_TAG}"
          docker push "ghcr.io/${IMAGE_NAME}:${IMAGE_TAG}"
```

::: {.notes}
Review `on` (triggers), `jobs` (groups of steps in the same execution env, and `steps`.
:::


## Re-using with GitHub Actions

* [Composite
    actions](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action):
  Callable `steps`
* [Re-usable workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows):
  Callable `jobs`
* [Starter
    workflows](https://docs.github.com/en/actions/using-workflows/creating-starter-workflows-for-your-organization):
  Copyable YAML text -- no args!

::: {.notes}
Composite actions and re-usable workflows are both callable from the local repo or by
referencing another repo with a ref.

Starter workflows are what you find in the "Marketplace" when you click "New workflow"
in the "Actions" tab on a repo.
:::


## In action

[diff](https://github.com/nsidc/usaon-vta-survey/commit/0978859bef571e17690b324fc1737e75dbe2cbce)

```yaml
name: "Build and release container image"

on:
  push:
    branches:
      - "main"
    tags:
      - "v[0-9]+.[0-9]+.[0-9]+*"


jobs:

  build-and-release-image:
    uses: "nsidc/.github/.github/workflows/build-and-publish-container-image.yml@main"
    secrets: "inherit"
```

::: {.notes}
This call of our re-usable workflow does not pass any arguments. The `with` keyword
would be used to pass arguments.

`secrets: "inherit"` enables secrets to be passed to the workflow _only if_ the
re-usable workflow is in the same organization or repository as the caller.
:::


## Re-usable workflow source

[source](https://github.com/nsidc/.github/blob/main/.github/workflows/build-and-publish-container-image.yml)

* The trigger event is `workflow_call`, which takes `inputs` and `secrets` as args
* Everything else is nearly identical to the example workflow
* ...except a bit of magic to automatically detect the calling repository's name:
  * `IMAGE_NAME: "nsidc/${{ github.event.repository.name }}"`


## Starter workflows

* For "bootstrapping" a new workflow file
* Enable sharing workflow configuration _text_ within the GitHub GUI
    * "Actions" -> "New workflow"
* Do not support passing arguments
* Can be defined at the org level by a combination of a YAML and JSON file in
  `my-org/.github` repo's `/workflow-templates` directory.

[NSIDC's starter workflows](https://github.com/nsidc/.github/tree/main/workflow-templates)

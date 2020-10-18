![](https://github.com/jupyterhub/repo2docker-action/workflows/Test/badge.svg) [![MLOps](https://img.shields.io/badge/MLOps-black.svg?logo=github&?logoColor=blue)](https://mlops-github.com)


# <a href="https://github.com/jupyter/repo2docker"><img src="https://raw.githubusercontent.com/jupyter/repo2docker/3fa7444fca6ae2b51e590cbc9d83baf92738ca2a/docs/source/_static/images/repo2docker.png" height="40px" /></a>  repo2docker GitHub Action


<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [What Can I Do With This Action?](#what-can-i-do-with-this-action)
- [API Reference](#api-reference)
	- [Mandatory Inputs](#mandatory-inputs)
	- [Optional Inputs](#optional-inputs)
	- [Outputs](#outputs)
- [Examples](#examples)
	- [mybinder.org](#mybinderorg)
		- [Cache builds on mybinder.org](#cache-builds-on-mybinderorg)
		- [Cache Builds On mybinder.org And Provide A Link](#cache-builds-on-mybinderorg-and-provide-a-link)
		- [Use GitHub Actions To Cache The Build For BinderHub](#use-github-actions-to-cache-the-build-for-binderhub)
	- [Push Repo2Docker Image To DockerHub](#push-repo2docker-image-to-dockerhub)
	- [Push Image To A Registry Other Than DockerHub](#push-image-to-a-registry-other-than-dockerhub)
	- [Change Image Name](#change-image-name)
	- [Test Image Build](#test-image-build)
- [Contributing](#contributing-to-repo2docker-action)

<!-- /TOC -->


Trigger [repo2docker](https://github.com/jupyter/repo2docker) to build a Jupyter enabled Docker image from your GitHub repository and push this image to a Docker registry of your choice.  This will automatically attempt to build an environment from configuration files found in your repository in the [manner described here](https://repo2docker.readthedocs.io/en/latest/usage.html#where-to-put-configuration-files).

Read the full docs on repo2docker for more information:  https://repo2docker.readthedocs.io

Images generated by this action are automatically tagged with both `latest` and `<SHA>` corresponding to the relevant [commit SHA on GitHub](https://help.github.com/en/github/getting-started-with-github/github-glossary#commit).  Both tags are pushed to the Docker registry specified by the user. If an existing image with the `latest` tag already exists in your registry, this Action attempts to pull that image as a cache to reduce uncessary build steps.

# What Can I Do With This Action?

- Use repo2docker to pre-cache images for your own [BinderHub cluster](https://binderhub.readthedocs.io/en/latest/zero-to-binderhub/setup-binderhub.html), or for [mybinder.org](https://mybinder.org/).
  - You can use this Action to pre-cache Docker images to a Docker registry that you can reference in your repo.  For example if you have the file `Dockerfile` in the `binder/` directory relative to the root of your repository with the following contents, this will allow Binder to start quickly by pulling an image you have already built:

    ```dockerfile
    # This is the image that is built and pushed by this Action (replace this with your image name)
    FROM myorg/myimage:latest
    ...
    ```
- Provide a way to Dockerize data science repositories with Jupyter server enabled that you can deploy to VMs, [serverless computing](https://en.wikipedia.org/wiki/Serverless_computing) or other services that can serve Docker containers as-a-service.
- Maximize reproducibility by allowing authors, without any prior knowledge of Docker, to build and share containers.

# API Reference

See the [examples](#examples) section is very helpful for understanding the inputs and outputs of this Action.

## Mandatory Inputs

**Exception: if the input parameter `NO_PUSH` is set to any value, these values become optional.**

- `DOCKER_USERNAME`:
    description: Docker registry username
- `DOCKER_PASSWORD`:
    description: Docker registry password or [access token](https://docs.docker.com/docker-hub/access-tokens/).  **If using DockerHub, we recommend using an access token instead of your password for security reasons.**

## Optional Inputs

- **`NOTEBOOK_USER`**:
    description: username of the primary user in the image. If this is not specified, this is set to `joyvan`.  **NOTE**: This value is also overriden with `jovyan` if the parameters `BINDER_CACHE` or `MYBINDERORG_TAG` are provided.
- **`REPO_DIR`**:
    Path inside the image where contents of the repositories are copied to, and where all the build operations (such as postBuild) happen. Defaults to `/home/<NOTEBOOK_USER>` if not set.
- **`IMAGE_NAME`**:
    name of the image.  Example - myusername/myContainer.  If not supplied, this defaults to `<DOCKER_USERNAME/GITHUB_REPOSITORY_NAME>`.
- **`DOCKER_REGISTRY`**:
    description: name of the docker registry.  If not supplied, this defaults to [DockerHub](https://hub.docker.com/)
- **`LATEST_TAG_OFF`**:
    Setting this variable to any value will prevent your image from being tagged with `latest`. Note that your image is always tagged with the [GitHub commit SHA](https://help.github.com/en/github/getting-started-with-github/github-glossary#commit).  
- **`ADDITIONAL_TAG`**:
    An optional string that specifies the name of an additional tag you would like to apply to the image.  Images are already tagged with the relevant [GitHub commit SHA](https://help.github.com/en/github/getting-started-with-github/github-glossary#commit).
- **`NO_PUSH`**:
    Setting this variable to any value will prevent any images from being pushed to a registry.  Furthermore, verbose logging will be enabled in this mode.  This is disabled by default.
- **`BINDER_CACHE`**:
    Setting this variable to any value will add the file `binder/Dockerfile` that references the docker image that was pushed to the registry by this Action.  You cannot use this option if the parameter `NO_PUSH` is set.  This is disabled by default.
    - Note: This Action assumes you are not explicitly using Binder to build your dependencies (You are using this Action to build your dependencies).  If a directory `binder` with other files other than `Dockerfile` or a directory named `.binder/` is detected, this step will be aborted.  This Action does not support caching images for Binder where dependencies are defined in `binder/Dockerfile` (if you are defining your dependencies this way, you probably don't need this Action).

      When this parameter is supplied, this Action will add/override `binder/Dockerfile` in the branch checked out in the Actions runner:
      ```dockerfile
      ### DO NOT EDIT THIS FILE! This Is Automatically Generated And Will Be Overwritten ###
      FROM <IMAGE_NAME>
      ```
- **`COMMIT_MSG`**:
     The commit message associated with specifying the `BINDER_CACHE` flag.  If no value is specified, the default commit message of `Update image tag` will be entered.
- **`MYBINDERORG_TAG`**:
    This the Git branch, tag, or commit that you want [mybinder.org](https://mybinder.org/) to proactively build from your repo.  This is useful if you wish to reduce startup time on mybinder.org.  **Your repository must be public for this work, as mybinder.org only works with public repositories**.
- **`PUBLIC_REGISTRY_CHECK`**:
    Setting this variable to any value will validate that the image pushed to the registry is publicly visible.

## Outputs

- **`IMAGE_SHA_NAME`**
    The name of the docker image, which is tagged with the SHA.
- **`PUSH_STATUS`**:
    This is `false` if `NO_PUSH` is provided or `true` otherwhise.

# Examples

## mybinder.org

A very popular use case for this Action is to cache builds for [mybinder.org](https://mybinder.org/).  If you desire to cache builds for mybinder.org, you must specify the argument `MYBINDERORG_TAG`.  Some examples of doing this are below:

### Cache builds on mybinder.org

Proactively build your environment on mybinder.org for any branch.  Alternatively, you can use [using GitHub Actions to build an image for BindHub generally](https://github.com/jupyterhub/repo2docker-action#use-github-actions-to-cache-the-build-for-binderhub), including mybinder.org.

```yaml
name: Binder
on: [push]

jobs:
  Create-MyBinderOrg-Cache:
    runs-on: ubuntu-latest
    steps:
    - name: cache binder build on mybinder.org
      uses: jupyterhub/repo2docker-action@master
      with:
        NO_PUSH: true
        MYBINDERORG_TAG: ${{ github.event.ref }} # This builds the container on mybinder.org with the branch that was pushed on.
```

### Cache Builds On mybinder.org And Provide A Link

Same example as above, but also comment on a PR with a link to the binder environment.  Commenting on the PR is optional, and is included here for informational purposes only.  In this example the image will only be cached when the pull request is opened but not if the pull request is updated with subsequent commits.

In this example the image will only be cached when the pull request is opened but not if the pull request is updated with subsequent commits.

```yaml
name: Binder
on:
  pull_request:
    types: [opened, reopened]

jobs:
  Create-Binder-Badge:
    runs-on: ubuntu-latest
    steps:
    - name: cache binder build on mybinder.org
      uses: jupyterhub/repo2docker-action@master
      with:
        NO_PUSH: true
        MYBINDERORG_TAG: ${{ github.event.pull_request.head.ref }}

    - name: comment on PR with Binder link
      uses: actions/github-script@v1
      with:
        github-token: ${{secrets.GITHUB_TOKEN}}
        script: |
          var BRANCH_NAME = process.env.BRANCH_NAME;
          github.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: `[![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/${context.repo.owner}/${context.repo.repo}/${BRANCH_NAME}) :point_left: Launch a binder notebook on this branch`
          })
      env:
        BRANCH_NAME: ${{ github.event.pull_request.head.ref }}
```        

### Use GitHub Actions To Cache The Build For BinderHub

Instead of forcing mybinder.org to cache your builds, you can optionally build a Docker image with GitHub Actions and push that to a Docker registry, so that any [BinderHub](https://binderhub.readthedocs.io/en/latest/) instance, including mybinder.org only has to pull the image.  This might give you more control than triggering a build directly on mybinder.org like the method illustrated above.  In this example, you must supply the [secrets](https://help.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets) `DOCKER_USERNAME` and `DOCKER_PASSWORD` so that Actions can push to DockerHub.  Note that, instead of your actual password, you can use an [access token](https://docs.docker.com/docker-hub/access-tokens/) — which may be a more secure option.

In this case, we set `BINDER_CACHE` to `true` to enable this option.  See the documentation for the parameter `BINDER_CACHE` in the [Optional Inputs](#optional-inputs) section for more information.

```yaml
name: Test
on: push

jobs:
  binder:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Code
      uses: actions/checkout@v2
      with:
        ref: ${{ github.event.pull_request.head.sha }}

    - name: update jupyter dependencies with repo2docker
      uses: jupyterhub/repo2docker-action@master
      with:
        DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
        DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
        BINDER_CACHE: true
        PUBLIC_REGISTRY_CHECK: true
```

## Push Repo2Docker Image To DockerHub

```yaml
name: Build Notebook Container
on: [push] # You may want to trigger this Action on other things than a push.
jobs:
  build:
    runs-on: ubuntu-latest
    steps:

    - name: checkout files in repo
      uses: actions/checkout@main

    - name: update jupyter dependencies with repo2docker
      uses: jupyterhub/repo2docker-action@master
      with:
        DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
        DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

## Push Image To A Registry Other Than DockerHub

```yaml
name: Build Notebook Container
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:

    - name: checkout files in repo
      uses: actions/checkout@main

    - name: update jupyter dependencies with repo2docker
      uses: jupyterhub/repo2docker-action@master
      with: # make sure username & password/token matches your registry
        DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
        DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
        DOCKER_REGISTRY: "gcr.io"
```

## Change Image Name

When you do not provide an image name your image name defaults to `DOCKER_USERNAME/GITHUB_REPOSITORY_NAME`.  For example if the user [`hamelsmu`](http://www.github.com/hamelsmu) tried to run this Action from this repo, it would be named `hamelsmu/repo2docker-action`.  However, sometimes you may want a different image name, you can accomplish by providing the `IMAGE_NAME` parameter as illustrated below:

```yaml
name: Build Notebook Container
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:

    - name: checkout files in repo
      uses: actions/checkout@main

    - name: update jupyter dependencies with repo2docker
      uses: jupyterhub/repo2docker-action@master
      with:
        DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
        DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
        IMAGE_NAME: "hamelsmu/my-awesome-image" # this overrides the image name
```

## Test Image Build

You might want to only test the image build withtout pusing to a registry, for example to test a pull request. You can do this by specifying any value for the `NO_PUSH` parameter:

```yaml
name: Build Notebook Container
on: [pull_request]
jobs:
  build-image-without-pushing:
    runs-on: ubuntu-latest
    steps:  
    - name: Checkout PR
      uses: actions/checkout@v2
      with:
        ref: ${{ github.event.pull_request.head.sha }}

    - name: test build
      uses: jupyterhub/repo2docker-action@master
      with:
        NO_PUSH: 'true'
        IMAGE_NAME: "hamelsmu/repo2docker-test"
```

_When you specify a value for the `NO_PUSH` parameter, you can omit the otherwhise mandatory parameters `DOCKER_USERNAME` and `DOCKER_PASSWORD`._

# Contributing To repo2docker-action

See the [Contributing Guide](./CONTRIBUTING.md).

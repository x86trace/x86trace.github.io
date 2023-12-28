---
title: "CI/CD Notes: Nest.JS + TravisCI + Docker"
date: 2020-04-25T18:00:00+02:00
draft: false
author: "Shan"

description: "A quick blog on how to make Travis CI push your App's Docker Image to Docker Hub after successful testing."
categories: ["Technology"]
tags: ["NestJS", "Travis-CI", "Docker", "Continuous Integration"]

toc:
  enable: true
  auto: true
---
<!--more-->

## Overview

I recently got into [Nest.JS](https://nestjs.com) as a Backend framework and found it more comfortable since I have worked with [Angular](https://angular.io) for a couple of years now. So I wanted to learn the framework a bit more than your usual Beginners Tutorial.

{{< admonition >}}
I am not going to dive deep into how to write Unit Tests (I am not a champ) or how to use Travis CI, but at some point it might be easier to let some automation take over your software tasks.
{{< /admonition >}}
## Learning Material

- I used the Academind 's YouTube Tutorial of Nest.JS + MongoDB to create a CRUD App for my MongoDB Atlas Cluster.

{{< youtube ulfU5vY6I78 >}}

- I stumbled upon a really good [GitHub Repository from Jay McDonial](https://github.com/jmcdo29/testing-nestjs) called `testing-nestjs` . It served as a reference to write Tests for my `ProductsComponent`

## Continuous Integration + Docker

I had some idea of using Travis-CI previous as a tool to conduct tests for code-bases but never found time to dive deep into it.

Docker has been a hot 🔥 tool around for a while and I decided upon integrating Travis and Docker with a simple aim:

> Once the tests for the API pass, build a Docker image and push it to a Docker registry.

### Travis-CI Configuration

At first glance [Travis CI](https://travis-ci.org) already had documentation for [Docker integration](https://medium.com/r/?url=https%3A%2F%2Fdocs.travis-ci.com%2Fuser%2Fdocker) and how to leverage [Travis for Pushing Docker Images to a Registry](https://medium.com/r/?url=https%3A%2F%2Fdocs.travis-ci.com%2Fuser%2Fdocker%2F%23pushing-a-docker-image-to-a-registry) which made it a tad bit easy.

However if you need to push a Docker Image to a registry you need to have a username and password or similar credentials. Exposing such credentials within a file is something one needs to avoid at all costs.

However, there is a way to pass important environment variables without having to mention them within the code base.

#### Credential Setup

Follow these steps:

1. You simply create a Travis-CI Build Project for your GitHub repository and click on __More Options__

{{< figure src="/images/technology/travisci_nestjs/travis_build_project.png" title="Travis-CI Build Project Snippet for the Github Project" >}}

2. Navigate to __Settings__ and find the __Environment Variables__ section

{{< figure src="/images/technology/travisci_nestjs/travis_env_var_section.png" title="Environment Variables Section in Project Settings" >}}

3. Add your Docker Hub's Credentials here i.e. `DOCKER_USERNAME` and `DOCKER_PASSWORD`

{{< figure src="/images/technology/travisci_nestjs/creds_travisci.png" title="Docker Credentials stored as environment variables for every Travis CI build" >}}

### The Gotcha for Docker Hub

It is easy to be fooled by assuming that our Docker Hub's password would be the same as we often use to login into the Hub's Website. However, that was not the case for me. Although I had added my Docker Hub's password within the respective environment variable `DOCKER_PASSWORD` I was still getting Access Denial upon pushing the Docker Image for my App. 

{{< admonition >}}
Example: [Build #14 for the project](https://medium.com/r/?url=https%3A%2F%2Ftravis-ci.org%2Fgithub%2Fshantanoo-desai%2Fnestjs-products-api%2Fjobs%2F679150053%23L319)
{{< /admonition >}}
{{< figure src="/images/technology/travisci_nestjs/travis_build_failure_log.png" title="Build Log which threw Access Denied from the Docker Hub" >}}

_It took me a while to figure out that Docker Hub usually will accept an __Accept Token__ as an accepted form of credential._

#### Access Tokens from Docker Hub

1. Navigate to your Docker Hub account on [hub.docker.com](https://hub.docker.com)

2. On the top right corner click on the __User Account__ and navigate to __Account Settings__

3. Navigate to __Security__ and click on __Generate Access Token__

4. Make sure to copy the generated Access Token within your Travis-CI Environment Variables Section for `DOCKER_PASSWORD`

This should now make all your Travis CI builds capable to push your Docker Images to a Registry.

### Configuration File

Travis CI file are written in YAML and are typically called `.travis.yml`

Here is mine:

{{< gist shantanoo-desai c175e2e60474298efb38ae19d2543ba0 >}}

- The `after_sucess` tells Travis then once my NestJS tests are successful, build the Docker Image

- The `before_deploy` stage tells Travis to log into the Registry. The commands within such stages are written as elements within a list for step-by-step execution. If you wish to publish your Image to Docker Hub make sure to use:
  ```bash
    docker login -u "$DOCKER_USERNAME" -p "$DOCKER_PASSWORD" docker.io
  ```

- If for any other Registry change the server name in the end.

- Restart your Travis Build or commit something to the repository to trigger a fresh new build.
- Check your Docker Hub if a new Docker Image has been pushed by Travis CI.

There you go! 🎊 Your first CI-CD DIY project.
Don't forget to add the cool Travis-CI Build Status to your Repository's README.md to flash your passing Builds.

I am not a purist so the Continuous Delivery part of pushing an Image to the Registry makes me happy. Plus, I am planning to learn how to handle deployment on Clouds so I guess I would be spared by the DevOps Lords in the Software universe.

## Resources

- [GitHub Repository for `nestjs-products-api`](https://github.com/shantanoo-desai/nestjs-products-api)
- [Travis-CI Project for `nestjs-products-api`](https://travis-ci.org/github/shantanoo-desai/nestjs-products-api)
- [Docker Hub Image for `nestjs-products-api`](https://hub.docker.com/repository/docker/shantanoodesai/nestjs-products-api)
---
title: "Create a custom Node-Red Container with Integration Testing"
date: "2022-04-10T11:27:22+02:00"
draft: false

author: "Shan"
tags: ["node-red", "docker", "docker-compose", "integration", "testing"]
categories: ["Technology"]

toc:
  enable: true
  auto: true
---
<!--more-->
## Pre-Requisites
This post assumes that the reader has an understanding about [Node-Red][1] Flow
Programming tool and wishes to work with a custom Docker Images for it.

This post also assumes the readers are familiar with python programming.

## Requirements

Since Node-Red provides a _visual_ way of connecting many components through Flows,
we might be interested in using it for our own personal stack.

If you haven't tried it out, [Node-Red Documentation for Docker][2] provides some nice
ways to getting started.

Since the Flows are rendered to a web-browser, we might want to take into consideration 
testing our custom Node-Red container when it might be integrated with other containers 
within a stack deployment.

This guide will provide a concise way of joining the dots between creating your custom
Node-Red container and testing its reachability within a `docker-compose` file using 
the [`pytest-docker-compose` suite][3].

## Follow Along

Please refer to the [GitHub Repository](https://github.com/shantanoo-desai/node-red-slim) for
the code structure and file content


## Custom Node-Red Docker Container Creation

Diving into the deep-end straight, we want to create a Node-Red custom container with 
just the following:

1. `node-red` runtime-engine
2. `node-red` dashboard

> NOTE: for this post sake, we stick to basic things, please feel free to add more things 
>       according to you needs.

### Adding Node-Red and its Dependencies

Standard practice with Node-Red lets the user add dedicated flows and dependencies via the 
`npm install` command-line. However within the [Node-Red-Docker Repo Wiki][4], you can also
add these flows/dependencies via a dedicated `package.json`.

Let's stick to this method and avoid adding things via CLI within the container.

Create a `package.json` file and add the following:

```json
{
    "name": "node-red-slim-container",
    "description": "A Slim Node-RED Docker Image running on Alpine Container",
    "dependencies": {
        "node-red": ">=2.2.0",
        "node-red-dashboard": "*"
    },
    "scripts": {
        "start": "node $NODE_OPTIONS node_modules/node-red/red.js $FLOWS --userDir=/data"
    }
}
```
Moving on to creating a `Dockerfile`

### Multi-Stage Dockerfile for a Slim Image

We will be using a [Multi-Stage Docker build process][5] to keep our image a bit _light_ on the 
size using `alpine` based containers and the official `minimal` image from Node-Red.

We will use __TWO-Stages__ in our build process:

1. prepare a base Image with `alpine`
2. use `nodered/node-red-minimal` image to install the flows/deps and prepare the `node_modules`

We then use the base Image in step 1 as a production image and copy the `node_modules` directory 
to this production image.

Our `Dockerfile` looks like the following:

```docker
FROM alpine:3.13 AS base


RUN apk add --no-cache \
            nodejs \
            npm && \
    mkdir -p /usr/src/node-red /data && \
    adduser -h /usr/src/node-red -D -H node-red -u 1000 && \
    chown -R node-red:node-red /data 

FROM nodered/node-red:2.2.2-minimal AS build

COPY package.json .

RUN npm install \
        --unsafe-perm --no-update-notifier \ 
        --no-audit --only=production

FROM base as prod

COPY --from=build --chown=node-red:node-red /data/ /data/

WORKDIR /usr/src/node-red
COPY settings.js /data/settings.js
COPY flows.json  /data/flows.json

COPY --from=build --chown=node-red:node-red /usr/src/node-red/  /usr/src/node-red/
USER node-red

CMD ["npm", "start"]
```

We start with the base image as `alpine:3.13` and prepare it by adding `nodejs` and `npm`
needing to run Node-Red. For security best-practices, we add a `node-red` user and group as
opposed to running everything as `root`. This avoids any privilege escalation within the container,
as well as misuse within the container.

Our `build` stage will install all the NPM packages in our `package.json` and then we copy the
necessary `node_modules` to our `prod` image. In the end, we will run the command as `npm start`

You can build this image locally using:

```bash
docker build -t node-red-slim:latest .
```
### Add a Healthcheck API in Node-Red

You can refer to this [Discussion Thread on Node-Red Forum][6] to create the flow, or use the following
[`flows.json`][7] file

We can check if the flow is working again but building the Docker Image and then try:

```bash
    curl -XGET http://localhost:1880/health
```
Don't forget to set the port 1880 when running the docker image e.g.

```bash
    docker run -p 1880:1880 node-red-slim:latest
```
If that works, good job! now it is time to make an integration test with python

## Integration Test of your Node-Red Image

One cool thing you can try out is the [pytest-docker-compose][3] which provides you a nice suite 
to do tests with `docker-compose`. I won't dive too deep into it, but I relied on this
[Dev.To post by Iuliia Volkova][8] for wrting a simple test that does a health check on our
`/health` API once the container is up and running.

### Testing Files

`pytest-docker-compose` is a plugin for `pytest` testing suite. The structure of your test directory
should be as follows:

```bash
.
├── README.md
├── requirements.dev.txt
├── scripts
│   ├── 00-build-test-env.sh
│   └── 01-run-tests.sh
├── test-docker-compose.yml
└── tests
    ├── conftest.py
    └── test_fixtures.py
```
Refer to [tests/integration][9] directory for file contents

The `conftest.py` is used to as fixture for the `test-docker-compose.yml` to be used for the test stack.

the `test_fixture.py` contains the test fixture to ping the `/health` API from our flow once the 
node-red container

For our case, if the `/health` API provides a HTTP 200 OK response, our test passes. You can add more 
tests depending on your requirements.

We will leverage simple bash scripts in the `scripts` directory to setup the python 
virtualenvironment as to bring the `test-docker-compose` up and run the pytest

### Additional: using this setup in a CI/CD Pipeline using GitHub

You can think of our steps we just went through as a CI/CD Pipeline. The steps are:

1. Build our custom __Node-Red__ Docker Image
2. Test Integration using `pytest-docker-compose`
3. Push the image to Docker Hub, if test passes

For the repository, I have setup a GitHub Workflow that pushes the docker image only 
when I push a Tag e.g. `v1.2.0` etc.

For this Workflow, you will need to setup the credentials of your Registry in your GitHub Secrets. 
For a sample of the YAML file, refer to this [deploy.yml][10]

## Inference

That's how you do custom node-red, docker, docker-compose and integration testing !

I am not aware of conducting tests of the individual tests of the flows generated in Node-Red, but this 
test focuses on integrating node-red with some possible stacks where different Databases are part of the
stack.

If you have comments, criticisms, feedback please reach out to me and I will be happy to learn and help!

[1]: https://nodered.org/
[2]: https://nodered.org/docs/getting-started/docker
[3]: https://pypi.org/project/pytest-docker-compose/
[4]: https://github.com/node-red/node-red-docker/wiki/Quick-Custom-Image
[5]: https://docs.docker.com/develop/develop-images/multistage-build/
[6]: https://discourse.nodered.org/t/is-there-an-http-api-where-i-can-ask-for-pings-for-integration-tests/59707
[7]: https://github.com/shantanoo-desai/node-red-slim/blob/main/flows.json
[8]: https://dev.to/xnuinside/integration-tests-for-bunch-of-services-with-pytest-docker-compose-j5i
[9]: https://github.com/shantanoo-desai/node-red-slim/tree/main/tests/integration
[10]: https://github.com/shantanoo-desai/node-red-slim/blob/main/.github/workflows/deploy.yml

---
title: "Seed Your MongoDB Container with JSON data using Docker and Docker-Compose"
date: 2021-11-09T11:00:00+01:00
draft: false

author: "Shan"
tags: ["mongodb", "docker-compose", "docker", "json"]
categories: ["Technology"]

toc:
  enable: true
  auto: true
---
<!--more-->
## Requirements

A couple of years ago I made a frontend app which rendered JSON data stored in-memory. The App provided a _check-list_ for
Quality Management called [QualiExplore](https://github.com/shantanoo-desai/qualiexplore) (under __Apache 2.0 License__).

Nothing too fancy, but we needed to make the app a bit more flexible and decided to use _MongoDB_ to render the information 
as well as make it easy to introduce new information to the JSON data that previously persisted in the App.

Fundamentally, the app should be _out-of-the-box_ usable for someone who wants to use it (with other additions currently on-going 
e.g., GraphQL backend + User authentication / authorization)

## Problem
The JSON data, that previously persisted in the app needs to be _seeded_ into the MongoDB instance once the complete stack is 
brought up.

Doesn't sound like a bit deal, should be possible to use volume mounts with docker, right? Yup, you guessed it but MongoDB doesn't
just ingest data because one mounted a volume to a specific directory. To the rescue comes `mongoimport` tool where one can load
data via CLI. Seems like our problem is solved!

We it turns out I wanted to seed data into the MongoDB in a _litte more secure_ manner aka. maybe use a username and password and a
dedicated initialized database where this imported data should persist under specific collections.

Turns out, once you figure out the `mongoimport` tool's Syntax, the next big headache was to figure out how to _securely_ pass this
information to a docker image which will insert this information into Mongo.

I generally have a stack design pattern where I try to keep changes into YAML files for an end-user minimum and let the changes reflect
through the environment variables passed to the stack. So passing the environment variables to the respective docker containers shouldn't
be a big hassle, right? _Oh! how wrong I was!_

## Approach

I decided to following the following sequential steps:

1. Bring the MongoDB container up
2. Create a docker container that is connected to the same network as MongoDB container as above that has `mongoimport` on it
3. Mount the JSON data into the container in step 2 and insert the data
4. In order to make the ingestion of data bit more secure pass the information like the database's URI, credentials via environment variables

So the initial project setup is as follows:

```bash
.
├── docker-compose.yml
├── mongo_seed
│   ├── Dockerfile
│   ├── factors.json
│   └── filters.json
└── .env
```
Here, the `mongo_seed` is the directory with the data to be ingested as well as the `Dockerfile` that will perform the necessary ingestion.

The root of the directory contains the `docker-compose.yml` for the stack and its respective `.env` file for environment variables

### Problems Faced

This requires a quick primer of Docker's `ARG` and `ENV` in context of `docker`. A decent read-through I used can be found 
[here](https://vsupalov.com/docker-arg-env-variable-guide/)

In a nutshell

> `ARG` is used during build-time i.e., when building a docker container. It's lifetime is short till the build is complete. Once the container
> is built, they are not available for further usage.
> `ENV`, on the other hand, is available for the run-time of the container too.

A common practice that a lot of container developers use is to declare an `ARG` and then use the value of the incoming build-argument as an
`ENV` for further usage.

so something like

```docker
ARG database-uri

ENV DATABASE_URI=$(database-uri)

# Use `$DATABASE_URI` to insert database

```

This wasn't the case since I ended facing problem which I documented on 
[StackExchange](https://stackoverflow.com/questions/69886273/docker-compose-file-doesnt-pick-up-environment-variable-from-a-dedicated-env-fi)

The problem concisely, is the environment variables from `.env` fille passed into the `mongo-seed` container doesn't get set during run-time

## Solution

After going through some StackExchange Queries, an answer from
[Zincfan on a Query regarding getting Env Variable in Dockerfile](https://stackoverflow.com/a/69173690/4851126) turned out to be the solution

### Steps

- Declare Same `ARG` and `ENV` of the same name as that of the environment variables
- Get the information from the build-argument and store the information into the env. variable using the syntax:
        ```
        ARG DATABASE_URI
        ENV DATABASE_URI ${DATABASE_URI}
        ```
- use `${DATABASE_URI}` in the run-time of the container
- to pass the information from the `.env` passed to the compose file should be in the following way:
    ```yaml
     services:
         mongo-seed:
             args:
                  - DATABASE_URI=$DATABASE_URI
    ```
### Files

So finally my `Dockerfile` looks like the following:

```docker
FROM mongo:5.0
 # Will be set through Environment Files
 ARG DATABASE_URI
 ARG USERNAME
 ARG PASSWORD

 ENV DATABASE_URI ${DATABASE_URI}
 ENV USERNAME ${USERNAME}
 ENV PASSWORD ${PASSWORD}

 COPY factors.json /factors.json

 COPY filters.json /filters.json

 CMD mongoimport --username ${USERNAME} --password ${PASSWORD} --uri ${DATABASE_URI} --collection factors --drop --file /factors.json && \
     mongoimport --username ${USERNAME} --password ${PASSWORD} --uri ${DATABASE_URI} --collection filters --drop --file /filters.json


```
__Good to Note__: Never split into two `CMD` in the file, rather merge multiple commands into one `CMD`.


my `docker-compose.yml` file looks like the following:

```yaml
services:
    # MongoDB
    mongo:
        container_name: mongodb
        image: mongo:latest
        env_file:
            - .env
        ports:
            - "27017:27017"
        networks:
            - "qualiexplore_net"

    # Initial Seed to QualiExplore Database
    mongo-seed:
        env_file:
            - .env
        build:
            context: ./mongo_seed
            dockerfile: Dockerfile
            args:
                - DATABASE_URI=$DATABASE_URI
                - USERNAME=$MONGO_INITDB_ROOT_USERNAME
                - PASSWORD=$MONGO_INITDB_ROOT_PASSWORD
        depends_on:
            - mongo
        networks:
            - "qualiexplore_net"
```

My `.env` file is as follows:

```
MONGO_INITDB_ROOT_USERNAME=root
MONGO_INITDB_ROOT_PASSWORD=example
MONGO_INITDB_DATABASE=qualiexplore
DATABASE_URI=mongodb://mongodb:27017/qualiexplore?authSource=admin****
```

## Comments, Suggestions

If you would like to suggest me some changes in my design or a better way to tackle this situation, I am all ears
and would love to improve and discuss aspects. You can contact me on `LinkedIn` or via E-Mail.

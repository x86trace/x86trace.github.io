---
title: "'Multi-Image' Docker Image"
date: "2022-06-24T21:02:46+02:00"
draft: false

author: "Shan"
tags: ["docker", "containers", "dotnet"]
categories: ["Technology"]

toc:
  enable: true
  auto: true
---
<!--more-->
## Problem

I was on the lookout for a solution to a problem in Docker that needed to do the following:

> Create a Docker Image for a bunch of compiled ASP.NET `.dll` files and try reducing the 
> container footprint

Sounds simple, right? But it turned out to be a bit more _messy_ than I imagined. To top it off,
I have __NEVER__ even programmed in .NET so I was feeling like a visually impaired person made to 
navigate his way through a dark room with furniture scattered all over!


## Thousand Leagues under the Container Sea!

Keeping the whole Maritime theme alive with Docker (and Kubernetes), I jumped into the sea of 
Docker Hub with millions of containers and found out that Microsoft hosts all the .NET related 
container images as [.NET by Microsoft registry][1].

Gentle reminder that I have never even thought of .NET before in my life, so just trying to figure out
whether I need an one or more of the following images:

- SDK
- ASP.NET Core Runtime
- .NET Runtime
- .NET Runtime Dependencies
- .NET Monitor Tool
- .NET Samples

was a side-mission in its own right.

### Side-Mission
A quick deep-dive into [Microsoft's Docker .NET Docs][2] made me think I only need the following docker image: 
`mcr.microsoft.com/dotnet/aspnet:6.0`

So packing everything in a simple `Dockerfile`

```Dockerfile

FROM mcr.microsoft.com/dotnet/aspnet:6.0-focal


WORKDIR /DotNetVoyage

COPY ./appsettings.json /DotNetVoyage/Server-Files/

# Copy every '.dll' file I was handed 
COPY ./* /DotNetVoyage/Server-Files/
CMD [ "dotnet", "Server-Files/Server.dll" ]
```
Building it locally and accessing the App initially on the dedicated ports seemed to work just fine and I thought 
I was getting off work a bit early today. I jinxed myself.

Upon some API testing of the image I started seems some logs that were strange to my _non .NETian brain_.

```
The specified framework can be found at:
It was not possible to find any compatible framework version
The framework 'Microsoft.AspNetCore.App', version '5.0.0' (x64) was not found.
 - The following frameworks were found:
    6.0.6 at [/usr/share/dotnet/shared/Microsoft.AspNetCore.App]
You can resolve the problem by installing the specified framework and/or SDK.
```

I was head scratching as to how I can make my runtime which is `6.0.0` potentially have an SDK version that must 
have `5.0.0` SDK on it. It got worse, I found out I needed the now _defunct and not supported_ `2.1` version of the SDK
too! 

### Build everything from scratch?

Unaware if there was a way to introduce the different SDKs into the already slimmed container image, I had lost hope 
and decided to build an image with maybe `debian` or `ubuntu` or `alpine`.

However, upon asking the Slack Community and rummaging through some StackExchange queries, I traced out a plan that was
pretty familiar to me: __Using Multi-Stage Docker Builds__.

## Charting towards a Destination

I use Multi-Stage Docker builds almost everyday for work and so I decided to design my container image as follows:

```
Stage 1: pull SDK 2.1
  1.1: find out which directory do the SDK Files persist

Stage 2: pull SDK 5.1
  2.1: find out which directory do the SDK Files persist

Stage Prod: Pull the 6.0.0 Runtime
  Prod.1: copy all the SDK Files (from Stage 1 and Stage 2) to the same directory here in Prod Stage
  Prod.2: copy all the DLL files from host and run the dedicated DLL
```

### Where art thou SDKs?

A quick pull:

```bash
docker pull mcr.microsoft.com/dotnet/sdk:2.1
```

and check within the container:

```bash
docker run -it --rm --name=sdk21-check mcr.microsoft.com/dotnet/sdk:2.1 dotnet --list-sdks
```
resulted in:

```bash
2.1.818 [/usr/share/dotnet/sdk]
```

Gotcha! the directory for the SDKs is `/usr/share/dotnet/`

### Multi-Image Image

what I mean by this is whether Docker is smart enough to directly pull an image either using `COPY` or `ADD`
instructions without me having to create `Stage 1` and `Stage 2` using syntax such as:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:2.1 as base21

FROM mcr.microsoft.com/dotnet/sdk:5.0 as base50
```

It turns out that `COPY` is extremely flexible! with the `COPY --from` instruction in the `Dockerfile`
is capable of pulling the dedicated image from a registry and the respective directories can be copied easily!

## Solution

__Final Checks for `Dockerfile`__

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:6.0-focal

# Add those missing SDKs here
## Syntax: COPY --from=<registry/image:version> image_directory ProdStage__dest_directory
COPY --from=mcr.microsoft.com/dotnet/sdk:2.1 /usr/share/dotnet /usr/share/dotnet/
COPY --from=mcr.microsoft.com/dotnet/sdk:5.1 /usr/share/dotnet /usr/share/dotnet/

# List all the SDKs / Runtimes available in the container
CMD ["dotnet", "--list-sdks", "dotnet", "--list-runtimes"]
```

build the image and run it and the output will be similar to:

```
Microsoft.AspNetCore.All 2.1.30 [/usr/share/dotnet/shared/Microsoft.AspNetCore.All]
Microsoft.AspNetCore.App 2.1.30 [/usr/share/dotnet/shared/Microsoft.AspNetCore.App]
Microsoft.AspNetCore.App 5.0.17 [/usr/share/dotnet/shared/Microsoft.AspNetCore.App]
Microsoft.AspNetCore.App 6.0.6 [/usr/share/dotnet/shared/Microsoft.AspNetCore.App]
Microsoft.NETCore.App 2.1.30 [/usr/share/dotnet/shared/Microsoft.NETCore.App]
Microsoft.NETCore.App 5.0.17 [/usr/share/dotnet/shared/Microsoft.NETCore.App]
Microsoft.NETCore.App 6.0.6 [/usr/share/dotnet/shared/Microsoft.NETCore.App]
```

Voilá ! Runtimes `v6.0`, `v5.0`, `v2.1` all in the same container !!

__Final `Dockerfile`__
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:6.0-focal

# Add those missing SDKs here
## Syntax: COPY --from=<registry/image:version> image_directory ProdStage__dest_directory
COPY --from=mcr.microsoft.com/dotnet/sdk:2.1 /usr/share/dotnet /usr/share/dotnet/
COPY --from=mcr.microsoft.com/dotnet/sdk:5.1 /usr/share/dotnet /usr/share/dotnet

WORKDIR /DotNetVoyage

COPY ./appsettings.json /DotNetVoyage/Server-Files/

# Copy every '.dll' file I was handed 
COPY ./* /DotNetVoyage/Server-Files/
CMD [ "dotnet", "Server-Files/Server.dll" ]
```

- Dedicated [StackOverflow Query I created and answered][3]

Just another day on the decks with Docker !!


Get in touch if you feel like dropping some suggestions, feedbacks or criticism! Always happy to learn and improve!

[1]: https://hub.docker.com/_/microsoft-dotnet
[2]: https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/docker/building-net-docker-images?view=aspnetcore-6.0
[3]: https://stackoverflow.com/a/72745810/4851126

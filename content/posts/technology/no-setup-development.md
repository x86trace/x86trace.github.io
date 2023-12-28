---
title: "No-Setup-Development: Productivity Experience with Docker"
date: "2022-05-08T10:26:29+02:00"
draft: false

author: "Shan"
tags: ["docker", "software-development", "node.js"]
categories: ["Technology"]

toc:
  enable: true
  auto: true
---
<!--more-->
## Why I stopped worrying about setting up environments!

If Stanley Kubrick were a Software Engineer, he would have named this post

> Dr. Nosetup: How I Stopped Worrying About Setting Up and Love the Development

(I'll see myself out with that pun!)

I tried contributing to an open-source project without actually setting up the
complete programming language tools, and it felt like worth documenting.

### Problem: So much to download, and setup before getting to work

I tried sending a feature to the [node-red GitHub Repository with a new TOML configuration node][1].

However, I didn't want to _taint_ (pardon me for using the word) my personal laptop by installing
`node.js` and `npm`.

One particular reason being, is that I have less time now to continue with Web Development stuff,
and `node.js` isn't my preferred language anyways. I want my host laptop to be as minimal as possible.

But I wanted to send the feature patch upstream because I was _in the zone_.

### Solution: Docker encapsulated Environment

Since I have been heavily using `docker` for a while now, I asked myself

1. What do I need to send a patch upstream?

A: Only relevant files

2. Does `docker` provide me a `node.js` environment?

A: Yes, certainly it does. Not just `node.js` but for all possible programming languages

3. How do I avoid doing _manual copy-paste_ labour for files between the container and my laptop?

A: __Volume-Mounts__. Any changes within the containe get reflected back to the host laptop and vice-versa


## Setting it Up!

All I really needed was `docker` on my machine and we are ready to go!

__Steps__:

1. clone the repository to dedicated directory on my host laptop

2. Visit __Docker Hub__ and find the `node-js` Image repository

3. Find the Long-Term-Support (LTS) version image tag. In my case it was `16.15.0`

So we almost have everything we want!

### Caveats

Remember that Docker Containers are their own _ephemeral_ worlds all together.

If the containers are designed to with `root` users, your files may change ownership, or might have
different owners. You could check this using `ls -la` in your directory.

I really want to avoid such scenarios, such ownership issues might affect your filesystem as well as
the upstream code. But no issues, `docker` CLI provides way to control the user and group settings 
before bringing the container up.

It is also worth mentioning that, the container environments also produce files that should be not be
reflected in your commits upstream. In the case of `node-red` the `package-lock.json` is a file created
within the container that will be mapped to the host machine.

It might be wise to keep such files into `.gitignore` as well as `.dockerignore` files within the development
repository to avoid accidently committing them upstream or bringing them within the container.

### Docker CLI

```bash
$ # assuming your are in the development repository
$ docker run -it --name=node-red-TOML \
     -u $(id -u):$(id -g) \
     -v $(pwd):/usr/src/app \
     -p 1880:1880 \
     node:16.15.0 \
     /bin/bash
```

The `-u` parameter maps your current user id and group to the container, avoiding any `root` ownership
conflicts.

The `-v` parameter is the volume mount that will map the codebase to the `/usr/src/app` directory in the
container.

There you have it ! A node-js environment without having to download and setup the tool on the host!

You can now code everything with ease with your Editor of your choice with the container running.

Any changes, either on the host or within the container will be reflected to your editor.

Just make sure to run the execution commands in the container.

## Benefits

This worked out well for me! I was able to get the codebase up and running in no time, without having to
worry about incompatibility issues.

Changes made in my editor (new files, refactored files) are available in the container to use and execute.

Running the commands within the container makes it easier to know what happens and all of this is _ephemeral_
so I don't have to do a lot of cleanup afterwards.

Just remove the container and commit the code!

On a side-note, the [feature patch upstream][1] wasn't required by the core team :(, but I could use the 
same development environment pattern to create a `node-red-contrib` node. So nothing does to waste!

Hope this helps, get in touch if you would like to provide some suggestions, criticisms!


[1]: https://github.com/node-red/node-red/pull/3599

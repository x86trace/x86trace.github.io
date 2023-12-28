---
title: "How Reproducibility + Documentation in Software goes deep: Practical Scenario"
date: "2023-06-11T20:00:00+02:00"
draft: false

author: "Shan"
tags: ["docker", "docker-compose", "vagrant", "ansible", "DevOps"]
categories: ["Technology"]

toc:
  enable: true
  auto: true
---
<!--more-->
## Problem

I have been working on a personal project which I recently open-sourced called [Komponist][1].
In a nutshell, the project is heavily reliant on some of the most widely used tools today in
the software landscape i.e., __Ansible__, __Docker__, __Docker Compose__.

Everything was _smooth-sailing_ with a new feature planned when suddenly I started getting a
strange error when bringing my container stack up using:

```bash
docker compose --project-directory=deploy up -d
```

The error was rather mysterious and not much verbose. It screamed something like:

```bash
Error response from daemon: Could not find the file / in container <container_id>
```

This seemed rather odd, the same thing that worked a couple of days back now is rendered 
buggy! As usual common searches didn't lead me anywhere. What was rather weird was the fact that
I never was mounting any root directory anywhere (`/`). The good thing about this error was
I had only two realms to look into, either __Docker Compose v2__ tool or the __Docker Engine__

### Hackin' Around

Since this was a sudden bug that I faced with, I suspected initially something going wrong with 
the Compose Files (`docker-compose.yml`) file which my tool generates. However, this felt rather
counter-intuitive because the tool itself also takes up the initiative to validate all my Compose Files.
Had there been any error, the validation logic would have failed and hence would not have generated me
a production-ready file in the first place.

Through some previous recollections, I was conscious about User ID within the container causing problems
especially when trying to mount secrets into Docker Containers. In essence, my tool mounts Secrets (credentials)
into the container via Docker Compose's specification which copies the values under the `/run/secrets` directory
within a container.

This feature, although great had some small caveats. As security best-practice most containers have a user within
them, which does _NOT_ have enough privileges to actually mount the secrets into the `/run/secrets` directory.
The only exception being container with `root` user (which is generally not the best idea in the first place).

A common fix is to add `user` parameter to each service with the current Host's User ID, e.g., `1000` which 
will run the container as the same user ID as that of the host's user. This was already part of the current system
in my project.

I started hacking around thinking this particular value might be the main culprit so I tried adding
various combinations like `user: "1000:1000"` or `user: "<my_host-machine_name>"` but it did not work.

A last dumpster dive into some GitHub Issues on the [Moby Project][2] which is essentially the Docker Engine
landed me onto [Issue 34142 for Moby Project][3] which is open since __2017__ (yes you read that right!).

At wit's end, I thought if the user on my host has stopped working, I might as well give the user within the image 
a shot i.e., if I have a Grafana container, upon running:

```bash
$ docker run -it grafana/grafana-oss:latest whoami
grafana
```
so updating my `user` parameter to `user: "grafana"` suddenly brought got everything working!

> Temporary fix for the Project! Hurrah!

## Reporting the Behaviour

I generally am very thorough in reporting bugs / issues to Open-Source Projects. Essentially I start
with mentioning my Host, all the software versions that are affected (here, Docker Engine / Docker Compose v2)
and provide a Minimum Example that can be reproduced by anyone. Since the fix was pertaining to Docker Compose v2's
`user` property I reported a bug on [Github's docker/compose repo][4].

The Maintainers had a look into it, but then came the nightmare answer of:

> Tried to reproduce it, I got something else !

### Dread of "It works on my machine, can't reproduce it"

After a few back and forth with the maintainer and even trying out certain examples, it felt like I had messed up
my Docker Engine with some configuration. It wasn't the case, since my configuration file was the least bit 
complicated and it was close to running an _out-of-the-box_ Docker Engine.

It seemed like this was going to be one of those GitHub Issues that would be left open and gets lost in the 
Oblivion of Open-Source Projects. Everything seemed haywired.

### One Last Effort!

Whilst the back and forth between my Bug report and the Maintainer's non-reproducible errors started to reduce,
I reverted back to the an older Docker Engine Version to see if a sudden update started to cause this error on my end.

For Context, the Issue was about using __Docker Engine v24.x__ with __Docker Compose v2.18.1__. I downgraded the 
Docker Engine to the last __v23.x.x__ and would you know it! Things started to work again. 

I was able to pinpoint out that things were going wrong, since the _leap of faith_ from Docker Engine __v23__ to __v24__.
I mentioned this difference in the [Issue's Thread](https://github.com/docker/compose/issues/10663#issuecomment-1580624862)

So I was convinced that something is breaking between the Docker Engine versions but I wasn't sure whether there was 
some discrepancy in Docker Compose v2 too.

This led me to a previous Compatibility Anaylyses that I had done about a year back called 
[Exploring the Marshland: Docker Compose Versions](https://shantanoo-desai.github.io/posts/technology/compose-incompatibility/)
where I was able to isolate different versions of the software to determine which versions were working with what
specific paramters. The tool I had used was __Vagrant__.

## Isolate, Compare, Determine!

I took the initiative of creating two __Isolated Vagrant Boxes__ with the following software:

| Vagrant Box | Docker Engine Version | Docker Compose Version(s) |
|:-----------:|:---------------------:|:-------------------------:|
| __Box 1__   | `v23.0.6`             | `v2.18.1`<br> `v2.17.3`<br> `v2.16.0` |
| __Box 2__   | `v24.0.2`             | `v2.18.1`<br> `v2.17.3`<br> `v2.16.0` |

### Documenting the Discrepancy!

I decided that I would rather add GIFs of the reproduction of the errors as opposed to adding a lot of
verbosity to an already complicated problem and create a repo that would reduce the time for the Bug reproduction.
This does come with an assumption that the Maintainer would also have __Vagrant__ or something similar to test at their
end.

I was able to reproduce the same bugs in a rather isolated environment using Vagrant disproving my theory that I had 
messed my Docker Engine up and perhaps the Maintainer should try the stuff out again at their end.

The Repo can be Found on [GitHub](https://github.com/shantanoo-desai/docker-engine-secrets-error)

## Inference

Upon providing an environment to reproduce this error as well as some GIF documentation, it turned out that this was
actually an error even the Maintainer can reproduce. This error is related to the Docker Engine v24 and not Docker Compose v2
and it is now being tracked with fixes already on the way.
See [Moby/Moby Issue #45719](https://github.com/moby/moby/issues/45719).

This might look like a lot of work, which in hindsight might be, but it is worth being comfortable with tools where a decent
amount of reproducibility via Isolation can provide a strong foundation to look into software a bit more deeper!

Vagrant is meant to be Developer-Friendly tool which indeed helped me overcome the dreadful situation of 

> It worked on my machine, why isn't it working on yours?!

It also helped collaborate on a problem which stemmed not in Docker Compose v2 but went all the way down to Docker Engine
itself. The maintainer was able to produce the same error with [Canonical's multipass](https://multipass.run/) which also
reached the same goal through a different route.

So in a nutshell, a few hours of going in the deep end of error reproducibility may feel like time wasted but it might
just solve a problem that is long-lasting (till something new pops up!)


[1]: https://github.com/shantanoo-desai/komponist
[2]: https://github.com/moby/moby
[3]: https://github.com/moby/moby/issues/34142
[4]: https://github.com/docker/compose/issues/10663

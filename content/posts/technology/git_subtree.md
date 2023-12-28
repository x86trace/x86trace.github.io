---
title: "subtrees in Git: How to split Directories into individual standalone repositories"
date: 2021-04-02T17:31:05+02:00
draft: false

author: "Shan"

description: "How to avoid a mess when a large Repository needs to split into smaller, standalone repositories"
categories: ["Technology"]
tags: ["git", "software-project-management"]

toc:
  enable: true
  auto: true
---
<!--more-->

## Problem

You start working on an idea without having to bother if the project needs to split into smaller components, projects.
Once the vision starts becomes clear and the horizon becomes clear, it makes sense to start thinking into managing components as their
own standalone software repositories.

One does not want to put a lot of effort towards this _migration_, and not mess the History your `git` repository has. Credit from your
colleagues should be manifested fairly in the new repositories.

How does one achieve this?

### Current Structure

```bash
$ cd big-repository
# MEGA-REPOSITORY
.
├── SubProject1 # this should be a standalone repository
│   └── SubModule1
└── SubProject2 # this should be a standalone repository
    ├── SubModule1
    ├── SubModule2
    └── tests
```

## Solution

The great thing about `git` is it diversified eco-system. The problem of making directories their own git repositories is something a large
pool of IT Developers face. The solution is to make the subprojects into a `subtree`.

### Usage

```bash
$ git subtree split --prefix <your_directory_name> --branch <branch_name>
```

here `branch_name` can be whatever you wish it to be, `your_directory_name` here in my case is `SubProject1` and `SubProject2`.

```bash
$ git subtree split --prefix SubProject1 --branch micro-project
```

git will make a branch called `micro-project` and check it out for you. Upon listing the files and directories it will only show the content of 
the directory mentioned (here, contents of `SubProject1`)

Similarly for `SubProject2`

```bash
$ git subtree split --prefix SubProject2 --branch nano-project
```

## Setting Up Repositories

1. Initialize your Repositories on your Git Server / Platform (GitHub, GitLab etc.)

2. Create a new directory

        $ mkdir micro-project/ && cd micro-project/

3. Initialize the directory as a git repository

        $ git init

4. Pull the subtree branch in the `big-repository` to the current directory

        $ git pull ../path/to/big-repository <branch-name>

    In our case for `micro-project` the branch name is `micro-project`

        $ git pull ../path/to/big-repository micro-project

5. You can verify if all the commit logs are still intact in the new repository using:

        $ git log --oneline

6. Add a `remote` to your new Repository using:

        $ git remote add origin <YOUR_GIT_SERVER_URL/micro-project.git>

7. Push to code to the Server:

        $ git push -u origin master

Continue the steps for `nano-project` too.


## Sources

- [Paul Cochrane's Blog Post on Moving Files and Directories to a New Repository in Git](https://ptc-it.de/move-files-to-new-repo-in-git/#moving-just-a-directory)
- [Fantastic StackOverflow Answer from coolaj86 titles The Easy Way](https://stackoverflow.com/a/17864475/4851126)

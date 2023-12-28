---
title: "How to (atleast) not get downvoted on StackExchange or anywhere: My Bug/Query Template"
date: 2021-07-15T15:08:58+02:00
draft: false

author: "Shan"
tags: ["software-bugs", "bug-reporting", "stackexchange", "stackoverflow", "documentation", "templates"]
categories: ["Technology"]

toc:
  enable: true
  auto: true
---
<!--more-->

{{< admonition type=warning title="Disclaimer" >}}
Everything mentioned in the blog are my own findings / discoveries to survive the often times _brutal_ down-voting of 
StackExchange / StackOverflow Queries. Please feel free to ignore any or all of this if you have a better way of documenting
errors / bug reports.
{{</ admonition >}}

## Problem

For someone who does not have an education in Software Engineering but yet loves writing Software, I frequently, if not always, rely on
the community of Software Developers / Engineers to help me figure out my problems with programming languages or software suites.

One of the major problems when I started working with code-bases / stacks (in 2015, not so old as I want to make it sound!) was the lack
of coherence in trying to frame my problem which I faced whilst working on something. The frustration that mounts on when you are deep-diving
into something and hitting a temporary barrier is sometimes baffling.

Now after 6 years of working with software and [2.4k Reputations on StackExchange](https://meta.stackoverflow.com/users/4851126/shan-desai?tab=profile) later, I looked back at why some of my posts keep getting me upvotes even though they were posted a while back!


## Bragging Time

At the time of writing this blog, I checked my reputations and I am ranked at:

- __Top 11% for this year__ on StackExchange
- __Top 7% Overall__ on Internet of Things Beta Community
- __Top 9% Overall__ on Rasbperry Pi StackExchange Community
- __Top 26% Overall__ on Unix Linux StackExchange Community


One might be tempted to think, that these numbers are weak and do not amount to anything which is absolutely fair to assume. However, please do
realize that for someone not with hardcore software background, surviving the _down-votes_ on such communities is an achievement in itself.

## Analyses

These are some of the things that have helped me over the years in obtaining answers to queries on forums and also a positive reputation

### Mark (this) Down

This is a rather software _philosophy_ that one can debate till the end of time or till the end of a project!

Documentation is not for everyone ! I hardly have met someone who is as much in love with __Markdown__ as I am at this point of my professional career. Don't believe me ? I am writing this blog post in __Markdown__!

I am mentioning Markdown because it is exceptionally handy to work with. From a blog post such as this, to StackExchange, to different community websites and even Reddit support it in some form or another. Not to mention how minimal it is compared to terrible HTML tags !

It also has some of the lowest learning curves for any Markup Language to my knowledge. 

> May it be opening an Issue on GitHub, or writing a blog post, or an opinionated post on your favourite subreddit - Markdown is always there to help you transfer your thoughts for you without having to click on that fancy editor buttons ! 


### Start a project with a NOTES.md

you can ignore this file in your software development and use this `NOTES.md` as a scribbling markdown file for all things you did/ you want / you will need to do in order to get something up and running.

Great thing about making this a markdown file is the flexibility of copying / pasting commands and code snippets that are suspicious.

A __CTRL + C__  and __CTRL + V__ is way comfortable that a million __CTRL + Z__ a lot of times ! 

### Being Concise

This is the hardest thing I had to adapt. In a frustrating bout with some code or a stack, I would have just loved to dump everything that has been through at my screen on a forum and wait for someone to catch my attention. However, life isn't like that and so is software troubleshooting!

Things that can help be concise:

1. Keep a small notebook to help keep track of basic Command-Lines executed (better yet use `history` command to know what was done)
2. If you don't care about jotting things down, try keeping a mindful state when developing in order to trace back some vital steps which may lead to failure.

Depending on your jotting down skills / recollection of your mind's memory you can use this information to summarize what lead you to the problem / bug in the first place and place those information in your `NOTES.md` for posterity


### Document your working system

Not everyone has the same computer / server etc. as yours ! So it is definitely worth keeping track of which OS you have (which build, distribution, etc.) to make it clear to someone that things may be a bit different.

Same logic goes for your programming languages. Each programming language's compiler, interpretor keeps updating itself and it is wise to provide someone willing to help you out with a bit more precise information on how you got started with your work.

Same logic should go for your Dependencies. A lot of programming environments provide a file that keeps track of all your dependencies, and although some are uglier than another (looking at your Java's `pom.xml` or other `.xml` files), it is worth mentioning the dependencies that you might have added recently that you suspect might have broken your code

Maybe a `git diff` can help you get an idea of what was recently added or changed.

### Copy / Paste all your URLs

Seems harsh but, if you browsed through some handy tutorial / blog / StackExchange Response, copy the URL in your `NOTES.md` or bookmark it!

We all suffer from the StackOverflow Copy/Paste Syndrome and in a frustrating event don't realize where and why we changed a bit of our work just get things up and running!

Adding URLs also is extremely beneficially for posterity when someone who wishes to pursue something similar to your work can benefit from it!

### Do not write massive paragraphs

My observations over the years is that a lot of communities don't care about a _Shakesperean_ Description of your problems you face ! Instead they prefer to read shorter sentences since they too want to get to the bottom of the mystery that you have posted on.

Granted, this strategy does not always work however it gets the ball rolling and further information can be requested in the comments sections.

### Do not Paste your massive code snippet

Let's be honest, not even your QA manager wants to dive deep into your 1000+ lines of code. There is most certainly a certain block of code you feel is the culprit and this can be pasted in the query in order for the readers to only focus on the things that are causing a problem.

### An Updated Query Provides More Context

Apropos, if someone asks you to provide more information for your query then a better way to address the additional information is writing it under a new section. A lot of forums provide an __Edited View__ on the query but a lot of times I find them to be let intuitive about the added information.



## What got me the Up-Votes?

At this point, you might be like:

> Yeah, Yeah ! Show me something substantial and enough with the words already!

Well here is a template I think might help people (new-comers/ beginners) out!

Click on the code snippet below which is already in markdown and add the information accordingly.

Please feel free to add / remove any other information that you might find useful (Let me know so I can improve myself too!)

### Template

This is how I have gotten around to structure my queries mostly!

```markdown
## Host System
- Mention Operating System
    - Mention Build Version / Kernel Version / Distribution

## Programming Language / Software Suite / Software Tool Environment Information

- Compiler / Interpretor Version (mostly available via `--version` flag or `version` parameter)

## Dependencies Graph

- Provide a list of changed dependencies (if any that you think might be the culprit)

## Problem Description / Scenario

Provide the following:

- I want to --INSERT Intent HERE--
- Upon following the step (mentioned below) I end up with
    --- PASTE ERROR LOGS here ---

### Provide a repo overview (if necessary)

Sometimes the repository structure may be helpful to get a better picture 
(Calling modules from different directories etc.)

- Use `tree` command-line to provide a necessary structure of your repository

### Code Snippets (If any, else use Reproduction Steps)

Remember not to dump all the 700 Line code here. 

Add a particular part of the code that you think may be the culprit

Add a note in the end or as a comment in the code snippet `// removed code for brevity` 
to let the reader know that you are focussing on what you think is causing a problem

### Reproduction Steps

Here you can actually make a list of all the command lines
you have executed in order to provide a better context.

A lot of times the error might lie in some of the steps executed here (missing flags etc.)

## Queries / Requirements

Add a list of questions you think you are unable to figure out from the problem 
or a list of requirementst that your code should be able to achieve.

Do not add a massive amount of information here, try to be concise as much as you can.

## Edit (only necessary if requested)

Add a list of updated information here, only if the community requests its.

It is easier to add information here than
rummaging through the comments under the query.
```


## For Posterity and beyond

I am aware that not a lot of my code will be super helpful to people however in terms of __Technical Debt__, I understand that finding a solution to a problem is not an easy task, however an even harder task is to take the effort for documenting it and making it available for others!
How great does it feel when, you stumble across a query that is very similar to what you are facing and there is an Accepted answer for it!

What also works incredibly well is to easily address the query and the solution in your Software code. You do not have write the complete bug description and the decision to solve the bug would already be mentioned in the URL of the Query. The URLs also work great for a company's Knowledge Base / CMS.

What also works well for me is going back to a particular bug I had previously documented on StackOverflow / StackExchange is quite easy. A few clicks from my user account and voila! I land up to the solution as well as related queries to the topic in no time.

Last but not the least, to be honest, everyone would love to have a quick Dopamine fix when they see an upvoted query or an answer !

## Show and Tell

After all this, do I have something to back my claims?

Yes I do!

Here is a list of all the Queries and Answers that have gotten me a lot of upvotes over the years and stick more or less to the template mentioned above:

__StackOverflow__

1. [Write a Recipe in Yocto for Python Application](https://stackoverflow.com/questions/50436413/write-a-recipe-in-yocto-for-a-python-application/52643185#52643185)

2. [Passing NODE_ENV into npm script for Windows 10](https://stackoverflow.com/questions/57589183/passing-node-env-into-the-npm-script-for-windows-10)

3. [Editing an already existing patch file](https://stackoverflow.com/questions/52928799/editing-an-already-existing-patch-file)

4. [Perform Nested Query in Python Graphene](https://stackoverflow.com/questions/57313080/perform-nested-query-in-python-graphene-with-distinct-types-in-schema/57326796#57326796)


__Internet of Things (Beta)__

1. [Make Embedded Board NTP Server for Arduino Boards in the subnet](https://iot.stackexchange.com/questions/3592/make-embedded-board-ntp-server-for-arduino-boards-in-the-subnet)

__Raspberry Pi__

1. [Which Model Raspberry Pi am I running](https://raspberrypi.stackexchange.com/questions/61699/which-model-raspberry-pi-i-am-running)

__Unix / Linux__

1. [systemd service does not trigger tmux command](https://unix.stackexchange.com/questions/444363/systemd-service-does-not-trigger-the-tmux-command-upon-reboot/444390#444390)

2. [Is it possible to use commands like `fg` in a shell-script](https://unix.stackexchange.com/questions/354673/is-it-possible-to-use-commands-like-fg-in-a-shell-script)


If you have tips that you feel I might have overseen or something that needs to be corrected than let me know! I am all about improving!
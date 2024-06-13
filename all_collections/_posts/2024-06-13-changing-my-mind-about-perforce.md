---
layout: post
title: 'Unreal Stack: Changing my mind about Perforce'
created-at: 2024-06-13
modified-at: 
categories: ["Unreal Engine", "Perforce", "Helix Core", "Git LFS"]
---
## TL;DR

Stopped using Perforce, started using Git LFS

## What happened?

Over the past few days I installed on-premise Perforce environments containing [Helix Core](https://www.perforce.com/products/helix-core) and [Helix Swarm](https://www.perforce.com/products/helix-swarm) on both Azure and AWS.

I was unable to make it work on Azure, kept trying to connect to the server using [P4V](https://www.perforce.com/downloads/helix-visual-client-p4v), but it seems that there was something refusing the connection on Azure (checked the VMs internally, and everything seemed to be fine). This is not exactly Perforce's fault, but it did leave a weird feeling about it all.

I was finally able to make it work deploying it to AWS. Worked essentially out of the box, all I had to do was allow my IP on the machine's security group.

But my problems really started when I was finally able to connect the P4V client to the remote server. It was very laborious to set-up account access, configure default streams, create workspaces, ...

I also had to directly access the VM using SSH to modify the server TypeMap, which is:

> "... a server-wide setting that determines how files are handled by the server. ..."

All of this to then start worrying about learning the specificities of how their versioning system works on a old and laggy tool.

I'm not saying that it is a bad tool overall, there is probably a reason why it is widely used across multiple game dev companies.

But I am looking for something easy to set up and use, so I can focus on what matters.

## Solution

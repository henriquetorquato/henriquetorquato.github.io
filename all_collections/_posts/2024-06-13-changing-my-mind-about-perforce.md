---
layout: post
title: 'Unreal Stack: Changing my mind about Perforce'
created-at: 2024-06-13
modified-at: 
categories: ["Unreal Engine", "Version Control"]
---
## TL;DR
Stopped using Perforce, started using Git LFS on GitHub

## What happened?
Over the past few days I installed on-premise Perforce environments containing [Helix Core](https://www.perforce.com/products/helix-core) and [Helix Swarm](https://www.perforce.com/products/helix-swarm) on both Azure and AWS.

I was unable to make it work on Azure, kept trying to connect to the server using [P4V](https://www.perforce.com/downloads/helix-visual-client-p4v), but it seems that there was something refusing the connection on Azure (I checked the VMs internally, and everything seemed to be fine). This is not exactly Perforce's fault, but it did leave a weird feeling about it all.

I was finally able to make it work deploying it to AWS. Worked essentially out of the box, all I had to do was allow my IP on the machine's security group.

But my problems really started when I was finally able to connect the P4V client to the remote server. It was very laborious to set-up account access, configure default streams, create workspaces, ...

I also had to directly access the VM using SSH to modify the server TypeMap, which is:

> "... a server-wide setting that determines how files are handled by the server. ..."

All of this to then worry about learning the specificities of how their versioning system works on a old and laggy tool.

I'm not saying that it is a bad tool overall, there is probably a reason why it is widely used across multiple game dev companies.

But I am looking for something easy to set up and use, so I can focus on what matters: Game Development.

### A side note on implicit costs
One of the reasons that I gave Perforce preference was based on the [pricing info page](https://www.perforce.com/resources/vcs/helix-core-pricing), that showed a free tier "cloud" option:

![Helix Cloud pricing info](/docs/assets/2024-06/2.png)

But I failed to consider that this was only for "Deployment", and we would still have to pay a steep price for a per-user license (around 39 dollars).

## Solution
Because of all these problems, we decided to go with the good "old" Git with LFS, [so we can avoid the repository turning into a 200 GB mess](https://stackoverflow.com/a/35578715). 

We are using a Free tier GitHub organization, meaning that there is no set up time or upfront work.

There are some costs associated using LFS, but they are minimum in comparison to self-hosting 2 VM's or paying for the [SaaS version of Core and Swarm](https://azuremarketplace.microsoft.com/en-us/marketplace/apps/perforce-hcc.perforce-helix-core-cloud?ocid=web-pricing&tab=PlansAndPrice).

The costs are basically data packs for storage on GitHub. It goes for 5 dollars for every 50 GB you use, and it will probably take some time until we get over those first 50 GB, so we are pretty comfortable with 5 dollars/month price tag for now.

![GitHub LFS Data Plan Billing](/docs/assets/2024-06/1.png)
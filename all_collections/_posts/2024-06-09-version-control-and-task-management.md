---
layout: post
title: 'Unreal Stack: Version Control and Remote Team Collaboration tools'
date: 2024-06-09
categories: ["Unreal Engine", "Perforce", "Helix Core", "Jira"]
---
## TL;DR

Jira & Perforce's Helix Core

## Why is this a problem?

This is not exactly a hard topic to make a decision on. Worst case scenario you can always fall back to using Git for version control and [Trello](https://trello.com) (or any other free tool) for team collaboration.

The biggest problem here is that while working with Unreal Engine, we are working with some big files, and some file types that are not supported by Git. Also, not every free collaboration tool has a seamless integration with the version control system.

## Version Control
### Perforce: Helix Core

If you type "version control unreal engine" on Google, the first link that shows up is for [Perforce](https://perforce.com/). And for what I could see by watching YouTube recordings of Dev talks, this is the most used tool by game development companies. Which means that it probably has a wide support and documentation, and most importantly, [it has a seamless integration with Unreal Engine](https://www.perforce.com/p/products/vcs/free-version-control-unreal-engine).

The main tool here is [Helix Core](https://www.perforce.com/products/helix-core). It is their version control tool, with both on-premise and cloud options. And what it is in my option the best part: is is free (including cloud services) for teams up to 5 people.

- [Perforce Game Dev Stack](https://www.perforce.com/solutions/game-development)
- [Helix Core Pricing](https://www.perforce.com/resources/vcs/helix-core-pricing)

### Unity: Plastic SCM

Plastic SCM was not always owned by Unity. It was acquired and integrated on Unity's own DevOps stack. But is is possible for you to use the version control service without having to work with Unity directly. 

The big problem for me here is that it feels very "shady". Using it to version code made on Unreal Engine feels like mixing everything up. Also there is not free pricing tear, it is always "pay as you go" with first 3 members and N GB usage free. The website feels kinda weird to navigate, taking me a considerable amount of time to find more details on the pricing model and the free options.

To be honest, I would definitely give it a good consideration if I was using Unity to work on my projects. But due to "vibe" issues, I prefer will give preference to Perforce's Helix Core.

## Remote Collaboration Tools

Like I mentioned, you can use whatever method you want. But for my specific needs I want something that can:

 1. Seamless link a task to a code change so I can easily track why that change was made;
 2. Has a top-down structure that I can use to categorize different tasks into different areas;
 3. Visual and easy to use/learn;
 4. Free.

Due to my professional experience using [Jira](https://www.atlassian.com/), it was the first place I looked, and was pleasantly surprised to learn that it has free tier for up to 10 users. So I didn't really went looking around for other options.

This checks points 2, 3 and 4 for me.

And with a little amount of search, I found out that [there is a good integration between Jira and Perforce](https://www.perforce.com/integrations/jira-and-perforce-integration).

Checking point 1.

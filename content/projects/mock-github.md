---
title: "mock-github"
draft: false
description: "A bunch of tools to create a local github environment"
tags:
  - open for contribution
  - github actions
  - testing
github: https://github.com/kiegroup/mock-github
---

Simply using **act-js** to test your GitHub Actions locally was not enough. I needed a set of tools to help me create a local GitHub environment in which I could run **act** as well as test some components of my actions individually. So I created a Node.js library called **mock-github**!

### Features  
- **Fully functioning, completely local git repositories:** : You can create git repositories whose remote origins are completely local. You can push, pull, merge and perform any other git actions on them.
- **Local artifact server:** You can launch a completely local artificat server that mimics the GitHub artifact service.
- **An Octokit like client to mock GitHub APIs:** Moctokit is a client that helps you mock GitHub APIs in a fine-grained manner. This client is designed to have an interface just like the GitHub's official API client.

There are many more features that this library offers. Feel free to Check out the [docs here](https://github.com/kiegroup/mock-github#mock-github)

![moctokit](/img/mock-github.gif)

### What made this project interesting?  

What really got me excited was creating a client similar to Octokit, but with the ability to mock GitHub API calls instead of actually executing them. As I dug deep into the Octokit library, I quickly realized that I just needed to crack four key components: a code generation script, type definitions, a generic request interface, and a generic response interface. With all these pieces in place, I was able to generate all the Octokit methods along with all their typings. Pretty cool!

Another interesting aspect about this library was recreate the GitHub artifact service locally. Turns out OSINT (Open Source Intelligence) can be applied beyond cybersecurity! A bunch of people have successfully figured out how the GitHub artifact service works, which allowed me to recreate it in a manner that seamlessly integrated with my library.

### Conclusion

Sometimes all you need is a code generating script, type definitions and OSINT!

#### Contributing

This project is open for contribution! Feel free to open new feature requests, issues and PRs!  
Project link: [https://github.com/kiegroup/mock-github](https://github.com/kiegroup/mock-github)
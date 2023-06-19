---
title: "act-js"
draft: false
description: "A library to run GitHub Actions locally and programmatically"
tags:
  - open for contribution
  - github actions
  - testing
github: https://github.com/kiegroup/act-js
---

Writing custom GitHub Actions and workflows to automate some of the boring manual work is all fun and games until you have to test them without ruining your repository. Testing typically involves pushing your workflows to GitHub, triggering them, manually verifying the logs, correcting any mistakes, and repeating the process until you get it right.

Oof, so much manual work when we didn't want to do any :weary:

**Introducing act-js**, a Node.js library that enhances the [nektos/act CLI tool](https://github.com/nektos/act) by enabling you to run your GitHub Actions locally and programmatically.

### Why not simply use nektos/act
While nektos/act is a popular tool for running GitHub actions locally, act-js provides additional functionality and flexibility. Here's why it stands out:

- **API Interaction:** act-js offers an API that allows seamless interaction with nektos/act programmatically. This allows you to use testing frameworks such as jest to easily test your workflows.
- **Mocking APIs:** You can mock APIs, such as the GitHub API, during workflow runs. This feature lets you provide different inputs and create various test scenarios for your workflows.
- **Mocking Steps**: act-js allows you to mock specific steps of a workflow. This feature is useful when you don't want to execute certain commands, such as `npm publish`, while testing your workflow.
- **Workflow execution results that can be checked programmatically:** The results returned back by act-js can be used to programmatically compare against the expected output and can be used with common testing libraries such as jest.

There are many more features that this library offers. Feel free to Check out the [docs here](https://github.com/kiegroup/act-js#act-js)

![Result of running automated tests on a workflow](/img/act-js.png)
*Results of running automated tests on a workflow using act-js*

### What made this project interesting?  
The most interesting part of this project was figuring out how to mock APIs during workflow execution. The challenge stemmed from the fact that a workflow can execute any command or program that internally could be using any client (such as curl, ajax etc) to make an API call. It was impossible to implement this on a client level basis like, as some libraries like nock do.

That's when I thought was taking a more broader look - act-js uses nektos/act to execute workflows, which in turn uses containers. Hallelujah!

With containers, I have some control over the environment in which these calls are being made. If I could somehow find a way to route all outgoing calls from the container to a server I control, then I can control the response. So essentially, I had to setup a forward proxy for any container spun up by act. 

Great, looks like problem solved? Well not so fast. While researching forward proxies, I discovered that `HTTPS` requests configured to use a proxy issue a `CONNECT` request to the proxy first in order to establish a TLS tunnel to the destination, and only then do they send the actual request to the destination. Unfortunately, this means that the proxy won't know the details of the request, making it unable to respond correctly if the request needs to be mocked.

I considered and tried the following options:
1. Changing the IP table rules for the container to somehow downgrade all `HTTPS` requests to `HTTP`.
2. Adding a nginx proxy that somehow downgrades all `HTTPS` requests to `HTTP`.
3. Setting up a MITM (Man In The Middle) proxy whose certificates are acceptible to the container.

None of the above option worked well (except for option 3, but I decided it might not be worth the effort right now; this is something to consider in the future). Fortunately, I discovered that not all clients are made in the "nice" way where they always issue a `CONNECT` request for `HTTPS` calls. In fact some of them force the request be downgraded to `HTTP`. So the solution was simple:
- Create a forward proxy that will handle the mocking of APIs.
- Configure the container to set the `HTTP_PROXY` and `HTTPS_PROXY` environment variables.
- Assume that most clients respect these environment variables and send all requests through the proxy.
- Hope that most of the clients aren't made in the "nice" way or the user has someway of controlling the base url of the API calls they want to mock.


### Conclusion

This was truly an interesting journey. I did not expect to find myself exploring and analysing such lower level networking concepts and solutions.Although I ended up discarding several interesting solutions due to time constraints, I do hope that I get some time in the future to implement the MITM proxy that would allow act-js to mock any API requests without depending on the good nature of various clients.

#### Contributing

This project is open for contribution! Feel free to open new feature requests, issues and PRs!  
Project link: [https://github.com/kiegroup/act-js](https://github.com/kiegroup/act-js)

---
title: "Testing Terraform Modules using Github Actions"
date: 2021-12-20T12:15:42+01:00
draft: true
---

I've long been an advocate of test-driven development for infrastructure code, especially when dealing with infrastructure code _composition_, meaning, sourcing your infrastructure code definitions through third parties or via a distributed system of self-contained components. In the world of modern infrastructure code development with Terraform that's done using Terraform's module system (and to a certain extend, its providers as well). The [additional benefits of TDD are well known](), and while I cannot overemphasize how much of a game-changer TDD for infrastructure can be, that would be the topic of an entirely different article all together.

Instead, I want to focus on walking through the steps required to test your infrastructure provisioning code using GitHub ACtions, Microsoft's latest CI addition, which is natively integrated into the world's largest source code hosting platform, GitHub.

I will be using my own Terraform module for provisioning an OIDC provider on AWS as an example.

## Challenges

### The testing conundrum

The module in question talks directly to the AWS cloud, which means, in order to run _meaningful_ tests against the definitions it will have to talk to the AWS API in some shape or form.

Folks have been struggling to define the boundaries between unit-, integration- and end-to-end-testing for Terraform since a while now, and I personally don't believe the discussion is as important as people make it out to be. Much rather, testing _something_ is more important than testing nothing at all. It also doesn't help that Terraform, through its declarative nature, doesn't lend itself all that much for what people would consider "proper" unit-testing, or even integration-testing, since isolating components rarely ever is trivial, or even possible at all. Terraform, in most cases, is just an orchestrator for other components interacting with one-another (e.g. HCL2 interpreter, providers, inputs, outputs etc), and as such the resulting actions executed against foreign components or APIs rather difficult to isolate or ignore.

For the sake of this article I will simply ignore the premise that the testing pyramid has to be strictly observed to keep tests relevant, and much rather focus on the usefulness and utility of the tests your write for infrastructure as code, as in, it can replace and enhance a lot of otherwise manually executed, brittle tests, which rely on changing input and outputs and rarely ever are deterministic.

### Talking to the cloud

Accessing the API of any modern cloud requires pre-shared credentials of some sort, some more, some less secure in their implementation. AWS has a rock-solid authentication and authorization service, with two pre-shared credentials commonly used for authenticating to the cloud, the access key ID and a secret access key, both of which are required to generate a short-lived session token, which is then used for subsequent requests.

In an environment like GitHub Actions that means sharing such credentials, which have to be associated with an actual user account, with the environment of each process that's supposed to execute anything targeting to the AWS API, commonly in GitHub Actions Secrets. However, that opens a whole can of worms you should rather want to avoid!

### Security implications

Pre-shared keys or credentials have one major loophole: they may leak at some point in time, which means a potential attacker gains access to whatever environment the credentials allow access to, with the same rights and privileges originally intended for the initial recipient.

Why this is a major issue for a CI environment such as GitHub Actions is fairly trivial to see.

Let's take the example of a pull request, submitted against the repository the Terraform module lives in. In order to ensure the pull request doesn't break any specific functionality the module comes with a test-suite written in Golang, using a framework called Terratest. For reasons I mentioned above, these tests are executing the module code in the context of the repository, and therefore have to talk to the AWS API to ensure the expectations for the module's functionality are met. But that's a problem.

Since the testing code has to run with privileges associated with the pre-shared AWS credentials inside the pipeline, it exposes this context to anyone messing with the code, either in a pull request, or even through direct interactions with the code base. In either case, there is little control over what will happen during the execution of the tests, since a pull request might feasibly add new test-cases, or modify existing cases in order to fix a bug. As you may see, key extraction in this case would be a fairly easy endeavour, especially since we have the entire power of a high-level programming language at our disposal!

The common mitigation here would be to scope the credential's access permissions to only allow for a limited blast radius, but any exposed secure environment is one environment too many!

Additionally, having shared, long-lived credentials associated with any environment potentially running compute resources is a security nightmare, and the general recommendation is to avoid shared credentials and/or rotate them frequently to ensure the risk of exposure is minimal. That would mean regularly rotating the access keys that are shared with GitHub Actions, updating them in GitHub Actions Secrets, and making sure the pipeline still runs after having update the credentials. That's a lot of work!

Ironically, the module in question enables you to "solve" the challenge of pre-shared secrets entirely by creating an OIDC provider inside your AWS account, which allows for GitHub Actions to seamlessly acquire short-lived, scoped credentials. It doesn't eliminate the potential exploitation via malicious pull requests, but at least you wouldn't be able to leak these credentials beyond their initial lifetime (which is an hour by default).

The brings me to the next challenge, though.

### Costs

Almost anything running in the cloud costs money. "The cloud is just someone else's computer", as they say, and that someone else like to charge a buck or two for you using their resources.

While most cloud providers offer, more or less, generous "free tier" packages for users and companies, the prospect of executing tests against cloud APIs frequently and indiscriminately will rake up costs very soon. Most people would then go ahead and limit test executions to a few approved workflow instances (e.g. when cutting a release), but that would defeat the purpose of the test suite entirely, since it is supposed to ensure that expectations are met with every single bit of code that's either added, changed or removed. The feedback loop for verifying your expectations would grow too large, rendering the tests less useful and their impact on business continuity a potential limiting factor (e.g. on releases, schedules, bonuses). If you're confident you want to cut a release, rapidly approaching your deadline, only to be kept from your goal by a failing test, chances are you're not going to be happy about the outcome. In a, unfortunately, fairly high number of these cases, one can even observe the tests being skipped entirely, reverting the release process to manual testing since it is perceived as having less of an impact (and it's easier to tick a box for "I've run this test" than to ignore a red pipeline).

So the goal should be to have these tests executed with a minimal burden on the bottom line, if not "cost neutral". But how to achieve that when you have to talk to the actual cloud API to run them?

## It works on my machine

Generally, working with the method of "run it on my machine, then run the same setup anywhere" is a fairly great concept, just because the initial feedback loop between test and potential failure is as short as possible, while giving you ample opportunity to adjust. This is also where the infamous trope "it works on my machine" is coming from.

Running tests in an isolated environment, that's not necessarily representative of the environment you're going to run your code in eventually is only borderline helpful though. So in this case, if I were to able to run the tests on my own laptop, with my own, user-tied credentials, I would be able to have them pass, but my laptop is neither representative of the actual space the code will run in eventually, nor can I give my user-tied credentials for the cloud to any CI environment[1].

How do I solve the problem of environment representation securely though?

### Reproducible test results through standardized environment setups

These days, achieving the first task it slightly more manageable than ever before, through the use of standardized environment configurations. Application runtimes and build artifacts inside containers have become a common sight in most enterprises, and as such, it's also fairly common to run your tests against containerized representations of your application, in order to ensure that whatever you run them against is "living" under the same pretenses as the eventual container that's running in a production context. The only thing you have to ensure afterwards is that the way your orchestrate and run whatever is inside the environment you run your tests in doesn't change along its path into a production context, but much rather change the "outside" parameters and inputs in a way that allows them to run the workloads in any of the contexts you wish to run them in.

So, in order to run our tests, all we have to do is to use an defined environment, with a version of Terraform the module is meant to run against, and version of Golang that's capable of running our Terratest tests. In GitHub Actions, you can define such an environment using `yaml`:

```yaml
name: Test
on:
  pull_request:
  push:
    branches:
      - main

jobs:
  terratest:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
      - uses: hashicorp/setup-terraform@v1
      - uses: actions/setup-go@v2
      - run: terraform init
      - name: terratest
        working-directory: tests
        run: go test
```

Lets break this apart:

1. We define a workflow in this `yaml` format by giving it a name ("Test") and allow for it be executed whenever somebody either a) submits a pull request or b) pushes a commit to the `main` branch.
2. We then define a single job in the `jobs` array, called `terratest`
3. Inside this job, we _define the environment we want this job to run in_ before, eventually, executing our tests, specifically, we tell GitHub Actions to run our instructions on a `ubuntu-20.04` machine
4. We then continue with the setup of the test environment: Check out the revision of the code this workflow runs against, and execute two pre-defined Actions, setting up Terraform and setting up Golang. Since there are strict requirements by this module for specific versions of either Terraform or Golang we can leave them out until we need to add them at some point (say, if Terraform releases an incompatible version, or Golang changes its API)
5. We then make sure that Terraform itself is initialized properly before executing the tests associated with the module. Otherwise, since all the tests are executed asynchronously, we might run into issues while 5-6 different Golang routines are trying to initialize the same codebase.
6. At least, we start running the tests!

This is well defined, and you could easily replicate the same setup on your laptop, either using containers, or actual virtual machines, and you would be getting the same results as GitHub Actions, likely.

However, there is one last issue: The tests are going to fail, because they are lacking the credentials to actually talk to AWS /o\

## You don't have to talk to the cloud at all

One of the most interesting aspects of discussions among engineers is usually that there's a premise and folks are starting to immediately think of solutions to challenges that arise from said premise. I've rarely ever had discussions in which people simply turned around and challenged the premise as a whole, especially if a probably solution is potentially "right around the corner". Instead, you're losing yourself in layers and layers of compromises and trade-offs, with some even bordering on bikeshedding, most of the time.

So lets challenge the premise this time: my main goal here is not talking to the cloud, any cloud for that matter, but rather to test my infrastructure code. My initial assumption was (and is) that I will have to execute these tests against the actual cloud APIs

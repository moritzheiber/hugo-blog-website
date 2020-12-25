+++
title = "Git client configurations for multiple identities"
date = "2019-03-29T13:11:52+01:00"
categories = ["git"]
tags = ["git", "configuration"]
aliases = ["/post/git-client-configurations-for-multiple-identities"]

+++

## The bane of maintenance

If you're like me, you're using your computer not just for work but also for your own personal projects and learning efforts. After all, why would you buy a completely separate machine just for the sake of maintaining two separate working environments, keeping them up-to-date and running while also constantly forgetting about where exactly you put that one file

.. was it your personal laptop?\
Or was it on your machine dedicated for work .. ?

The way I'm [provisioning my laptop(s)](https://github.com/moritzheiber/laptop-provisioning) definitely makes it easier to at least keep a consistent state across working environments, but the issue still remains that there's bound to be a time where you'll just be fed up with the overhead of having to maintain two separate environments for anything you do. The similarities are just too numerous.

## Using a single machine for work and play

I seldom meet other colleagues or business acquaintances who aren't using [Git](https://git-scm.org) for version control. Some are also using it for purposes beyond just maintaining a consistent code state across boundaries, for things like [agile software collaboration](https://nvie.com/posts/a-successful-git-branching-model/), or [project management](https://bitband.atlassian.net/wiki/spaces/BD/pages/248840196/Git+Jira+Integration#GitJiraIntegration-Gitcommands), personally I prefer to stick to its original intended goal, which is to make it easy for developers to share and merge code state effortlessly in distributed environments.

While Git certainly isn't a tool which is mastered overnight, it lends itself to day-to-day activities fairly easily, especially if you're already familiar with other alternatives such as [TFS](https://en.wikipedia.org/wiki/Team_Foundation_Server), [CVS](https://en.wikipedia.org/wiki/Concurrent_Versions_System) or [Subversion](https://en.wikipedia.org/wiki/Apache_Subversion). It can also be configured with [a plethora of options](https://git-scm.com/docs/git-config#_variables), most of which I will not be mentioning or introducing in the article. However, a few I find incredibly useful, especially when having to use separate "[personas](https://en.wikipedia.org/wiki/Persona)" for personal projects and for work.

### The Git configuration conundrum

Git is globally configured via a single configuration file inside your home directory, called `.gitconfig`. Any option you either specify by running `git config --global <option> <value>`, or by editing the file directly, will get picked up when you're using Git anywhere on your computer. You may override this behavior on a repository-by-repository basis by just skipping the `--global` parameter and executing the relevant configuration command inside the relevant repository.

Mind you though, these changes only carry a local effect and are not attached to the repository. Should you ever wipe the relevant repository's source code from your computer and clone a fresh copy the instructions you carefully set before will be gone.

And therein lies the dilemma.

### Identity crisis

Because locally specifying which configuration, identity and other options you want to use might sound appealing, but definitely not if you'd have to do it over and over again, for each and every repository diverging in purpose from your main, "global" configuration. Chances are, you'll most likely forget over time which repository you added different settings to before, and which are still using your global configuration settings. And then you'll end up with commit messages specifying the wrong email address, username, GitHub handle or even GPG signature.

This can have serious implications for your workflow. Signed commits aren't exactly a new feature of Git (they're actually rather old) and Git repository hosting companies like [GitHub](https://github.com) and [GitLab](https://gitlab.com) have introduced features like [requiring signed commits with verified signatures for protected branches](https://help.github.com/en/articles/about-required-commit-signing). It does increase your level of assurance since you can be certain a specific commit has actually been submitted by the person the contributor claims to be (if you trust the signing process). Therefore it's important to use the right combination of username, email address and GPG signature within your Git repositories.

### There's more than just identity

But there's more! What if you only wanted to sign your commits for a certain work package, inside a particular directory, but with a large number of repositories? What about other options you have set globally but wanted to have changed to either comply with localized requirements or to make your collaboration easier on certain projects?

You've seen the list of options you can configure with `git config`, I'm sure there'll be a few you would rather have set differently depending on your context.

This is where **conditional Git configuration includes** come into play.

## Context matters

Git allows for you to include specific configuration files depending on directory structures, i.e. if you have certain contexts under which you are operating (e.g. "Project X", "Project Y", "client A" or "client B"), and you're keeping separate working directories for all of their source code tracked in Git, you may include specific configuration files which apply to all of the top-level repositories and subsequent levels under said directory structure!

As an example, here's my current, global `.gitconfig` file:

```ini
[user]
  email = hello@heiber.im
  name = Moritz Heiber
  signingkey = 2F3A1C05
[push]
  default = simple
[github]
  user = moritzheiber
[credential]
  helper = /usr/share/doc/git/contrib/credential/libsecret/git-credential-libsecret
[includeIf "gitdir:~/Code/thoughtworks/"]
  path = ~/Code/thoughtworks/gitconfig
[log]
  showSignature = true
[commit]
  gpgsign = true
[branch]
  sort = committerdate
```

Let's pick it apart!

### User context

The first part of the file references my "global" identity I'd like to use for anything Git that's no specific to any of the work I'm doing professionally:

```ini
[user]
  email = hello@heiber.im
  name = Moritz Heiber
  signingkey = 2F3A1C05
```

As you can see, it's clearly showing that I am who you might think I am ("Moritz Heiber"), and that the email address I'd prefer using for my commits is `hello@heiber.im`. It also specifies the short handle for a GPG key I will want to use for signing my commit messages, should I decide to do it.

Now, Git is fairly good at guessing which key to use, based upon my email address or even user handle, but specifying it explicitly does away with any ambiguity and reassures me that I'll always be using the "right" key associated with this particular identity.

### Push settings

This snippet defines the type of push `git push` is using by default:

```ini
[push]
  default = simple
```

It became the default since Git 2.0, but I've been keeping it in my configuration to, again, reduce ambiguity. There are [a lot of other options you can potentially choose from](https://git-scm.com/docs/git-config#Documentation/git-config.txt-pushdefault), and you should make up your own mind which way you want to go.

### GitHub handle

```ini
[github]
  user = moritzheiber
```

This is a non-standard way of specifying your GitHub username, and it's used by some tooling I'm currently using for some projects.

### Credential helper

```ini
[credential]
  helper = /usr/share/doc/git/contrib/credential/libsecret/git-credential-libsecret
```

Not everyone seems to know about credentials helpers and their usefulness. Personally, I've been defaulting to HTTP-based endpoints for repositories for quite a while now, because it's faster and doesn't rely on the exchange of a key pair to work correctly. There's [convenient documentation available for all major operating systems on how to set up a credential helper for your local working environment](https://help.github.com/en/articles/caching-your-github-password-in-git), and at least for me it has proven to be invaluable, especially for repositories that do not allow for SSH access (because they're hidden behind a firewall or a proxy server).

No more entering your username and password over and over again!

### Showing signatures in logs

By default, `git log` does not show whether or not a certain commit has been signed by a verified entity. In fact, it'll be completely lost on you whether or not it is signed at all!

The following snippet changes that:

```ini
[log]
  showSignature = true
```

The output before the change:

```
commit f3a9fbb796508d30f22b63f838d9ebc47e848dfe (HEAD -> master, origin/master)
Author: Moritz Heiber <hello@heiber.im>
Date:   Fri Mar 29 10:44:49 2019 +0100

    Added repo for LDAC/ACC/AptX Bluetooth support (probably obsolete after 19.04)
```

and after it:

```
commit f3a9fbb796508d30f22b63f838d9ebc47e848dfe (HEAD -> master, origin/master)
gpg: Signature made Fr 29 Mär 2019 10:44:49 CET
gpg:                using RSA key 638614C7C9374D714EA6DA41650DA6EE5526CD96
gpg: Good signature from "Moritz Heiber <hello@heiber.im>" [ultimate]
Primary key fingerprint: 7430 7F3F 5312 0B4C BBAC  4C28 1F77 2A09 2F3A 1C05
     Subkey fingerprint: 6386 14C7 C937 4D71 4EA6  DA41 650D A6EE 5526 CD96
Author: Moritz Heiber <hello@heiber.im>
Date:   Fri Mar 29 10:44:49 2019 +0100

    Added repo for LDAC/ACC/AptX Bluetooth support (probably obsolete after 19.04)
```

Before you never would've guessed that the commit is actually signed with a valid key. How about that!

### Sorting branches in the overview

How many branches do you have? Personally, I love working with [trunk-based development](https://trunkbaseddevelopment.com/) (especially since I'm also a huge advocate of [Continuous Delivery](https://en.wikipedia.org/wiki/Continuous_delivery), however, not every project is seeing eye-to-eye with this concept. You'll find a incredibly high amount of branches on almost every larger project, and at a glance you really wouldn't know which branch has actually seen activity recently .. or would you?

```ini
[branch]
  sort = committerdate
```

This convenient snippet introduces `git branch` to the concept of a more useful sorting, giving you an idea which branch has seen recent commits by displaying them at the top. It's a personal preference, but I find it invaluable to keep up-to-date with whatever branches are still in use and which branches are stale, over time.

### Last but not least .. Conditional includes

I've saved the best one for last. [Conditional includes](https://git-scm.com/docs/git-config#_conditional_includes).

```ini
[includeIf "gitdir:~/Code/thoughtworks/"]
  path = ~/Code/thoughtworks/gitconfig
```

This statement basically says: "**If** there is a directory named **`~/Code/thoughtworks`** and it contains a file called **`gitconfig`**, use it to **override the global settings for any and all git repositories under the directory structure in `~/Code/thoughtworks/`**".

It's important to mention that this only ever overrides my **global** settings, **not the per-repository** settings I may or may not have entered separately. The order of escalation is:

```
Repositority Git configuration > Directory Git configuration > $HOME/.gitconfig
```

### Changing my identity, conditionally!

What's in the file though? This!

```ini
[user]
  email = mheiber@thoughtworks.com
  name = Moritz Heiber
  signingkey = F75C32B1
[commit]
  gpgsign = true
```

It simply tells Git to rather use my "corporate" identity (my email "`mheiber@thoughtworks.com`" and the matching GPG key) and to absolutely enforce commit signing, no matter what the global configuration tells it to do. I might want to decide to skip signing my commits for my personal projects, but it'll say mandatory for any and all projects contained in the `~/Code/thoughtworks/` directory (unless I specify a different option on a repository level).

You could add any other Git configuration option to this file, specifically addressing concerns in and around your working environment, project regulations or compliance issues.

## Newly found flexibility

I never thought to be able to solve the issues I've been having with Git and its configuration so elegantly, but conditional includes really are helping me to maintain a level of flexibility for all of my work, whether it's personal or work-related, and it's a joy to work with. I have already added the `gitconfig` file to my [laptop-provisioning repository](https://github.com/moritzheiber/laptop-provisioning/commit/62ff85444abe090db3e600a09e0c98531925a319) and I'm also keeping the global configuration of my dotfiles stashed away separately.

This will ensure I'll always be able to replicate this setup and to keep working with several different identities securely and efficiently when using Git!

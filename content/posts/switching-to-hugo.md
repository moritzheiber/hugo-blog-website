+++
categories = ["blog", "golang"]
date = "2015-05-10T20:12:10+02:00"
tags = ["hugo", "golang"]
title = "Switching to Hugo"
aliases = ["/post/switching-to-hugo"]

+++

[Hugo](http://gohugo.io) is a static site generator written in [Go](http://golang.org). It's incredibly fast and flexible, has a unique template engine, and writing themes is really easy, should the slew of available themes not be to your liking. I decided to use it because it lets me get rid of all the dependencies [Jekyll](http://jekyllrb.com/) came with, it had a comprehensive preview mechanism (`hugo server`) and it allowed for a very narrow deployment pipeline to [GitHub Pages](http://pages.github.com), where this site is hosted at. Coincidentally, this is also why it doesn't offer TLS supports.

So without further ado, here are some instructions on how to get a Hugo site of your own, hosted on GitHub, for free!

## Installing Hugo

Since Hugo is written in Go downloading and installing it can be as easy as pulling the individual binary and shoving it into the right place in your `$PATH`. Just go to their [release website](https://github.com/spf13/hugo/releases) and take your pick. I used the `_amd64.deb` since I'm running Ubuntu:

```bash
$ wget https://github.com/spf13/hugo/releases/download/v0.13/hugo_0.13_amd64.deb
$ dpkg -i hugo_0.13_amd64.deb
```
This will install `hugo` to `/usr/bin`:

```bash
$ dpkg -L hugo
/usr/bin/hugo
```

Obviously, you can also pull down the sources (via git or by downloading the zip file) and compile your own binary. [There are detailed instructions on Hugo's website](http://gohugo.io/overview/installing#installing-from-source).

## Creating your new Hugo site

To set up the Hugo scaffolding for your new site all you need to run is:

```bash
$ hugo new site <path>
```

Now your directory in `<path>` should look something like this:

```bash
$ ls
archetypes  config.toml  content  data  layouts  static
```

Hugo uses `toml` as its configuration syntax by default. However, you can also use `yaml` or `json` if you prefer either of those. The few directories carry a certain significance since taxonomy in Hugo is based on its directory structure:

- [archetypes](http://gohugo.io/content/archetypes/): This contains the default definitions for new content. You can add things like `tags` or `categories` here.
- [content](http://gohugo.io/content/organization/): Here is where all your content goes. This means regular pages or, nested in directories, posts, categories etc.
- [data](http://gohugo.io/extras/datafiles/): You can store a poor man's database in here and reference it right in your templates and pages.
- [layouts](http://gohugo.io/templates/overview/): Relevant site layouts can be found in here. Usually, those would be provided by themes you're using.
- `static`: Any content you don't need to have touched by Hugo but want to use on your website goes in here.

Now, the `config.toml` is where you main site configuration is stored. For my site it looks like this:

```toml
# Your website URL
baseurl = "http://heiber.im"  
# The language you're using on it
languageCode = "en-us"  
# The main title of your website
title = "musings about silly things"  
# This should be here if you plan on using GitHub Pages
canonifyurls = true
# This is the editor Hugo calls upon when you run "hugo new post/something.md"
editor = "vim" 
# [...]
```
You can take a look at the current file [on GitHub](https://github.com/moritzheiber/huge-blog-website/blob/master/config.toml). There are also a lot of theme dependent variables in there, which you can use pretty much anywhere in your layouts, theme definitions and content. [There's a whole section on template variables in the docs](http://gohugo.io/templates/variables/).

## Grab a theme you like

There is a [separate repository containing a lot of diverse and different themes](https://github.com/spf13/hugoThemes). As of this moment there isn't a comprehensive overview of the available themes accessible yet, however, there's [a ticket in GitHub which you can follow](https://github.com/spf13/hugoThemes/issues/35) about finally getting a site up and running for exactly that purpose.

There are two ways of still giving them a try/taking a peek:

1. Each theme should contain a screenshot of it's whole glory in the `images/` directory. If it doesn't consider filing an issue.
2. You can use the instructions mentioned in the `README.md` to clone all of the available themes in the repository and then run `hugo server -t <theme-name>` in order to try out each theme with your newly created blog

## Start creating content!

Now just run `hugo new post/new-first-post.md` and have at it! If you want to see your changes appear incrementally you can run `hugo -w` (`-w` for "watch for changes") in another window and navigate to [http://localhost:1313](http://localhost:1313).

Especially with complicated markdown formatting this is a huge timesaver.

## Push it to the web

After you're done just run `hugo` once in the blog's directory. Mind your theme choice; you can use `-t <theme-name>` if you haven't added it to the configuration. 

Your statically created site is now ready for consumption in `public/`. You can upload it anywhere you so desire. I personally host my website on GitHub Pages. There's [a chapter on it in the Hugo docs](http://gohugo.io/tutorials/github-pages-blog/). Just make sure to use the right method of branching/pushing, i.e. personal websites do not use a `gh-pages` branch but rather `master` directly.

I've created a `deploy.sh` script which alleviates some of pain in the repetitive tasks you need to repeat in order to get your site onto GitHub Pages:

```bash
#!/bin/bash

GIT_URL="git@github.com:moritzheiber/moritzheiber.github.io.git"
GIT_BRANCH="master"

echo -e "\033[0;32mDeploying updates to GitHub...\033[0m"

push_git () {
  msg="Rebuilding site `date`"
  if [ $# -eq 1 ] ; then 
    msg="$1"
  fi
  
  # Commit changes.
  git commit -m "$msg"

  # Push source and build repos.
  git push origin master
  git subtree push --prefix public ${GIT_URL} ${GIT_BRANCH}
}

# Build the project. 
hugo -t hyde-x

# Add changes to git.
git add --all
git diff --staged --stat

while true; do
    read -p "Do you wish to push these changes? " yn
    case $yn in
        [Yy]* ) push_git; break;;
        [Nn]* ) exit;;
        * ) echo "Please answer yes or no.";;
    esac
done
```

The script:

1. Builds the current state of the site using `hugo`.
2. Adds all the changes to the git index and stages them.
3. Gives you a brief overview of what they actually are.
4. Asks you to accept these changes.
5. Commits the changes, either using a provided messages (i.e. `./deploy.sh "your message here"`) or a timestamped default.
6. Pushes the changes to the main repository, also holding the `hugo` codebase.
7. Pushes just the content in `public/` to a separate repository, using the `master` branch, so that it gets picked up by GitHub Pages.

That's it. Your changes are now online.

## Troubleshooting

If you see a go backtrace at any point in time and you don't happen to have any content in your blog yet it could be you've chosen a theme that relies on content to build certain parts of the site i.e. certain variables and/or content snippets. You should be fine once you've created more than or equal to one post/page.

## Further reading

Hugo is very well documented, literally anything you desire can either by found [in the docs](http://gohugo.io/overview/introduction/) or on [their buzzing forums](http://discuss.gohugo.io/).

## Conclusion

I couldn't be happier with the setup at the moment. Right now, creating new posts is just about running `hugo new post/<post>.md`, adding my content and then `./deploy.sh`. After that it's online. Two simple steps.

It's much easier and more seamless as it was before. I'm sorry to be moving away from such an accomplished project as Jekyll, but maintaining Jekyll and it's dependencies just isn't what I'm after and [Octopress 3.0](http://www.octopress.org) isn't ready yet.

For the time being, here's to a bright future using [Hugo](http://gohugo.io)!

+++
categories = ["docker", "devops", "java"]
date = "2016-02-07T14:09:05+01:00"
description = "Lets take a look at how to to create a very small, versatile Docker base image which you can use for pretty much any purpose in today's changing container landscape"
keywords = ["alpine linux", "docker", "security"]
title = "Creating a good, secure Docker base image"

+++

**tl;dr**: Build small, efficient images, use [Alpine Linux](https://www.alpinelinux.org/) as your foundation, build from there, add glibc if necessary, remove static/generated files and documentation, never run more than one process per container and use verified, trustworthy sources.

# The premise

When I first started using Docker, everyone kept raving about how easy and intuitive it was to use, how incredibly well it handled itself and how much time everyone was saving because of it. Once I gave it a try I discovered that not only was almost any image bloated beyond recognition, used very insecure practices (no package signing, `curl | sh` installing, blind trust in upstream hub images etc.) but also none of them adhered to the concept Docker was intended for originally: *Isolated, single-process, easily distributable, lean images*.

Docker images, explicitely, are not designed to replace complex virtual machine setups, fully integrated with logging, monitoring, alerting and several resources running side-by-side. Instead, Docker encourages composition by following the paradigm of the kernel environment abstraction though [cgroups](https://en.wikipedia.org/wiki/Cgroups) and [namespaces](https://en.wikipedia.org/wiki/Cgroups#Namespace_isolation). It's as if you were trying to say

> Give me the very same bare-bones environment the `init` process gets on my machine once the kernel has finished initializing.

> I'll take over from there

That is also why the process you're specifying in your `Dockerfile`'s `CMD` instruction is started with `PID 1`. It's a close resemblence of what defines Unix as a whole.

Look at your process list right now, using `top` or `ps` for example, and you will find the process `init` claiming that very same `PID`. It's the core of very Unix operating system, the mother of all processes. Once you have internalized this concept, that every single process on Unix is a child of `init`, you will understand the environment a Docker container is supposed to live in: No-frills, bare kernel exposure. The miminum for any process to live on.

That's our starting point

# How to get started

Now, today's applications are complex software systems. Most of them require a lot of libraries to work correctly; there are schedulers, actors, compilers and a hundreds other helpers involved in designing and implementing modern software applications. Their architecture, usually, is hermetically shielded from us, through layers and layers of abstractions and interfaces, and while that can also be said for containers in a sense, at least from a system architecture perspective we need to think a little simpler than we did before with whole virtual environments.

## Taking Java as an example

To start things off, and to build the most basic container you can run your application on, think of your application by itself. What does it actually need to run?

Chances are, if you're running a Java application, it'll probably need a Java runtime environment. If you're running a Rails application it will need a runtime Ruby interpreter. The same logic applies to Python applications. Golang and other compiled languages are a little different in this regard, but I will come that later.

Advancing with the Java example, the next step to think about is: What does a JRE require to run? Since it's the single most vital component to getting your application to run, the next logical step would be to figure out what the JRE needs to run your code.

As it turns out, not that much really. The Java Virtual Machine (JVM) is meant to function as an operating system abstraction layer; to run code independently of its host environment. Thus you are pretty much set once you have a JRE (Java Runtime Environment) ready to go.

_(In reality, operating system independence cannot be taken for granted. There are a ton of operating system specific APIs and proprietary system extensions. But for the sake of this example, let's focus on the easier alternative.)_

I will be presenting this example case based on the assumption we are talking about a Linux x86_64, since, although Java theoretically runs independent of OS architectures, Docker, the container engine described in this example, does not. [It runs on Linux, and Linux only](https://docs.docker.com/machine/overview/#why-should-i-use-it).

On Linux, the JVM mainly hooks into an existing C library for making most of its calls to the underlying OS, and therefore your machine. Oracle's official JRE, which unfortunately most people use to this day, does this through interfacing with **libc**, otherwise also know as **glibc**. This means, for being able to actually run a Java program of any sort you need to be able to run its virtual machine, which in turn requires glibc. Apart from that you probably want some sort of shell to manage your environment, and a way of interfacing with the outside world, i.e. networking and resource abstraction.

To sum up our requirements to run an example Java application:

- A JRE, in your example, we'll take the Oracle JRE
- glibc, for running the JRE
- A shell, for the sake of familiarity we will take the popular bash shell
- A barebones environment to give us awareness for networking, memory management, filesystem support and other important abstractions like (ttys, input/output handling etc.)

# Enter Alpine Linux

[Alpine Linux](https://www.alpinelinux.org/) has been gaining a lot of support as a distribution recently, mainly because it [packs quite a punch in terms of selection of pre-build, signed and trusted packages](https://pkgs.alpinelinux.org/packages) while only clocking a very impressive 2MB (!) in size when attached to a Docker container. To put this into perspective, while writing this post, the lastest base images (already stripped down to their very code functionality) for other distributions maintain the following sizing:

- `ubuntu:latest`: 66MB (already fairly slim compared to, e.g. earlier images which sometimes carried around 600MB+ with them)
- `debian:latest`: 55MB (same as above, quite alright considering they started with 200MB+)
- `arch:latest`: 145MB (I took the [dock0/arch image](https://hub.docker.com/r/dock0/arch/) as there are no "official", Docker Inc maintained images for Arch Linux)
- `busybox:latest`: 676KB (Yes, that's **Kilobytes**; I'll come to that in a second)
- `alpine:latest`: 2MB (In words, TWO megabytes. And that's an actual Linux system with a package manager!)

I won't go into detail of what Alpine Linux actually is and why it exists. [They are doing a pretty good job on their own](https://www.alpinelinux.org/about/).

## Busybox as the smallest contender?

As you can see from the comparison, The only one beating Alpine Linux to the punch in terms of sizing is the Busybox image. 676KB are a testament as to why Busybox is used in pretty much all embedded systems requiring any kind of shell environment these days. It's used in routers, switches, credit card terminals (seriously) and, someday, [probably your toaster](http://www.bbc.com/future/story/20150216-be-afraid-of-the-smart-toaster). It's as barebones as barebones can be, while still providing a sufficient, well maintained shell system interface.

And that's also why you will want to choose Alpine Linux over Busybox if you want a little more wiggle room in terms of flexibility.

There are a lot of articles on the web explaining [why people chose Alpine Linux over "just" Busybox](http://odino.org/minimal-docker-run-your-nodejs-app-in-25mb-of-an-image/) (Alpine Linux is based upon Busybox). But to sum them up in an instant:

- **Versatile, open, actively maintained package repository**: Alpine Linux uses the `apk` package manager, which is baked right into the Docker image that's shipped through official channels. With `Busybox` you'd need to add a package manager, such as `opkg`, and even then you still need a reliable source for package information, which hardly exist. The [OpenWRT project](https://openwrt.org/) maintains some repositories, but they are working for a very specific angle, and Alpine Linux ships a larger subset of the unilaterally available packages you would usually find available from any of the other major distributions out there. To sum it up: If you still compile software like NodeJS or Ruby to use them in your container you can just run `apk add nodejs ruby` in the future and be done in a matter of seconds.
- **Size does matter. But not when you're arguing functionality, versatility and accessibility in the realms of 1 1/2 megabytes**: The added benefits Alpine Linux gives you over Busybox are immense, so much so, that adding just a fraction in size for your later image does not really matter in this case. To put this into perspective: Pulling in `bash`, as a package on Alpine Linux, adds a whooping 5MB to your Docker image. That's two Alpine containers, and then some!
- **Broad, open support**: Docker Inc. has hired [the creator of Alpine Linux](https://twitter.com/n_copa) to work on it in Docker's interest. All official images, maintained by Docker Inc., [will be moving (or have been moved already) to Alpine Linux](https://www.brianchristner.io/docker-is-moving-to-alpine-linux/). This means first class citizen vendor support for the very foundation of your images. There is nothing better and more convincing than this if you're starting to build your own containers.

# Building a Java enabled base image

As I just explained, Alpine Linux is a good choice for the foundation of your own image, specifically the `FROM` instruction in your `Dockerfile`. Hence, we will be using it to build our own lean and efficient Docker image. Let's get started!

## The seams: Alpine + bash

Every `Dockerfile` starts with an instruction which specifies its parent container. Usually, that's an image you are inhereting from; in our case it's the `alpine:latest` image:

    FROM alpine:latest
    MAINTAINER Moritz Heiber <hello@heiber.im>

We are also specifying who's responsible for the image. This information is vital to the Docker ecosystem and also for uploading the image to the [Docker Hub](https://hub.docker.com/) at some later point in time.

That's it, now you have a canvas to work with. Let's install our chosen shell, `bash`, and run our container for the first time. Add the following instructions to your `Dockerfile`:

    RUN apk add --no-cache --update-cache bash
    CMD ["/bin/bash"]

The resulting `Dockerfile` should look like this:

    FROM alpine:latest
    MAINTAINER Moritz Heiber <hello@heiber.im> # You want to add your own name here

    RUN apk add --no-cache --update-cache bash
    CMD ["/bin/bash"]

Great! Let's build this container:

```bash
$ docker build -t my-java-base-image .
Sending build context to Docker daemon 2.048 kB
Step 1 : FROM alpine:latest
 ---> 2314ad3eeb90
Step 2 : MAINTAINER Moritz Heiber <hello@heiber.im>
 ---> Running in 63433312d77e
 ---> bfe94713797a
Removing intermediate container 63433312d77e
Step 3 : RUN apk --no-cache --update-cache add bash
 ---> Running in 12ae43605260
fetch http://dl-4.alpinelinux.org/alpine/v3.3/main/x86_64/APKINDEX.tar.gz
fetch http://dl-4.alpinelinux.org/alpine/v3.3/main/x86_64/APKINDEX.tar.gz
fetch http://dl-4.alpinelinux.org/alpine/v3.3/community/x86_64/APKINDEX.tar.gz
fetch http://dl-4.alpinelinux.org/alpine/v3.3/community/x86_64/APKINDEX.tar.gz
(1/5) Installing ncurses-terminfo-base (6.0-r6)
(2/5) Installing ncurses-terminfo (6.0-r6)
(3/5) Installing ncurses-libs (6.0-r6)
(4/5) Installing readline (6.3.008-r4)
(5/5) Installing bash (4.3.42-r3)
Executing bash-4.3.42-r3.post-install
Executing busybox-1.24.1-r7.trigger
OK: 13 MiB in 16 packages
 ---> 2ea4fbc1c950
Removing intermediate container 12ae43605260
Step 4 : CMD /bin/bash
 ---> Running in d2291684b797
 ---> ecc443d68f27
Removing intermediate container d2291684b797
Successfully built ecc443d68f27
```

And run it:

```bash
$ docker run --rm -ti my-java-base-image
bash-4.3#
```

Success! We have an image running `bash` on top of Alpine Linux.

Now for the rest of the required parts.

## glibc and friends

As mentioned before, Oracle's JRE needs a working copy of glibc to run properly. Alpine Linux doesn't use glibc though, it uses a much smaller alternatives named [musl libc](http://www.musl-libc.org/). If you have ever worked with glibc you know how it has grown over the years (like any large software product) to include provisions for pretty much any software problem you could possibly come up with in C. Obviously, that versatility comes with trade-offs, one of them being size. A regular glibc, compiled to work on Alpine Linux, clocks in at roughly 5MB, and the resulting packages based on that package would carry an overhead as well. The alternative, musl-libc comes as a single-binary, 897KB image, and supports all the necessary parts to run modern binaries on Linux architectures. It's a trade-off you'll gladly make.

Unless you have to use glibc to run proprietary code.

With Oracle's JRE, there is no way around adding glibc to our small image. Luckily, [Andy Shinn](https://github.com/andyshinn) has done all of the work for us already, preparing pre-compiled, signed glibc images for Alpine Linux. They are in the [alpine-pkg-glibc](https://github.com/andyshinn/alpine-pkg-glibc) repository on GitHub, with the most recent release being [2.22-r5](https://github.com/andyshinn/alpine-pkg-glibc/releases/tag/2.22-r5).

Let's add these packages by changing our `Dockerfile` in the following way:

```docker
ENV GLIBC_PKG_VERSION 2.22-r5

RUN apk add --no-cache --update-cache curl ca-certificates bash && \
  curl -Lo glibc-${GLIBC_PKG_VERSION}.apk "https://github.com/andyshinn/alpine-pkg-glibc/releases/download/${GLIBC_PKG_VERSION}/glibc-${GLIBC_PKG_VERSION}.apk" && \
  curl -Lo glibc-bin-${GLIBC_PKG_VERSION}.apk "https://github.com/andyshinn/alpine-pkg-glibc/releases/download/${GLIBC_PKG_VERSION}/glibc-bin-${GLIBC_PKG_VERSION}.apk" && \
  apk add --allow-untrusted glibc-${GLIBC_PKG_VERSION}.apk && \
  apk add --allow-untrusted glibc-bin-${GLIBC_PKG_VERSION}.apk && \
  /usr/glibc-compat/sbin/ldconfig /lib /usr/glibc-compat/lib
```

Our `Dockerfile` now looks like this:

```Docker
FROM alpine:latest
MAINTAINER Moritz Heiber <hello@heiber.im>

ENV GLIBC_PKG_VERSION 2.22-r5

RUN apk add --no-cache --update-cache curl ca-certificates bash && \
  curl -Lo glibc-${GLIBC_PKG_VERSION}.apk "https://github.com/andyshinn/alpine-pkg-glibc/releases/download/${GLIBC_PKG_VERSION}/glibc-${GLIBC_PKG_VERSION}.apk" && \
  curl -Lo glibc-bin-${GLIBC_PKG_VERSION}.apk "https://github.com/andyshinn/alpine-pkg-glibc/releases/download/${GLIBC_PKG_VERSION}/glibc-bin-${GLIBC_PKG_VERSION}.apk" && \
  apk add --allow-untrusted glibc-${GLIBC_PKG_VERSION}.apk && \
  apk add --allow-untrusted glibc-bin-${GLIBC_PKG_VERSION}.apk && \
  /usr/glibc-compat/sbin/ldconfig /lib /usr/glibc-compat/lib

CMD ["/bin/bash"]
```

Let's walk through these instructions one by one:

```Docker
ENV GLIBC_PKG_VERSION 2.22-r5
```

We want to stay on the current version of glibc released on GitHub. While you could just exchange URLs each and every time a new release comes up, putting the current version into a variable within the `Dockerfile` makes switching versions as easy as editing a single line.

```Docker
RUN apk add --update-cache curl ca-certificates bash && \
```

This `RUN` instruction will use the `apk` command to install the packages we need in order to fetch resources from somewhere else. Obviously, I chose `curl`, and I also installed the `ca-certificates` package to make sure we aren't going to run into trouble when accessing TLS enabled websites. Lastly, `bash` at the end we already had in our last iteration of our `Dockerfile`.

```bash
 curl -Lo glibc-${GLIBC_PKG_VERSION}.apk "https://github.com/andyshinn/alpine-pkg-glibc/releases/download/${GLIBC_PKG_VERSION}/glibc-${GLIBC_PKG_VERSION}.apk" && \
  curl -Lo glibc-bin-${GLIBC_PKG_VERSION}.apk "https://github.com/andyshinn/alpine-pkg-glibc/releases/download/${GLIBC_PKG_VERSION}/glibc-bin-${GLIBC_PKG_VERSION}.apk" && \
```

These commands are appended to the `RUN` instruction we just discussed. They are downloading the release packages for glibc directly from GitHub.

```bash
/usr/glibc-compat/sbin/ldconfig /lib /usr/glibc-compat/lib
```

The last step is to ensure the linker provided by glibc is up-to-date when it come to all the available libraries it's supposed to be serving for dynamically linked binaries. It's essential since otherwise the JRE binaries wouldn't be able to find the libc libraries to hook into at runtime.

All right! We now have a full-fledged environment ready to run (almost) all package requiring glibc!

## The Java Runtime Environment

Traditionally, Oracle doesn't take all that kindly to people downloading their software some a repository. However, people have found a way of doing so regardless. You can install their JRE by adding the following command(s) to your `Dockerfile`:

```Docker
ENV JAVA_VERSION_MAJOR 8
ENV JAVA_VERSION_MINOR 72
ENV JAVA_VERSION_BUILD 15
ENV JAVA_PACKAGE server-jre

RUN curl -jksSLH "Cookie: oraclelicense=accept-securebackup-cookie" \
  "http://download.oracle.com/otn-pub/java/jdk/${JAVA_VERSION_MAJOR}u${JAVA_VERSION_MINOR}-b${JAVA_VERSION_BUILD}/${JAVA_PACKAGE}-${JAVA_VERSION_MAJOR}u${JAVA_VERSION_MINOR}-linux-x64.tar.gz" | gunzip -c - | tar -xf - && \
  apk del curl ca-certificates && \
  mv jdk1.${JAVA_VERSION_MAJOR}.0_${JAVA_VERSION_MINOR}/jre /jre && \
  rm /jre/bin/jjs && \
  rm /jre/bin/keytool && \
  rm /jre/bin/orbd && \
  rm /jre/bin/pack200 && \
  rm /jre/bin/policytool && \
  rm /jre/bin/rmid && \
  rm /jre/bin/rmiregistry && \
  rm /jre/bin/servertool && \
  rm /jre/bin/tnameserv && \
  rm /jre/bin/unpack200 && \
  rm /jre/lib/ext/nashorn.jar && \
  rm /jre/lib/jfr.jar && \
  rm -rf /jre/lib/jfr && \
  rm -rf /jre/lib/oblique-fonts && \
  rm -rf /tmp/* /var/cache/apk/* && \
  echo 'hosts: files mdns4_minimal [NOTFOUND=return] dns mdns4' >> /etc/nsswitch.conf

ENV JAVA_HOME /jre
ENV PATH ${PATH}:${JAVA_HOME}/bin
```

Woah! There is so much going on with these commands, what's happening?!

All right, all right, let's step through them one by one before we take a look at the `Dockerfile` as a whole:

```Docker
ENV JAVA_VERSION_MAJOR 8
ENV JAVA_VERSION_MINOR 72
ENV JAVA_VERSION_BUILD 15
ENV JAVA_PACKAGE server-jre

WORKDIR /tmp
```

This one's pretty simple. It defines the software version we want to obtain from Oracle's servers. While writing this document, the versions mentioned above where the most recent. That might have changed since. You can get to the latest available version by [looking at Oracle's website](http://www.java.com/en/download/linux_manual.jsp).
It also specifies the instruction `WORKDIR` which basically just changes the working directory to `/tmp` in this instance. We need a temporary directory to work from, `/tmp` seems an appropriate location as any.

```Docker
RUN curl -jksSLH "Cookie: oraclelicense=accept-securebackup-cookie" \
  "http://download.oracle.com/otn-pub/java/jdk/${JAVA_VERSION_MAJOR}u${JAVA_VERSION_MINOR}-b${JAVA_VERSION_BUILD}/${JAVA_PACKAGE}-${JAVA_VERSION_MAJOR}u${JAVA_VERSION_MINOR}-linux-x64.tar.gz" | gunzip -c - | tar -xf - && \
```

This one's a tad bit more complicated. It uses `curl` to pass a special header (`"Cookie: oraclelicense=accept-securebackup-cookie"`) to Oracle's download servers before obtaining the actual software package. It's required since otherwise you'd be getting a negative response.
It then proceeds to construct a valid URL from the variables we passed into the `Dockerfile` in our first step, while piping the result directly into `gunzip` and `tar`.
In other words: It doesn't actually save the downloaded tarball anywhere, but rather extracts the contained software package directly onto our filesystem. Quite nifty!

```bash
apk del curl ca-certificates && \
```

At this point in time both of these packages have done their deed and are no longer required. They were "build dependencies", needed to set up the container, but not to actually run the application, eventually. It's good practice to remove them right now, to conserve space.

```bash
  rm /jre/bin/jjs && \
  rm /jre/bin/keytool && \
  rm /jre/bin/orbd && \
  rm /jre/bin/pack200 && \
  rm /jre/bin/policytool && \
  rm /jre/bin/rmid && \
  rm /jre/bin/rmiregistry && \
  rm /jre/bin/servertool && \
  rm /jre/bin/tnameserv && \
  rm /jre/bin/unpack200 && \
  rm /jre/lib/ext/nashorn.jar && \
  rm /jre/lib/jfr.jar && \
  rm -rf /jre/lib/jfr && \
  rm -rf /jre/lib/oblique-fonts && \
  rm -rf /tmp/* /var/cache/apk/* && \
```

The JRE comes with a lot of tools you probably will never need to run your actual application. Again, to save on space we are going to remove most of them.

_Note: This might differ for your environment. Maybe you need a couple of them. It depends on what you need to run your application._

The last line, finally, removes all of our temporary files and also the packaging caches built by `apk`. Within the immutable container we do not need either of them anymore.

```bash
echo 'hosts: files mdns4_minimal [NOTFOUND=return] dns mdns4' >> /etc/nsswitch.conf
```

Last but not least, we modify the `nsswitch.conf` in order to make sure we are able to properly resolve networking entities. This isn't used by Alpine Linux directly, but by glibc (and therefore, Java), and thus we still need to apply this rather crude hack of re-arranging the order hosts are resolved from.

In the end, your `Dockerfile` should now look a little something like this:

```Docker
FROM alpine:latest
MAINTAINER Moritz Heiber <hello@heiber.im>

ENV JAVA_VERSION_MAJOR 8
ENV JAVA_VERSION_MINOR 72
ENV JAVA_VERSION_BUILD 15
ENV JAVA_PACKAGE server-jre
ENV GLIBC_PKG_VERSION 2.22-r5

WORKDIR /tmp

RUN apk add --no-cache --update-cache curl ca-certificates bash && \
  curl -Lo glibc-${GLIBC_PKG_VERSION}.apk "https://github.com/andyshinn/alpine-pkg-glibc/releases/download/${GLIBC_PKG_VERSION}/glibc-${GLIBC_PKG_VERSION}.apk" && \
  curl -Lo glibc-bin-${GLIBC_PKG_VERSION}.apk "https://github.com/andyshinn/alpine-pkg-glibc/releases/download/${GLIBC_PKG_VERSION}/glibc-bin-${GLIBC_PKG_VERSION}.apk" && \
  apk add --allow-untrusted glibc-${GLIBC_PKG_VERSION}.apk && \
  apk add --allow-untrusted glibc-bin-${GLIBC_PKG_VERSION}.apk && \
  /usr/glibc-compat/sbin/ldconfig /lib /usr/glibc-compat/lib && \
  curl -jksSLH "Cookie: oraclelicense=accept-securebackup-cookie" \
  "http://download.oracle.com/otn-pub/java/jdk/${JAVA_VERSION_MAJOR}u${JAVA_VERSION_MINOR}-b${JAVA_VERSION_BUILD}/${JAVA_PACKAGE}-${JAVA_VERSION_MAJOR}u${JAVA_VERSION_MINOR}-linux-x64.tar.gz" | gunzip -c - | tar -xf - && \
  apk del curl ca-certificates && \
  mv jdk1.${JAVA_VERSION_MAJOR}.0_${JAVA_VERSION_MINOR}/jre /jre && \
  rm /jre/bin/jjs && \
  rm /jre/bin/keytool && \
  rm /jre/bin/orbd && \
  rm /jre/bin/pack200 && \
  rm /jre/bin/policytool && \
  rm /jre/bin/rmid && \
  rm /jre/bin/rmiregistry && \
  rm /jre/bin/servertool && \
  rm /jre/bin/tnameserv && \
  rm /jre/bin/unpack200 && \
  rm /jre/lib/ext/nashorn.jar && \
  rm /jre/lib/jfr.jar && \
  rm -rf /jre/lib/jfr && \
  rm -rf /jre/lib/oblique-fonts && \
  rm -rf /tmp/* /var/cache/apk/* && \
  echo 'hosts: files mdns4_minimal [NOTFOUND=return] dns mdns4' >> /etc/nsswitch.conf

ENV JAVA_HOME /jre
ENV PATH ${PATH}:${JAVA_HOME}/bin
ENV LANG en_US.UTF-8
```

Notice how I have merged the two `RUN` instructions. This is mainly because it's better to use a smaller amount of intermediary layers, especially since this is supposed to be a container everybody uses as a building block. If you want to find out more about Docker, containers, images and layers I recommend [the official documentation on what layers are and how images are using them to their benefit](https://docs.docker.com/engine/userguide/storagedriver/imagesandcontainers/).

_As a rule of thumb: More layers if you want more flexibility, less layers if you want to save on size and complexity. It depends on your preferences._

I also added a very small step at the end:

```Docker
ENV LANG en_US.UTF-8
```

This is the ensure we are running a clean environment with a defined language. Some application might expect to find these values already set, some might be setting them yourself. Obviously, your preference in language might also differ. [You can adjust the LANG parameter to your liking](https://www.gnu.org/software/gettext/manual/html_node/Locale-Environment-Variables.html).

## Where did the CMD instruction go?

I mentioned before, we are building an image meant to be used as a foundation for other services. As such, it doesn't need to carry a `CMD` instruction, as it will never be run "verbatim", but rather with a service attached to it which will occupy the `CMD` instruction.

You can still run commands inside the container by either using `docker run` or `docker exec`. For running a shell in your newly created container, use the following command:

```bash
$ docker run --rm -ti my-java-base-image /bin/bash
```
The last statement at the end of the `docker` command will be executed instead of the container's `CMD` instruction.

# Building the final image

Finally, we've reached a point where we can build our image:

```bash
$ docker build -t my-java-base-image .
Sending build context to Docker daemon 3.584 kB
Step 1 : FROM alpine:latest
 ---> 2314ad3eeb90
Step 2 : MAINTAINER Moritz Heiber <hello@heiber.im>
 ---> Running in 95ed975f09f9
 ---> 93cc2bc0bd60
Removing intermediate container 95ed975f09f9
Step 3 : ENV JAVA_VERSION_MAJOR 8
 ---> Running in 222af26f76fb
 ---> d03512eacddf
Removing intermediate container 222af26f76fb
Step 4 : ENV JAVA_VERSION_MINOR 72
 ---> Running in bace6309bed5
 ---> 1ef907c5fc42
Removing intermediate container bace6309bed5
Step 5 : ENV JAVA_VERSION_BUILD 15
 ---> Running in e711d3930716
 ---> fc58167137f3
Removing intermediate container e711d3930716
Step 6 : ENV JAVA_PACKAGE server-jre
 ---> Running in c0a11671c3a2
 ---> 1551bf0e99c3
Removing intermediate container c0a11671c3a2
Step 7 : ENV GLIBC_PKG_VERSION 2.22-r5
 ---> Running in 9932f9681e32
 ---> ed65857df324
Removing intermediate container 9932f9681e32
Step 8 : WORKDIR /tmp
 ---> Running in d9712f67c5d9
 ---> e768637ed058
Removing intermediate container d9712f67c5d9
Step 9 : RUN apk add --no-cache --update-cache curl ca-certificates bash &&   curl -Lo glibc-${GLIBC_PKG_VERSION}.apk "https://github.com/andyshinn/alpine-pkg-glibc/releases/download/${GLIBC_PKG_VERSION}/glibc-${GLIBC_PKG_VERSION}.apk" &&   curl -Lo glibc-bin-${GLIBC_PKG_VERSION}.apk "https://github.com/andyshinn/alpine-pkg-glibc/releases/download/${GLIBC_PKG_VERSION}/glibc-bin-${GLIBC_PKG_VERSION}.apk" &&   apk add --allow-untrusted glibc-${GLIBC_PKG_VERSION}.apk &&   apk add --allow-untrusted glibc-bin-${GLIBC_PKG_VERSION}.apk &&   /usr/glibc-compat/sbin/ldconfig /lib /usr/glibc-compat/lib &&   curl -jksSLH "Cookie: oraclelicense=accept-securebackup-cookie"   "http://download.oracle.com/otn-pub/java/jdk/${JAVA_VERSION_MAJOR}u${JAVA_VERSION_MINOR}-b${JAVA_VERSION_BUILD}/${JAVA_PACKAGE}-${JAVA_VERSION_MAJOR}u${JAVA_VERSION_MINOR}-linux-x64.tar.gz" | gunzip -c - | tar -xf - &&   apk del curl ca-certificates &&   mv jdk1.${JAVA_VERSION_MAJOR}.0_${JAVA_VERSION_MINOR}/jre /jre &&   rm /jre/bin/jjs &&   rm /jre/bin/keytool &&   rm /jre/bin/orbd &&   rm /jre/bin/pack200 &&   rm /jre/bin/policytool &&   rm /jre/bin/rmid &&   rm /jre/bin/rmiregistry &&   rm /jre/bin/servertool &&   rm /jre/bin/tnameserv &&   rm /jre/bin/unpack200 &&   rm /jre/lib/ext/nashorn.jar &&   rm /jre/lib/jfr.jar &&   rm -rf /jre/lib/jfr &&   rm -rf /jre/lib/oblique-fonts &&   rm -rf /tmp/* /var/cache/apk/* &&   echo 'hosts: files mdns4_minimal [NOTFOUND=return] dns mdns4' >> /etc/nsswitch.conf
 ---> Running in 07d9d69bb0f6
fetch http://dl-4.alpinelinux.org/alpine/v3.3/main/x86_64/APKINDEX.tar.gz
fetch http://dl-4.alpinelinux.org/alpine/v3.3/main/x86_64/APKINDEX.tar.gz
fetch http://dl-4.alpinelinux.org/alpine/v3.3/community/x86_64/APKINDEX.tar.gz
fetch http://dl-4.alpinelinux.org/alpine/v3.3/community/x86_64/APKINDEX.tar.gz
(1/9) Installing ncurses-terminfo-base (6.0-r6)
(2/9) Installing ncurses-terminfo (6.0-r6)
(3/9) Installing ncurses-libs (6.0-r6)
(4/9) Installing readline (6.3.008-r4)
(5/9) Installing bash (4.3.42-r3)
Executing bash-4.3.42-r3.post-install
(6/9) Installing openssl (1.0.2f-r0)
(7/9) Installing ca-certificates (20160104-r2)
(8/9) Installing libssh2 (1.6.0-r0)
(9/9) Installing curl (7.47.0-r0)
Executing busybox-1.24.1-r7.trigger
Executing ca-certificates-20160104-r2.trigger
OK: 15 MiB in 20 packages
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   609    0   609    0     0   1006      0 --:--:-- --:--:-- --:--:--  1009
100 2867k  100 2867k    0     0   719k      0  0:00:03  0:00:03 --:--:--  918k
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   613    0   613    0     0   1132      0 --:--:-- --:--:-- --:--:--  1137
100 1732k  100 1732k    0     0   506k      0  0:00:03  0:00:03 --:--:--  684k
(1/1) Installing glibc (2.22-r5)
OK: 19 MiB in 21 packages
(1/1) Installing glibc-bin (2.22-r5)
OK: 21 MiB in 22 packages
(1/4) Purging curl (7.47.0-r0)
(2/4) Purging ca-certificates (20160104-r2)
(3/4) Purging openssl (1.0.2f-r0)
(4/4) Purging libssh2 (1.6.0-r0)
Executing busybox-1.24.1-r7.trigger
OK: 19 MiB in 18 packages
 ---> 44974e8beb52
Removing intermediate container 07d9d69bb0f6
Step 10 : ENV JAVA_HOME /jre
 ---> Running in 1a373dc6f66e
 ---> 132c9c576162
Removing intermediate container 1a373dc6f66e
Step 11 : ENV PATH ${PATH}:${JAVA_HOME}/bin
 ---> Running in 89d4ecb314b4
 ---> f1fbbb7fa768
Removing intermediate container 89d4ecb314b4
Step 12 : ENV LANG en_US.UTF-8
 ---> Running in ac38d442c8d8
 ---> 911a6604cec9
Removing intermediate container ac38d442c8d8
Successfully built 911a6604cec9
```

Woohoo! It finished successfully. To make sure it actually did what we asked for, let's try running `java` inside the container, shall we?

```bash
$ docker run -ti --rm my-java-base-image java -version
java version "1.8.0_72"
Java(TM) SE Runtime Environment (build 1.8.0_72-b15)
Java HotSpot(TM) 64-Bit Server VM (build 25.72-b15, mixed mode)
```

Brilliant! That's exactly what we wanted. Now we have a full-fledged base image with Oracle's JRE ready to be used by another application. In the future, the only thing you need to do is to use your own base image as a `FROM` instruction in your application's `Dockerfile`:

```Docker
FROM my-java-base-image

[...]
```

## How large is the resulting image?

Let's find out:

```bash
$ docker images | grep my-java-base-image | awk '{print $7,$8}'
121.2 MB
```

That is quite large to be honest. Our original image clocked in at a mere 9MB.

But that's Java for you, I guess ;)

# Conclusion

We have built a solid, small and efficient Docker container image together which is capable of running pretty much any Java application you throw at it. Of course, there are edge cases for which you will have to adapt the configuration, but the general ideas behind it, starting small, growing carefully, using secure sources for building your image, are transcending these probable changes.

Once you realize that a Docker container shouldn't be anything else but a barebones, single-process container for your application you can start focussing on just the essentials without having to care about any of cruft that's usually pulled in alongside regular setup routines.

## A few simple guidelines

- **Run one process per container**: If you need multiple processes (logging, monitoring, databases etc.) build a composition of containers, utilizing tools like [docker-compose](https://docs.docker.com/compose/overview/). An example of this paradigm is the [Piwigo docker-compose configuration](https://github.com/moritzheiber/piwigo-docker) I built a short while ago.
- **Start (very) small**: You don't need a whole Debian or Ubuntu image to get started, especially if you're using languages that compile to static binaries (i.e. C/C++/Golang). Almost every time a small Alpine Linux image + your compiled application "just works".
- **Be efficient with your layering**: Adding more layers onto an image allows for easier tagging/debugging, but it will bloat your container ecosystem over time. Try to manage your container layers as you manage your container size.
- **Security is vital, be sure which images you pull from**: If you are not, you're essentially using unsigned, unverified images. [Docker has started working on this issue](https://docs.docker.com/engine/security/trust/content_trust/) but there still is a long way to go. The images we used, as well as the glibc package and the JRE, are either obtained from official sources (Docker distributed the Alpine Linux image; Oracle's JRE is downloaded directly from their servers) or signed/verified throughout the installation process (glibc package).

**Now, go ahead and build small, lean and efficient containers!**

## Feedback

I hope you enjoyed this article. Should you have comments, questions or suggestions (or even constructive critism ) let me know on [Twitter](https://twitter.com/moritzheiber) or [write me an email](mailto:hello@heiber.im).

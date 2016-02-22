+++
categories = ["docker", "devops", "aws"]
date = "2016-02-21T16:21:11+01:00"
description = ""
keywords = ["docker", "heka", "mozilla", "logging", "kinesis", "aws"]
title = "Building a Kinesis-enabled heka package for logging on AWS"
draft = true

+++

[Heka](https://github.com/mozilla-services/heka) is an excellent logging and processing framework made by none other than [Mozilla](https://mozilla.org), the creators of [Firefox](https://getfirefox.com). There are other frameworks like [logstash](https://www.elastic.co/products/logstash), [fluentd](https://fluentd.org), or proprietary solutions like [Loggly](https://loggly.com) or [Splunk](https://www.splunk.com). But none of them combine the flexibility of heka, through its included [Lua](https://www.lua.org) scripting engine, and the scalability, through its modular engine based on [Golang](https://golang.org), with the extensibility through its [plugin ecosystem](http://hekad.readthedocs.org/en/latest/developing/plugin.html) and versatile processing workflow engine.

If you were starting fresh I would probably recommend fluentd. It's easy to get started with (it literally takes a couple of seconds to write an appropriate `Gemfile`), it has a powerful processing pipeline of its own and a vast plugin ecosystem. It even comes with built-in high availability. It lets you take on your task with data driven discussions quickly and easily.

Unfortunately its very core, Ruby, is also its biggest disadvantage. When you're talking about a couple of 100 of megabytes a day, fluentd will get you there no problems. But once you're hitting a couple of 100 of gigabytes of logging and metric data a day it's going to falter sooner or later, especially if you're doing pre- and post-processing in the same pipeline.

This is where heka comes in.

Heka is written in Golang, making it a very small, portable package. With the available plugins, and its extensibility for processing and filtering offered by the internal Lua parser engine, it's perfectly capable of scaling to pretty much any workload you throw at it.

## Streaming logs with heka and Kinesis

A traditional setup with a logging daemon running somewhere would mean aggregating logs, either by tailing local files, or accepting messages on a socket or port in a structured format such as JSON or messagepack. And that's all right.

I've been working with a lot of projects deployed to AWS lately. Decoupling the log streaming from the transport, the processing and the storage obviously makes sense. For streams you can chose either SQS or Kinesis on AWS to use readily available services. I usually go for Kinesis over SQS for [many reasons](http://aws.amazon.com/kinesis/streams/faqs/), but mostly it's the streaming part of what Kinesis has tooffer and its seamless Lambda integration that seals the deal for me. Also, since you're essentially just paying for a stream of data and not per-record events, Kinesis tends to be a lot cheaper than SQS for large volume data transfers.

Also, with the addition of [Kinesis Firehose](https://aws.amazon.com/blogs/aws/amazon-kinesis-firehose-simple-highly-scalable-data-ingestion/) lately, processing huge amounts of already aggregated data has become even easier with Kinesis. Try doing that with SQS. Your billing department will not like it.

### Kinesis as an output for heka

[There is a sizable list of available output plugins](http://hekad.readthedocs.org/en/latest/config/outputs/index.html) for heka. Unfortunately, Kinesis isn't one of the natively developed plugins. You will have to build it yourself.

Fear not though, [there is a solution provided by the community](https://github.com/crewton/heka-plugins). When you are trying to use it you will probably want to use the [slightly patched-up version](https://github.com/MattLTW/heka-plugins) of my colleague [Matthew Lloyd](https://github.com/MattLTW). It has support for the latest version of heka, 0.10.0 (and beyond).

## Building your own heka package with Docker

Docker seems like the ideal way of building a system package. It's isolated from the overall operating system you're building it, your build environment can be set up in an instant, over and over again, and there are no remnants left behind afterwards.

I'm going to use the following `Dockerfile` for building a heka package with the Kinesis plugin enabled:

```Dockerfile
FROM golang:latest
MAINTAINER Moritz Heiber <hello@heiber.im>

ARG uid

RUN apt-get update && \
    apt-get install -y curl cmake git mercurial build-essential debhelper rpm

RUN useradd --uid ${uid} -mU heka

# 0.10.0
ENV version v0.10.0

RUN git clone -n https://github.com/mozilla-services/heka /tmp/heka
WORKDIR /tmp/heka
RUN git checkout ${version}

# Add the kinesis plugin
RUN sed -i '152igit_clone\(https\:\/\/github.com\/vaughan0\/go-ini\ a98ad7ee00ec53921f08832bc06ecf7fd600e6a1\)\ngit_clone\(https\:\/\/github.com\/aws\/aws-sdk-go\ 90a21481e4509c85ee68b908c72fe4b024311447\)\nadd_dependencies\(aws-sdk-go\ go-ini\)' cmake/externals.cmake
RUN echo "add_external_plugin(git https://github.com/MattLTW/heka-plugins.git master kinesis)" >> cmake/plugin_loader.cmake

ADD build-heka.sh .
RUN chown -R heka:heka /tmp/heka

VOLUME /tmp/heka-build

USER heka
CMD ["./build-heka.sh"]
```

### Step by step

```Dockerfile
ARG uid
```

This is a build argument, only introduced to Dockerfiles recently. You specify it when you run `docker build`. In this instance it helps us provide an image which belongs to the user running the package build. More on this later.

```Dockerfile
RUN apt-get update && \
    apt-get install -y curl cmake git mercurial build-essential debhelper rpm
```

These are all the required packages for building heka. Since the `golang:latest` `Dockerfile` is based on Debian Jessie we have the `apt-get` package manager at our disposal. Heka is using `git` and `mercurial` internally to manage dependencies.

```Dockerfile
RUN useradd --uid ${uid} -mU go
```

This adds a user with the same system id of the user running the Docker build. This is important because otherwise the package coming out of the container is going to belong to the user root, which you might be able to copy somewhere, but especially coming out of a pipeline it's going be a tough problem to deal with.

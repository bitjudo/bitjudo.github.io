---
layout: post
title: "Building Efficient Dockerfiles - Node.js"
date: 2014-03-13 09:34:02 -0400
comments: true
author: David Weinstein
categories: docker, nodejs, dockerfiles
---

TL;DR
====

Use the following code snippet (or a variation) after all your app dependencies
but before you ADD your app code to the container... this way you don't rebuild
your modules each time you re-build your container. If your `package.json` file
changes then your modules will be rebuilt.

{% codeblock lang:text Add this to your Dockerfile https://gist.github.com/dweinstein/9468644 gist %}
ADD package.json /tmp/package.json
RUN cd /tmp && npm install
RUN mkdir -p /opt/app && cp -a /tmp/node_modules /opt/app/
{% endcodeblock %}

Using cached layers for modules
===============================

This article is about making efficient use of docker layers, and as a side
effect how to reduce development and debugging time for Node.js applications
hosted in Docker containers. As you migrate from developing everything on your
host system into Docker, there are some growing pains, mainly arround rapid
change and test workflows.

<!-- more -->

There was a time when each time I'd make a slight modification to my
application, I'd spend time waiting for my docker container to reubild, usually
waiting for the modules to be reinstalled.  I'd spend more time waiting for the
dependencies to rebuild than actually fixing the problem.  I hope this article
helps others get out of that ditch...

{% pullquote %}

Writing an efficient `Dockerfile` is part of the fun of working with Docker as a
new technology. {" Docker forces you to think a little differently and once you
get in the right mindset you'll find yourself inventing new tricks. "}

{% endpullquote %}

One key is to understand how Docker layers work. For now I'll direct you to the
[documentation](http://docs.docker.io/en/latest/terms/layer/) to see the
graphic showing the various layers. Commands in your `Dockerfile` will create
new layers. When possible, docker will try to use an existing cached layer if
it's possible. You should try to take advantage of layers as much as possible
by organizing your commands in a specific order. We'll get into that order in a
second for dealing with node modules in your application.

{% codeblock lang:javascript Example - package.json %} 
{
  "name": "myApp",
  "description": "This is my awesome app...",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "docker.io": "*",
    "redis": "*",
    "restify": "*"
  }
}
{% endcodeblock %}

This snippet should generally go after all dependencies for your application
have been installed, and just before you add your application's code to the
container.

So here's a full example:

```
FROM ubuntu
MAINTAINER David Weinstein <david@bitjudo.com>

# install our dependencies and nodejs
RUN echo "deb http://archive.ubuntu.com/ubuntu precise main universe" > /etc/apt/sources.list
RUN apt-get update
RUN apt-get -y install python-software-properties git build-essential
RUN add-apt-repository -y ppa:chris-lea/node.js
RUN apt-get update
RUN apt-get -y install nodejs

# use changes to package.json to force Docker not to use the cache
# when we change our application's nodejs dependencies:
ADD package.json /tmp/package.json
RUN cd /tmp && npm install
RUN mkdir -p /opt/app && cp -a /tmp/node_modules /opt/app/

# From here we load our application's code in, therefore the previous docker
# "layer" thats been cached will be used if possible
WORKDIR /opt/app
ADD . /opt/app

EXPOSE 3000

CMD ["node", "server.js"]
```

The idea here is that if the `package.json` file changes (line 14) then Docker
will re-run the `npm install` sequence (line 15)... otherwise Docker will use
our cache and skip that part. This way our modules are cached so we aren't
rebuilding them every time we change our apps source code!



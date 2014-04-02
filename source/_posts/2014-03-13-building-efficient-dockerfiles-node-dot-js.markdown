---
layout: post
title: "Building Efficient Dockerfiles - Node.js"
date: 2014-03-13 09:34:02 -0400
comments: true
author: David Weinstein
categories: docker nodejs dockerfiles
---

TL;DR
====

Use the following code snippet (or a variation) after all your app dependencies
but before you ADD your app code to the container... this way you don't rebuild
your modules each time you re-build your container. If your `package.json` file
changes then your modules will be rebuilt. See this
[gist](https://gist.github.com/dweinstein/9550188) for a full example.

{% codeblock lang:text Add this to your Dockerfile, after your deps, but before your app code. https://gist.github.com/dweinstein/9468644 gist %}

ADD package.json /tmp/package.json
RUN cd /tmp && npm install
RUN mkdir -p /opt/app && cp -a /tmp/node_modules /opt/app/
{% endcodeblock %}

Using cached layers for modules
===============================

This article is about making efficient use of docker layers. As a side effect
we'll see how to reduce development and debugging time for Node.js applications
hosted in Docker containers. As you migrate from developing everything on your
development host system to Docker, there are some growing pains... mainly
arround interactive modify-and-test workflows.

<!-- more -->

There was a time that whenever I'd make a slight modification to an application
I'd spend time waiting for docker containers to rebuild. Usually I was waiting
for the modules to be reinstalled.  I spent more time waiting for the
dependencies to build than actually fixing the problem. I hope this article
helps others get out of that cycle...

{% pullquote %}

Writing an efficient `Dockerfile` is part of the fun of working with Docker as
a new technology. {" Docker forces you to think a differently. Once you get in
the right mindset you'll find yourself inventing new tricks. "}

{% endpullquote %}

One key is to understand how Docker layers work. For now, visit the
[documentation](http://docs.docker.io/en/latest/terms/layer/) to see a graphic
showing the various layers involved with Docker. Commands in your `Dockerfile`
will create new layers. When possible, docker will try to use an existing
cached layer if it's possible. You should try to take advantage of layers as
much as possible by organizing your commands in a specific order. We'll get
into that order in a second for dealing with node modules in your application.

First here's an example dependency file for Node:

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

And here's what we're going to insert into our old Dockerfile:

{% codeblock lang:text Add this to your Dockerfile, after your deps, but before your app code. https://gist.github.com/dweinstein/9468644 gist %}
ADD package.json /tmp/package.json
RUN cd /tmp && npm install
RUN mkdir -p /opt/app && cp -a /tmp/node_modules /opt/app/
{% endcodeblock %}

This snippet should generally go after all dependencies of your application
are installed, but just before you add your application's code to the
container.

A *bad* `Dockerfile` could look like this:

{% gist 9550778 %} 

This is bad because we copy the app's working directory  on [line
12](https://gist.github.com/dweinstein/9550778#file-bad-dockerfile-L12)--which
has our `package.json`--`.` to our container and then build the modules. This
results in our modules being built everytime we make a change to a file in `.`.

Here's a full example of a better implementation:

{% gist 9550188 Dockerfile %}

The idea here is that if the `package.json` file changes ([line
14](https://gist.github.com/dweinstein/9550188#file-dockerfile-L14)) then
Docker will re-run the `npm install` sequence ([line
15](https://gist.github.com/dweinstein/9550188#file-dockerfile-L15))...
otherwise Docker will use our cache and skip that part. 

Here's a log showing how building our Docker container is now using the `cache`
for the module dependency step when building the Dockerfile shown earlier.


{% gist 9550105 %}

This whole example is contained in this
[gist](https://gist.github.com/dweinstein/9550188) so that you can repeat it
exactly as I have.

Assuming you've built the container once before (i.e., `docker build -t
testProject .`), and then uncommented [line
7](https://gist.github.com/dweinstein/9550188#file-server-js-L7) in our example
`server.js` the  above log shows what happens when we rebuild our container,
i.e., simulating a change to our app's logic. Looking at the log, on [line
32](https://gist.github.com/dweinstein/9550105#file-gistfile1-txt-L32) the
`cache` was used but on [line
38](https://gist.github.com/dweinstein/9550105#file-gistfile1-txt-L38) the
cache was *not* used...

Conclusion
==========

Now our modules are cached so we aren't rebuilding them every time we change
our apps source code! This will speed up testing and debugging nodejs apps.
Also this caching technique can work for ruby gems which we'll talk about in
another post.


---
layout: post
title: "Authentication for a Docker Registry"
date: 2014-03-12 10:48:18 -0400
comments: true
categories: docker
author: Erik Kristensen
---

Docker is an amazing tool that has only been around for a short while, but has taken the DevOps world by storm, and organizations from small to large are starting to use it from integration testing, to dynamically scaling an application's environment. 

Docker released an open-source registry that allows you to remote store images you create with docker, but what the did not release is an open-source index. The index is arguably (depending on your use case) the most important part, it handles the authentication layer of a registry. A registry can be used without and index, and if you are just wanting a place to store your images and you don't want to worry about who can access them, then no need to read on.

For my job I needed a way to host images securely, but still allow various people inside and outside my organization to access certain images, so in my spare time (and since no one had done it yet that I could find) I started writing an index using Node.JS. It is currently in its infancy, but it works as of right now.

The project lives on github at [docker-index](https://github.com/ekristen/docker-index) and it is setup as a trusted build and deployed to the public docker registry/index at https://index.docker.io/u/ekristen/docker-index/. Just run `docker pull ekristen/docker-index` to get started. Specific instructions on how to get the registry, index, and redis working together can be found on the project's [README](https://github.com/ekristen/docker-index/blob/master/README.md#how-to-use)

It has only one dependency at this time and that is redis, that will most likely change to use mongodb or maybe even mysql to store users, permissions, and other index data, but redis was a quick and simple medium when doing the intial prototyping.

My future plans for the index are to create an admin interface that facilitates the management of users, namespaces, repositories and their associate permissions. For now all that has to be done by editing the configuration file.

This was a big help for me, hopefully having this code out there can help others too.

If you are interested in helping out pull requests are always welcome. If you have feedback, let me know.
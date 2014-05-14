---
layout: post
title: "Partial Continuous Deployment with Docker and SaltStack"
date: 2014-05-13 21:10:57 -0400
comments: true
published: false
author: Erik Kristensen
categories: saltstack docker devops
---

If you haven't figured out by now, I'm a big fan of both [Docker](http://www.docker.io) and [SaltStack](http://www.saltstack.com). I've been using them both separately for a while now, but recently started using them together. Here's my first iteration of continuous deployment using Docker and SaltStack.

This article will show you how to use `SaltStack` to re-deploy a container when a new version becomes available. I've made a few assumptions: (1) you know what Docker and SaltStack are, (2) you understand what a SaltStack State is, and (3) that you will use a docker index/registry to pull in your docker images (check out my [docker-index](https://github.com/ekristen/docker-index) project).

This post will break down the state file to explain each step, so even if you are not a Salt guru it should generally make sense. I won't cover building docker images or how to trigger SaltStack to run the state file. (That might come in a future article.)

<!-- more -->

# The Salt State
What makes this all possible is a crafting a SaltStack state to run certain commands when certain conditions apply. For example you can have the state stop a running container if a new version of an image is available, but if the image hasn't changed then do nothing.

For the rest of this article we'll be using [docker index](https://github.com/ekristen/docker-index) (just an application I wrote) as the example application to deploy.

If you want to skip the explanation and just grab the whole state file, jump down the page a bit.

## The State Explained

### Step 1 - Always Pull the Most Recent Image
First we want to make sure to **ALWAYS** pull the latest image. We use different tags based on the environment, `:dev` for dev, `:qa` for quality assurance, `:prod` for production. This allows taggging specific versions for release to different environments, but also allows us to tag an application as its version number plus the environment it goes to. By making the state always perform a pull we make sure that the latest image pushed to the registry is pulled down and ready.

{% codeblock docker-index.sls lang:yaml %}
docker_index_image:
  docker.pulled:
    - name: index.docker.io/ekristen/docker-index:dev
    - force: True
    - order: 100
{% endcodeblock %}

### Step 2 -- Stop If New Image Available
Next, we check and see if the latest version is running. We want the state to be idempotent if possible.

{% codeblock docker-index.sls lang:yaml %}
docker_index_stop_if_old:
  cmd.run:
    - name: docker stop docker_index
    - unless: docker inspect --format "{{ "{{ .Image "}}}}" docker_index | grep $(docker images | grep "index.docker.io/ekristen/docker-index:dev" | awk '{ print $3 }')
    - require:
      - docker: docker_index_image
    - order: 111
{% endcodeblock %}

The `unless` basically looks at the list of available images and grabs the unique hash of `index.docker.io/ekristen/docker-index:dev` and then greps the `.Image` value from the `inspect` command of the current running container named `docker-index`. This works because in **Step 1** we made sure we pulled down the latest version of the `dev` tag.

### Step 3 - Remove If We Stopped
If we stopped the container in **Step 2** then we want to also remove the container so that we can re-deploy it in **Step 3** and start it in **Step 4**.

{% codeblock docker-index.sls lang:yaml %}
docker_index_remove_if_old:
  cmd.run:
    - name: docker rm docker_index
    - unless: docker inspect --format "{{ "{{ .Image "}}}}" docker_index | grep $(docker images | grep "index.docker.io/ekristen/docker-index:dev" | awk '{ print $3 }')
    - require:
      - cmd: docker_index_stop_if_old
    - order: 112
{% endcodeblock %}

The `unless` does the same as in **Step 2** and checks to ensure the existing container is using a different image then the latest one.

### Step 4 - Create the Container
Now that we have successfully stopped and removed the old container, we can install the new container, and start it up (**Step 5**)

This part of the state defines the container and installs it but does not start it.

{% codeblock docker-index.sls lang:yaml %}
docker_index_container:
  docker.installed:
    - name: docker_index
    - image: index.docker.io/ekristen/docker-index:dev
    - environment:
      - "REDIS_HOST": "192.168.1.100"
      - "REDIS_PORT": "6379"
      - "PORT": "5100"
    - ports:
      - "5100/tcp"
    - require:
      - docker: docker_index_image
    - order: 120
{% endcodeblock %}

### Step 5 - Start the Container
We are almost done. Finally we want to make sure the container is running.

**Note:** Unfortunately the documentation on the docker states is still pretty rough. Pay close attention when you are defining port bindings, they must be double indented to work properly!

{% codeblock docker-index.sls lang:yaml %}
docker_index_running:
  docker.running:
    - container: docker_index
    - port_bindings:
        "5001/tcp":
            HostIp: "0.0.0.0"
            HostPort: "5100"
    - require:
      - docker: docker_index_container
    - order: 121
{% endcodeblock %}


## The Whole State
Here is the state file in its entirety. 

{% codeblock docker-index.sls lang:yaml %}
docker_index_image:
  docker.pulled:
    - name: index.docker.io/ekristen/docker-index:dev
    - force: True
    - order: 100

docker_index_stop_if_old:
  cmd.run:
    - name: docker stop docker_index
    - unless: docker inspect --format "{{ "{{ .Image "}}}}" docker_index | grep $(docker images | grep "index.docker.io/ekristen/docker-index:dev" | awk '{ print $3 }')
    - require:
      - docker: docker_index_image
    - order: 111

docker_index_remove_if_old:
  cmd.run:
    - name: docker rm docker_index
    - unless: docker inspect --format "{{ "{{ .Image "}}}}" docker_index | grep $(docker images | grep "index.docker.io/ekristen/docker-index:dev" | awk '{ print $3 }')
    - require:
      - cmd: docker_index_stop_if_old
    - order: 112

docker_index_container:
  docker.installed:
    - name: docker_index
    - image: index.docker.io/ekristen/docker-index:dev
    - environment:
      - "REDIS_HOST": "192.168.1.100"
      - "REDIS_PORT": "6379"
      - "PORT": "5100"
    - ports:
      - "5100/tcp"
    - require:
      - docker: docker_index_image
    - order: 120

docker_index_running:
  docker.running:
    - container: docker_index
    - port_bindings:
        "5001/tcp":
            HostIp: "0.0.0.0"
            HostPort: "5100"
    - require:
      - docker: docker_index_container
    - order: 121
{% endcodeblock %}


# Conclusion
This solution works fairly well for my purpose. Docker is fast, and I plan to use this in a rolling restart configuration in production. The fact that it first stops the running container, removes it and then starts the new one isn't a big deal for me. In my other environments, a few second hiccup is perfectly acceptable while the old is removed and the new is started. 

I also have a few other tricks with Docker that I hope to be writing about soon, so please check back, and let me know what you thought about this article.

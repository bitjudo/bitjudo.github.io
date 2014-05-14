---
layout: post
title: "Docker Scripts - Updated"
date: 2014-05-07 21:35:28 -0400
published: true
author: Erik Kristensen
comments: true
categories: docker devops scripts
---

In a follow up to the original post [Docker Scripts](), Docker has made it so that you no longer need to have `jq` installed, you can do everything with the `docker` client's inspect command and its (semi)new format feature.

I've updated the scripts to use the format command.

<!-- more -->

## Update Scripts

### docker-get-ip
{% codeblock lang:bash docker-get-ip %}
# Usage: docker-get-ip (name or sha)
[ -n "$1" ] && docker inspect --format "{{ "{{ .NetworkSettings.IPAddress "}}}}" $1
{% endcodeblock %}

### docker-get-id
{% codeblock lang:bash docker-get-id %}
# Usage: docker-get-id (friendly-name)
[ -n "$1" ] && docker inspect --format "{{ "{{ .ID "}}}}" $1
{% endcodeblock %}

### docker-get-image
I find this one useful for DevOps uses, when I have the friendly name, I can compare the current running container's image with the one I expect it to have (like an update) and redeploy.
{% codeblock lang:bash docker-get-image %}
# Usage: docker-get-image (friendly-name)
[ -n "$1" ] && docker inspect --format "{{ "{{ .Image "}}}}" $1
{% endcodeblock %}

### docker-get-state
{% codeblock lang:bash docker-get-state %}
# Usage: docker-get-state (friendly-name)
[ -n "$1" ] && docker inspect --format "{{ "{{ .State.Running "}}}}" $1
{% endcodeblock %}


## Conclusion
These are just easy shortcuts, I'm sure you can write these in different ways or as aliases. Perhaps others will find this useful.

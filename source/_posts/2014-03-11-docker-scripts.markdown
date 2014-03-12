---
layout: post
title: "Docker Scripts"
date: 2014-03-11 19:37:06 -0400
comments: true
categories: docker devops scripts
author: Erik Kristensen
---

David (the other author on this blog) and I have been doing a lot of work with docker with respect to our daily jobs. We ended up needed to orchestrate the management of multiple docker containers. This was before we discovered projects like [maestro](https://github.com/toscanini/maestro) and [maetro-ng](https://github.com/signalfuse/maestro-ng). David found an awesome command line for parsing json called [jq](http://stedolan.github.io/jq/) and put it to use by creating a bash one liner to grab a container's IP address.

{% codeblock lang:bash docker-get-ip %}
# Usage: docker-get-ip (name or sha)
[ -n "$1" ] && docker inspect $1 | jq -r '.[0] | .NetworkSettings | .IPAddress' 
{% endcodeblock %}

I took it one step further and created several more one liners to get other various pieces of information from containers.

{% codeblock lang:bash docker-get-id %}
# Usage: docker-get-id (friendly-name)
[ -n "$1" ] && docker inspect $1 | jq -r '.[0] | .ID'
{% endcodeblock %}

{% codeblock lang:bash docker-get-port %}
# Usage: docker-get-port (name|sha) port/protocol
[ -n "$1" ] && docker inspect $1 | jq -r ".[0] | .NetworkSettings | .Ports | .[\"$2\"] | .[0] | .HostPort"
{% endcodeblock %}

The great news is that with `jq`, you can create simple one liners for just about anything with the `docker inspect` command and drop them in `/usr/loca/bin` and make your administrative life a little easier. I'm sure there are more useful ones, but those are the top 3 that David and I use that make things more simple.


---
layout: post
title: "Full Disk Encryption, Surving a Reboot without Intervention or Storing the Key"
date: 2014-03-13 20:49:00 -0400
comments: true
published: false
categories: devops saltstack luks encryption longread tutorial
---

Recently I was challenged with ensuring that all data on several of my servers was encrypted while at rest. Since these servers are linux, LUKS encryption made the most sense. The real challenge was how to provide full disk encryption without storing the encryption key on the server, and have the drive mount after a reboot without human intervention. The rest of this article is a TL;DR combined with a tutorial of sorts to help you set this up.

# TL;DR

By taking advantage of a couple features that [SaltStack]() brings to the table, you can automate the mounting of your LUKS volumes after the server has started. The Salt minion has the ability to run certain states (read scripts) upon start, which allows you to run a LUKS state that will ensure the volume exists, unlock it, and mount it. Salt states also allow you to make them dependent on each other or trigger other states to run, by using these features you can ensure that your services that require the encrypted volume only start after the volume is available.

Basically the salt minion starts and is told by its configuration to run luks encryption script, this script ensures the LUKS volume exists, unlocks and mounts the drive. The other script the minion runs (in my case) is a mongodb service to script to ensure that mongodb is running, and if it isn't start it. When the LUKS script runs, it requests pillar data from the salt master, included in this data is the LUKS encryption key. It is transferred using zeromq and AES encryption. The key is temporarily stored locally on the minion while the script runs, then is deleted.

What we end up getting is the ability to mount encrypted drives and start any dependent services while keeping the encryption key on a single server that can be monitored and controlled and temporarily transferring that key to the servers that need it to mount the drive. While this is not bulletproof, in my opinion it is better then storing the encryption key permanently on the server.

# The Quasi How-To

I'm not going to cover setup and/or configuring SaltStack, LVM or LUKS encryption. What I will provide is the example Salt state and pillar files, some steps you can use to get going, it should be pretty straight foward. 


## The Salt State

I original found this state and after a little bit of searching I cannot find the original place I found it, but I ended up modifying it a bit for my needs. If you know where it original came from, let me know make a note. The state ensures that the LVM I need is setup and then sets up the LUKS volume. If all that is already setup then the script will simply unlock and mount the LUKS volume at the the defined mount point.

*Assumptions:* You have a non-formatted drive attached to the server and you've defined that in your pillar data, make sure you assign the correct drive device, otherwise you might end up erasing data.

{% gist 9541650 %}

*Note:* Please note that this state will create the LVM physical and logical volumes or attempt too if that fails LUKS and mounting will fail too.

### Extra Credit

If you want to learn how to make states reliant on each other check out the `requires` directive in the salt documentation. If you want to notify states when another state is done running check out `require_in`.

## The Salt Pillar

The pillar contains the configuration for the LVM and LUKS volumes and the mount point. Ensuring that you populate this pillar with the appropriate data is important. 

{% gist %}

## Putting it all together

Assuming you've associated the salt state and pillar to the appropriate target you'll be able to use `highstate` or `state.sls` to setup your LUKS encrypted drive. To make sure this happens each time the server starts up make sure you edit the minion's configuration and setup `startup_states` to *highstate*.

You should be able to tell the minion to run the LUKS state and find your drive mounted. Once that happens do a quick reboot, you should also find a few secons to a minute after reboot your drive is mounted and accessible.

# Conclusion

I'm pretty happy with this solution, it allows my servers to be rebooted without human interaction and for the critical services to come online once their data is available. It keeps my encryption key from having to be stored on the server. 

Thoughts, comments, questions? Let me heard from you. Leave your comments below.
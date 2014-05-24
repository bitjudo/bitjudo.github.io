---
layout: post
title: "Just in Time Encryption Keys using SaltStack"
date: 2014-03-19 11:15:00 -0400
comments: true
categories: devops saltstack luks encryption longread tutorial
author: Erik Kristensen
---

Recently, I was challenged with ensuring the encryption of all data at rest for several servers.  Unlike laptops or desktops, server nodes need to be able to come up and down in response to various requests. When spinning up multiple nodes you definitely don’t want them waiting for human interaction. Enter SaltStack and LUKS volumes. The real challenge was how to provide full disk encryption without storing the encryption key itself on the server. 

Since these were Linux servers, LUKS encryption made the most sense. In essence what this tutorial describes is a way to provide “just in time” delivery of disk encryption keys. This is done using SaltStack features.

The rest of this article is a TL;DR combined with a tutorial of sorts to help you set this up.

## TL;DR

By taking advantage of a couple features that [SaltStack](http://www.saltstack.com/) brings to the table, it is possible to automate the mounting of your LUKS volumes after the server has started. The salt minion has the ability to run certain states (scripts) upon start.  This allows the user to run a LUKS state that will verify the existence of the volume, unlock it, and mount it. Using salt states also allows the user to build state dependencies, or trigger other states to run.  These features ensure that any services requiring the encrypted volume only start after the volume is available.

<!-- more -->

## How it Works

Upon initializing of SaltStack, the configuration tells the Salt minion to run two scripts.  One script is the LUKS encryption script, which confirms that the LUKS volume exists, then unlocks and mounts it.  When the LUKS script runs, it requests pillar data from the Salt master.  Included in the requested data is the LUKS encryption key, which is transferred using zeromq and AES encryption. The key is temporarily stored locally on the minion while the script runs, and is deleted upon completion.

The second script checks whether a particular service is dependent upon the encrypted volume is running.  If the specific service is found to not be running, the script will also initialize it.  In this particular case, the service is a mongodb service.  

Ultimately, we end up with the ability to mount an encrypted drive and start dependent services.  However, we have relegated the encryption key to a single separate server and only temporarily transfer it to the servers that need it, exactly when they need it.  In simpler terms, we’ve created a just-in-time delivery mechanism for an encryption key.  While this method may not be bulletproof, I have found it preferable to storing the encryption key permanently on the same server as the paired encrypted drive.


## The Quasi How-To

The below does not cover setup and/or configuring SaltStack, LVM or LUKS encryption.  Rather, it’s an example of salt state and pillar files, some steps you can use to get going. It should be pretty straightforward. 


### The Salt State

I originally found this state and after a little bit of searching I cannot find the original place I found it, but I ended up modifying it a bit for my needs. If you know where it originally came from, let me know and I’ll make a note. The state ensures that the LVM I need is setup and then sets up the LUKS volume. If all that is already setup then the script will simply unlock and mount the LUKS volume at the the defined mount point.

*Assumptions:* You have a non-formatted drive attached to the server and you've defined that in your pillar data, make sure you assign the correct drive device, otherwise you might end up erasing data.

{% gist 9541650 luks.state %}

**Note:** Please note that this state will create the LVM physical and logical volumes or attempt too if that fails LUKS and mounting will fail too.

**Caveat:** This script will initialize a single drive as part of an LVM, with some tweaking it could combine multiple drives. This script was design to bring a server up from scratch and have a LUKS volume mounted and ready to go.

#### Extra Credit

If you want to learn how to make states reliant on each other check out the `requires` directive in the salt documentation. If you want to notify states when another state is done running check out `require_in`.

### The Salt Pillar

The pillar contains the configuration for the LVM and LUKS volumes and the mount point. Ensuring that you populate this pillar with the appropriate data is important. 

* **password** is the encryption key for the LUKS volume.
* **vg_name** is used to define both the volume group but the logical volume name too.
* **mountpoint** is where you want the LVM and LUKS volume to be mounted.
* **dev_name** is the device that you want converted to LVM and then LUKS encrypted.

{% gist 9541650 luks.pillar %}

**Note:** I don't discuss how to do it, but you can duplicate this pillar and change the **password** for each server if you want.

### Putting it all together

Assuming you've associated the salt state and pillar to the appropriate target you'll be able to use `highstate` or `state.sls` to setup your LUKS encrypted drive. To make sure this happens each time the server starts up make sure you edit the minion's configuration and setup `startup_states` to *highstate*.

You should be able to tell the minion to run the LUKS state and find your drive mounted. Once that happens do a quick reboot, you should also find a few seconds to a minute after reboot your drive is mounted and accessible.

## Conclusion

I'm pretty happy with this solution, it allows my servers to be rebooted without human interaction and for the critical services to come online once their data is available. It keeps the encryption key from having to be stored on the server. Using this strategy you could easily expand this for other encryption technologies.

Thoughts, comments, questions? Let me hear from you. Leave your comments below.




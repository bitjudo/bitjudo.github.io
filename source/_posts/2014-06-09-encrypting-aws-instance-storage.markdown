---
layout: post
title: "Encrypting AWS Instance Storage"
date: 2014-06-09 20:43:52 -0400
comments: true
published: true
author: Erik Kristensen
categories: saltstack aws encryption luks ec2 storage
---

**Encrypted Data at Rest** is the big term that has been floating around for several years. Just recently AWS started offering encrypted EBS volumes, the only problem with that is you cannot encrypt Instance Storage (aka Ephemeral Storage) volumes or Root volumes. This solution will not work for Root volumes, but it will for the Ephemeral volumes. The only potential problem with the encrypted EBS volumes is that AWS retains controls of the encryption keys for you in their IAM system. However since you've chosen to use the cloud that might not be a problem. Thankfully using SaltStack and my previous trick [Just in Time Encryption Keys using SaltStack](), you can automatically encrypt your Instance Storage on your EC2 Instances giving you that extra layer of security. This can be extremely beneficial if you want to convey to your clients that you are doing **Full Disk Encryption** and you want the ability to use SSD storage instead of EBS volumes.

<!-- more -->

## Overview

Using the same principles from my previous post about using SaltStack to deliver the encryption keys you can automatically encrypt all instance storage very quickly and effectively using LUKS. You will need SaltStack installed, with the [EC2 Grains plugin](https://github.com/saltstack/salt-contrib/blob/master/grains/ec2_tags.py) (Please note that you'll need the Boto python plugin installed for it to work). The EC2 Grains plugin gathers all the metadata from about your instance and updates the minion's grain information. Using the EC2 grains we can find all attached ephemeral (aka instance storage) disks and encrypt them at boot.

Current this state has only been tested on Ubuntu systems, but should be easily expandable to others as well, just need to make sure the proper LVM packages are installed.

This state also stores the encrypt volume paths in a grain, so that you can access them from other states in Salt.

## The Breakdown

**Lines 1 and 2** are basic setup for the state, if you want to set a custom password, just make sure you set a `instanceluks:password` pillar data. You can also do per volume or drive passwords, on **Line 25**

**Lines 4 through 16** install the necessary crypsetup and LVM for ubuntu, with a little bit of work other operating systems can be supported.

**Line 18** starts the loop through the instance storage (ephemeral disk)

**Line 19** checks to see if the ephemeral disk exists, **Note:** Earlier I stated you had to install the ec2_tags.py grains module, this is why.

**Line 25** if you set `instanceluks:passwords:NAME` in your pillar data, the state will use it, else it will use the global password set on **Line 2**.

The rest of the state are all the state commands to check if the LUKS partition has already been created, if not then create it, otherwise mount it after unlocking the volume.

After each volume is encrypted and mounted the state stores the volume mount path back into the hosts grains so that other states can access it, see **Lines 60-63**

## The Full State

{% gist 8f88244ba6f8c253cfca instanceluks.state %}

## Conclusion

If you find this useful let me know. As always comments, questions, critique, etc is always welcome. Cheers!
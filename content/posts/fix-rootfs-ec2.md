+++
title = "Fix RootFS related errors in your failed EC2 Linux instance"
date = 2020-10-25T19:05:44+05:30
images = []
tags = []
categories = ["ec2", "troubleshooting", "linux"]
draft = false
+++
![Amazon EC2](/posts/fix-rootfs-ec2/ec2-front.jpeg)

## TLDR;

If you face the RootFS related issue, try to check whether the AMI was having an older version of the Linux kernel. If it does, chroot into your volume and perform a kernel upgrade. Once done, detach the volume and create an AMI out of it. This will fix the 1/2 status checks issue.

## Context

If you have worked with Amazon EC2 in the past, there might be a chance that sometimes the EC2 instance fails to launch after a fresh start/stop-start of the instance. It even might have happened that you tried to launch from a very old AMI (which someone shared with you/you migrated it from another account) and the instance fails the 2/2 checks thing.

The catch-all solution could be to simply terminate the instance and create a new one or maybe create a snapshot first (if that instance contains any database or similar). Sometimes there are some edge cases and the new instance which was recreated also fails the 2/2 checks. I’m discussing one of the scenarios where it fails due to a RootFS mount issue.

## The scenario

In my case, I was performing a cross-account migration of AMIs between 2 AWS accounts. The instance was an Amazon Linux AMI (2017 version). I created the AMI and launched an instance from it successfully but it failed on the _"Instance Reachability"_ status check. I tried to terminate the instance and relaunch a new instance out of the same AMI, thinking it might be something related to the server hardware (sometimes they launch your instance in a degraded hardware) but again it failed the check. I checked on the system log and it said:

```
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
```

This cryptic message means that somehow the instance was unable to mount the EBS volume (in my case my instance type was having an EBS-backed volume).

On checking the **"Resolution"** section of the [troubleshooting page](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/TroubleshootingInstances.html#FilesystemKernel) it said the following:

```
Stop and modify to use modern kernel
```

Not really helpful.

## The resolution

After 10–15 minutes of searching around, I found a SO answer to update the kernel. However, you need to have the access of the system first to update the kernel in the first place.

Fortunately enough, my instance-type was having EBS-backed root volumes. So, I stopped the instance, detached it’s volume, created a recovery instance and attached the volume to the recovery instance. Also, make a note that simply attaching the volume to the instance doesn’t mean you can simply start reading from it. You need to create a mount point for your volume. This can be done by simply running these commands (run these from a root-level prompt):

```
lsblk # To check whether the device is available in system or not
mkdir /data
mount /dev/xvdf1 /data # In my case the volume was mounted as /dev/xvdf1. Yours might vary
```

After mounting the volume, the challenge was to use the volume. This was possible using the _chroot_ command. But first, you need to mount the required filesystem directories:

```
mount -t proc none /data/proc # mount the processes in the volume
mount --rbind /sys /data/sys # Recursively mount system directories
mount --rbind /dev /data/dev # recursively mount all devices
```

Now, we’re ready to chroot as simple as:

```
chroot /data /bin/bash
```

This will mount your bash shell to your volume’s filesystem as /.

Now, the kernel update (This is an Amazon Linux OS) was as simple as:

```
yum update -y
yum install -y kernel
```

After the kernel was updated, I stopped my recovery instance, safely unmounted the volume and re-attached the volume back to the original instance. After starting the instance, Voila! The failing check passed and thus the day was saved.

## Moral of the story

Always make sure that you perform kernel upgrades in the regular intervals so that whenever you need an AMI, you know that you’ll be getting an AMI that will atleast not fail due to an outdated kernel.
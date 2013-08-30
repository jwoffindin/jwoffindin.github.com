---
layout: post
title: "Migrating a AWS VPC between regions - part 1"
date: 2013-08-20 10:10
comments: true
categories: AWS
---

**Diclaimer: This post is a work in progress. It's disorganised and any
commands presented below are likey to destroy your production database,
so take the following with a large grain of salt**

This is first part of a series that describes the process we are going
through migrating AWS-based VPC instances from US region to AU and using
CloudFormation for setting up VPC. It is also an excuse to finally start
a blog and kick the tires of [Octopress].
 
[Octopress]: http://octopress.org/

## Background

We have two VPC networks on AWS with servers split across several
subnets. OS is Oracle Enterprise Linux 5.5. The original AMIs are no
longer provided by Oracle, and there are no corresponding Kernel (AKI)
or Ramdisk Image (ARI) in the `ap-southeast-2` region, so using AWS EC2
Copy [does not work][copy-amis].
   
Fortunately the target region supports [pv-grub] so the migration will
will also include making migrated server bootable via pv-grub.

So the plan is to:

1. Write script to copy root and data EBS volumes to the "target"
   region.
1. Update copied root volumes to make then bootable under pv-grub
   the target region.
1. Create a CloudFormation template referencing migrated AMIs so
   instances are created in the correct subnets and correct data volumes
   are mounted on the expected hosts.
1. Clean up after ourselves.

The first step is straight forward so I'll cover it here. The remaining
activities will be covered in their own posts.

## Tooling

For this project I've used the AWS command-line tools rather than one of
the SDKs. These tools can be installed using apt-get or brew (if you're
on OS/X or a Debian child).  See [AWS Documentation] installation and
usage instructions if you've any problems.

[AWS Documentation]: http://docs.aws.amazon.com/AWSEC2/latest/CommandLineReference/Welcome.html

```bash
$ sudo apt-get install ec2-api-tools ec2-ami-tools
$ brew install ec2-api-tools ec2-ami-tools
```

You will need to setup your AWS credentials for the ec2 tools to work.
I'm using the following simple shell script to set environment variables
required:

```bash env.sh
$ export EC2_CERT=~/.ec2/cert.pem
$ export EC2_PRIVATE_KEY=~/.ec2/private-key.pem
$ export JAVA_HOME="$(/usr/libexec/java_home)"
$ export EC2_HOME="/opt/boxen/homebrew/Library/LinkedKegs/ec2-api-tools/jars"
$ export EC2_URL=https://ec2.ap-southeast-2.amazonaws.com/
```

Note that I've explicitly set EC2_URL to avoid having to always specify
`--region ap-southeast-2` on every command.

## Migrating Root Images

Each server instance mounts two EBS volumes - a *system* EBS volume
(containing the root and boot partitions) and a *data* volume. 

The system volume is a 3 partition disk image:

* `/dev/sda1` - the `/boot` volume
* `/dev/sda2` - the root volume (mounted as '/')
* `/dev/sda3` - swap (more on this one later)

Below is a simple ZSH script I wrote to migrate root EBS
volumes from one region to another. It sputs out a CSV file 
which we'll need later to correlate the migrated instances against the
original instance ids.

{% gist 6373731 %} 

The current version only migrates the root images, will update later
with something a bit smarter.

Note that snapshotting the filesystems is a bit tricky since the OREL 5.5
systems aren't using LVM and don't support [fsfreeze].
As a workaround, the script will remount the root filesystem read-only,
trigger ec2 snapshot and then remount as writeable. The issue is that we
are not handling application consistency correctly (e.g. databases).

[pv-grub]: http://wiki.xen.org/wiki/PvGrub
[copy-amis]: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/CopyingAMIs.html



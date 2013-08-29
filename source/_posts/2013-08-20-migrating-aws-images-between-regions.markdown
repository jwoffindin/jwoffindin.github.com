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

This is first part of a series that describes the process I went through
migrating AWS-based VPC instances from US region to AU and using
CloudFormation for setting up VPC. It is also an excuse to finally start
a blog and kick the tires of [Octopress].

[Octopress]: http://octopress.org/

## Background

We have two VPC networks on AWS with servers split across several
subnets. OS is Oracle Enterprise Linux 5.5. The original AMIs are no
longer provided by Oracle, and there are no corresponding Kernel (AKI)
or Ramdisk Image (ARI) in the `ap-southeast-2` region, so using AWS EC2
Copy [does not work][1].

Fortunately the target region supports [pv-grub] so the migration will
will also include making migrated server bootable via pv-grub.

So, the plan is to:

1. Write script to copy root and data EBS volumes to the "target"
   region.
1. Update these copied root volumes to make then bootable under pv-grub
   the target region. Each server has an unknown set of changes made so
   we must migrate all instances.
1. Create a CloudFormation template referencing migrated AMIs so
   instances are created in the correct subnets and correct data volumes
   are mounted on the correct hosts.
1. Create the server instances using CloudFormation and test
1. Clean up after ourselves.

The first step is straight forward so I'll cover it here. The remaining
activities will be covered in their own posts.

## Migrating Root Images

Each of the instances has two EBS volumes - a *system* EBS volume
(containing the root and boot partitions) and a *data* volume. The
system volume is a disk image with 3 parititons:

* `/dev/sda1` - the `/boot` volume
* `/dev/sda1` - the root volume (mounted as '/')
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




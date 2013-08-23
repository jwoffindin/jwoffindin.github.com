---
layout: post
title: "Migrating AWS Images between regions"
date: 2013-08-20 10:10
comments: true
categories: AWS
published: false
---

We have a project for an Australian client with 3 VPC clusters running in AWS _us-west_
region (about 40 servers across test, uat and production). 
With the opening of a Sydney-based data centre, it makes sense to
migrate to the new location. Unfortunately we run into several issues:

* Servers are Oracle Enterprise Linux 5.5. The original AMIs are no longer
  provided by Oracle, and there are no corresponding Kernel (AKI) 
  or Ramdisk Image (ARI) in the `ap-southeast-2`, so using AWS EC2 Copy 
  [does not work][1]
* Images have an unknown amount of customisation applied, so starting
  from scratch isn't an attractive choice if we can avoid it.
* The instances and network configuration were done manually using AWS
  Console, so no artifacts for reuse.

First issue, we can't just snapshot/copy the AMIs because we don't have
matching AKIs/ARIs at the other end.

There are a [couple of presentations][migrate-presentation] which give an introduction to migrating
instances between regions. This should allow us to migrate the 40+
servers to the apac data center. 

I implemented a basic export/import script that used staging instances.
The script can be found [here](git@github.com:jwoffindin/aws-staged-copy.git)

[migrate-presentation]: http://www.slideshare.net/pleasebiteme/presentation-migrating-aws-ebs-backed-amis-between-regions

### Create image from snapshot

architecture `x86_64`
pv-grub AMI is `aki-3d990e07`
root device is `/dev/sda2` not `/dev/sda1`


####
```bash
ec2-attach-volume vol-f3c1bdc1 --instance i-55538e69 --device /dev/sdk --region ap-southeast-2

```


On target staging

```bash
bash <<EOF
device=$(cat /proc/partitions | awk '! /xvde/ && / xvd.[0-9]?/ { print
$4 ; exit }')
export chroot=/chroot
mount /dev/${device}2 ${chroot}
mount /dev/${device}1 ${chroot}/boot
for d in /dev /sys /dev/shm /proc ; do mount -o bind $d ${chroot}${d}; done
```

```bash
# take 1
$ sudo -i
root $ export d=/chroot
root $ mkdir $d
root $ mount -o bind /dev $d/dev
root $ mount -o bind /sys $d/sys
root $ mount -o bind /dev/shm $d/dev/shm
root $ mount -o bind /proc $d/proc
root $ chroot /chroot
root $ /sbin/mkinitrd -f -v  --builtin uhci-hcd --builtin ohci-hcd --builtin \
 ehci-hcd --preload xennet --preload xenblk --preload dm-mod --preload \
 linear --force-lvm-probe /boot/initrd-2.6.18-194.0.0.0.3.el5xen.img \
 2.6.18-194.0.0.0.3.el5xen
root $ exit
```

tidy up

```bash
[root@ip-10-240-10-89 ~]# for mount in /dev/shm /dev /proc /sys /boot / ; do umount ${d}${mount} ; done
```

Detaach the volume and create a snapshot

```bash
$ REGION=ap-southeast-2
$ ec2-detach-volume vol-f3c1bdc1 --region ${REGION}
$ ec2-create-snapshot vol-f3c1bdc1 --region ${REGION}
$ SNAP=snap-468d4475
```

Wait for it to become available

```bash
$ ec2-describe-snapshots ${SNAP} --region ${REGION}
```

and 
```
$ ec2-register --region ${REGION} --name "Test Image" --description "Created for testing migration " \
--architecture x86_64 --kernel aki-3d990e07 --root-device-name /dev/sda1 -s ${SNAP}
```

Launch the instance
```bash
$ ec2-run-instances --region ${REGION} ami-890a98b3
```

Wait for it 
```
$ INSTANCE_ID=i-a82cf294
ec2-get-console-output --region ${REGION} ${INSTANCE_ID} | less
```

crap.

### Plan B : Try as a single image, i.e. boot off /dev/sda rather than partition

The original image had 3 partitions:

* `/boot` on `/dev/sda1`
* `/` on `/dev/sda2`
* swap on `/dev/sda3`

If I understand

Start from a fresh copy, create a volume from copied snapshot
```
$ export SNAP=snap-8faa63bc
$ ec2-create-volume --snapshot ${SNAP} -z ${REGION}b
$ VOLUME_ID=vol-ee146fdc
$ ec2-describe-volumes ${VOLUME_ID}
$ ec2-attach-volume ${VOLUME_ID} --instance i-55538e69 --device /dev/sdk
```

and on the instance:
```bash
$ sudo -i
# e2fsck /dev/xvdo2
# time partimage -B=N  -z1 -d save /dev/xvdo2 /tmp/xvdo.partimg.gz -o -m -M
partimage: status: initializing the operation.
partimage: status: Saving partition to the image file...
partimage: status: reading partition properties
partimage: status: checking the file system with fsck
partimage: status: writing header
partimage: status: copying used data blocks
```



```bash
$ ec2-terminate-instances --region ${REGION} ${INSTANCE_ID}
$ ec2-attach-volume ${VOLUME_ID} --instance i-55538e69 --device /dev/sdk 
$ ssh staging.target
[ec2-user] $ sudo dd if=/dev/xvdo2 | gzip -c > /tmp/xvdo2.gz
...wait...
[ec2-user] $ gunzip -c /tmp/xvdo2.gz | sudo dd of=/dev/xvdo
[ec2-user] $ mount /dev/xvdo /mnt
[ec2-user] $ export version=
[ec2-user] $ echo <<EOF > /boot/grub/menu.lst
default 0
timeout 1
title My Linux Server (version)
        root (hd0)
        kernel /boot/vmlinuz-${version} ro root=LABEL=/
        initrd /boot/initrd-${version}.img
EOF
[ec2-user] $ umount /mnt
```

```bash
$ ec2-detach-volume vol-f3c1bdc1 --region ${REGION}
$ ec2-create-snapshot vol-f3c1bdc1 --region ${REGION}
$ SNAP=snap-468d4475
$ ec2-describe-snapshots ${SNAP} --region ${REGION}
$ ec2-register --region ${REGION} --name "Test Full-disk image" 
--architecture x86_64 --kernel aki-31990e0b --root-device-name /sda -s ${SNAP}
```

Launch the instance
```bash
$ ec2-run-instances --region ${REGION} ami-890a98b3
```

### Sidetrack : partimage

Should be able to speed up dump/restore of filesystes

```bash
$ sudo yum install partimage









 Step 1 : Create a "lab" environment to test the migration

I started two instances in `us-east-1` as a trial using `ami-63d2130a` AMI Oracle Linux
5.5 x86_64. One will be the image we want to migrate, the other is the
"helper" box that will be used for the migration.

```bash
$ mkdir -m 0700 ~/.ssh/aws
$ cp ~/Downloads/orel-migration-test.pem ~/.ssh/aws
$ chmod 0600 ~/.ssh/aws/*
```

add following `~/.ssh/config` entries. 

```
# Instane i-585c3f30
host box-1.source
  Hostname ec2-184-73-48-11.compute-1.amazonaws.com
  User root
  IdentityFile ~/.ssh/aws/orel-migration-test.pem
  PreferredAuthentications publickey

# Instance i-5a5c3f32
host staging.source
  Hostname ec2-54-211-116-254.compute-1.amazonaws.com
  User root
  IdentityFile ~/.ssh/aws/orel-migration-test.pem
  PreferredAuthentications publickey
```

Check connectivity

```bash
$ ssh box-1.source echo okay
...
okay
$ ssh staging.source echo okay
...
okay
```

# Step 2 : Prepare EBS volume for migration



[1]: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/CopyingAMIs.html


---
layout: post
title: "Migrating AWS Images between regions"
date: 2013-08-20 10:10
comments: true
categories: AWS
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
 linear /initrd-2.6.18-194.0.0.0.3.el5xen.img \
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
$ ZONE=${REGION}b
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

Start from a fresh copy, create a volume from copied snapshot:

```bash
$ export SNAP=snap-8faa63bc
$ ec2-create-volume --snapshot ${SNAP} -z ${REGION}b

# This is our source 3-partition image migrated
$ VOLUME_ID=vol-ee146fdc
$ ec2-describe-volumes ${VOLUME_ID}
$ ec2-attach-volume ${VOLUME_ID} --instance i-55538e69 --device /dev/sdk
```

Create volume which will be standalone. Note that the volume must not be
smaller than the original filesystem otherwise `partimage` will coredump.

```
$ ec2-create-volume -z ${ZONE} -s 11
$ VOLUME_ID=vol-a3097291 # copy & paste job
$ ec2-attach-volume ${VOLUME_ID} --instance i-55538e69 --device /dev/sdl
```

### Create EBS volume with single filesytem

On the staging box we export from /dev/sda2 (/xvdo2) to newly minted
EBS volume.

Dump the root partition to a file using partimage (which is faster than
doing plain `dd` as it only copies used blocks in the filesystem)

```bash
% e2fsck /dev/xvdo2
% partimage -B=N  -z1 -d save /dev/xvdo2 /tmp/xvdo.partimg.gz -o -m -M
partimage: status: initializing the operation.
partimage: status: Saving partition to the image file...
partimage: status: reading partition properties
partimage: status: checking the file system with fsck
partimage: status: writing header
partimage: status: copying used data blocks
partimage: status: commiting buffer cache to disk.
partimage: Success [OK]
partimage:  Operation successfully finished:
Time elapsed: 11m:13sec
Speed: 232.90 MiB/min
Data copied: 2.55 GiB
```

Then we copy the image onto new EBS volume.

```
% partimage restore /dev/xvdp /tmp/xvdo.partimg.gz.000 -f3 -B=N
partimage: status: initializing the operation
partimage: status: Restoring partition from the image file...
partimage: status: reading partition informations
partimage: status: reading header
partimage: status: copying used data blocks
partimage: status: commiting buffer cache to disk.
```

and then prepare the new volume for booting. We need to fix the 
grub menu and fstab.

```bash
$ mount /dev/xvdp /mnt
$ cp menu.lst.template /mnt/tmp
$ sudo chroot /mnt
% export version=2.6.18-194.0.0.0.3.el5xen
% cat /tmp/menu.lst.template | sed 's/${version}/'${version}/g > /boot/grub/menu.lst
% cat /boot/grub/menu.lst
default 0
timeout 1
title Test Server
        root (hd0)
        kernel /vmlinuz-2.6.18-194.0.0.0.3.el5xen ro root=LABEL=/
        initrd /initrd-2.6.18-194.0.0.0.3.el5xen.img
# /sbin/mkinitrd -f -v  --builtin uhci-hcd --builtin ohci-hcd \
--builtin ehci-hcd --preload xennet --preload xenblk --preload dm-mod \
--preload linear /initrd-${version}.img ${version} \
# vi /etc/fstab
/dev/sda               /                       ext3    defaults        1 1
tmpfs                   /dev/shm                tmpfs   defaults 0 0
devpts                  /dev/pts                devpts  gid=5,mode=620 0 0
sysfs                   /sys                    sysfs   defaults 0 0
proc                    /proc                   proc    defaults 0 0
# exit
$ sudo umount /mnt
```

```bash
$ ec2-detach-volume ${VOLUME_ID}
$ ec2-create-snapshot ${VOLUME_ID} 
$ SNAP=snap-468d4475
$ ec2-describe-snapshots ${SNAP}
$ ec2-register --name "Single filesystem image" --architecture x86_64 --kernel aki-31990e0b --root-device-name /dev/sda -s ${SNAP}
$ TEST_AMI=ami-b1089a8b
$ ec2-run-instances ${TEST_AMI}
$ TEST_I=i-6401fb58
$ ec2-describe-instance-status ${TEST_I}
$ ec2-get-console-output ${TEST_I}
```

If all is good:

```
Enterprise Linux Enterprise Linux Server release 5.5 (Carthage)
Kernel 2.6.18-194.0.0.0.3.el5xen on an x86_64

localhost.localdomain login:
```


Cleanup:

```bash
$ ec2-terminate-instances  ${TEST_I}
$ ec2-deregister ${TEST_AMI}
$ ec2-delete-snapshot ${SNAP}
```

## Old stuff - to be cleaned up. 

Describing migration approach

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
# Instance i-585c3f30
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



[1]: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/CopyingAMIs.html


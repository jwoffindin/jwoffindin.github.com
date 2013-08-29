---
layout: post
title: "Updating Migrated EBS volume to boot via pv-grub"
date: 2013-08-29 16:04
comments: true
categories: AWS
---

## Booting Migrated Volumes

Once we have root EBS snapshots migrated, high-level steps we need to go
through are:

1. Prepare a 'staging' server in the target availability zone
1. Create a Volume from copied snapshot
1. Make any requires changes to the file system by mounting it on the
   staging server
1. Unmount and snapshot the Volume
1. Create an AMI from the snapshot, setting the appropriate
   architecture, and *pv-grub* enabled AKI.
1. Boot the instance.
1. Check the console log to see if blows up somewhere along the way

If it fails, then terminate the instance, delete the snapshot and AMI
that we created in steps 4 onwards.

### Tooling

I'm on OS/X using Boxen / Brew to install api/ami tools, but an `apt-get`
should work just as well. See [AWS Documentation] for the official
documentation. 

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
```

In addition, also explicitly setting EC2_URL to the *ap-southeast-2*
endpoint to avoid having to always specify the `--region` parameter.

```bash
$ export EC2_URL=https://ec2.ap-southeast-2.amazonaws.com/
```

### Selecting correct AMI

I've stumbled across [this document][pv-grub-akis] which lists the
available pv-grub AKIs by region and disk layout. I'm not sure who the
original author was, apologies!

[pv-grub-akis]: /downloads/documents/convert-to-pv-grub.docx

For this case (root on `/dev/sda2`, and *ap-southeast-2* region), I will be using:

 property       | value
 ---------------|-----------------
 architecture   | `x86_64`
 pv-grub AKI    | `aki-3d990e07`


### Preparing the image

Assuming you have a snapshot `$SNAP_ID` and your staging instance-id
`$STAGING_INSTANCE_ID`, then the following will create a new volume
and mount it on your staging server.

I've written a small/dumb script for creating the volume and attaching
it to the staging instance. It can be found [here](https://gist.github.com/jwoffindin/6337827)

```bash Create and attach volume
$ ./create-volume-and-snapshot.sh snap-XXXXX i-YYYYYYY
```

TODO: Put instructions on making the image bootable

### Try it out

The next [script](https://gist.github.com/6338180) will create an AMI
from the candidate volume and boot it.


####
```bash
ec2-attach-volume vol-f3c1bdc1 --instance i-55538e69 --device /dev/sdk --region ap-southeast-2

```


On target staging

```bash
bash <<EOF
device=$(cat /proc/partitions | awk '! /xvde/ && / xvd.[0-9]?/ { print $4 ; exit }')
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


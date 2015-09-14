---
layout: post
comment: true
title: "Secondary disk partitioning during Kadeploy post-install"
date: 2015-09-14 09:33:00
tags:
- Grid5000
- Tutorial
- Kadeploy
---
Tagged with {{ page.tags | join: ' - ' }}

## Introduction

On [Grid'5000](https://www.grid5000.fr), in order to have a custom partition table, you need to use a `-t destructive` [OAR](https://oar.imag.fr) job 
and the `--set-custom-operations` option of [Kadeploy](http://kadeploy3.gforge.inria.fr) (See 4.4.4 page 69 from [kadeploy documentation](https://gforge.inria.fr/frs/download.php/file/34834/Kadeploy-3.3.4.stable.pdf)).
This method implies after each OAR job a deployment of the production environment in order to restore the correct partition table, also the node is unavailable longer than a job without `-t destructive` option.

At Rennes, you have a lot of servers with 2 or 5 disks (clusters parasilo, paravance and paranoia). These disks can be partitioned easily during deployment using the Kadeploy post-install.

## Create a custom post-install

### Extract the post-install comming with your environment

We will use the environment `jessie-x64-base` provided by Grid'5000.

{% highlight console %}
$ kaenv3 -p jessie-x64-base
...
postinstalls:
- archive: server:///grid5000/postinstalls/debian-x64-base-2.4-post.tgz
  compression: gzip
  script: traitement.ash /rambin
...
$ cd
$ mkdir -p envs/postinstall/debian-x64-custom-2.4-post
$ cd envs/postinstall/debian-x64-custom-2.4-post
$ tar xvzf /grid5000/postinstalls/debian-x64-base-2.4-post.tgz
dest/
dest/usr/
dest/usr/lib/
dest/usr/lib/g5k-checks/
dest/usr/lib/g5k-checks/data/
dest/usr/lib/g5k-checks/data/partitions
dest/etc/
dest/etc/securetty
dest/etc/fstab
dest/etc/iftab
dest/etc/inittab
prepostinst
traitement.ash

{% endhighlight %}

### Edit post-install

{% highlight console %}
$ touch partition.sh
$ chmod a+x partition.sh
{% endhighlight %}

Edit file `traitement.sh` and add before `exit 0` :

{% highlight sh %}
# Secondary disk partitioning
$POST_DIR/partition.sh $POST_DIR $DEST_DIR
{% endhighlight %}

Edit file `partition.sh` with this content :

{% highlight bash %}
#!/bin/bash

POST_DIR=$1
DEST_DIR=$2
PREPOST_DIR=$1/prepost
. $PREPOST_DIR/prevar.ash
REPORT_FILE=$DEST_DIR/root/postinst.log

if [[ $_CLUSTER =~ parasilo|paravance|paranoia ]]; then
  parted -s /dev/sdb mklabel msdos
  parted -s /dev/sdb --align optimal mkpart primary ext4 0 100%
  sleep 2
  mkfs.ext4 -U 820eb8c7-743c-4967-90fa-999db1d80f7a /dev/sdb1 > /dev/null
  mkdir $DEST_DIR/mnt/scratch
  echo "UUID=820eb8c7-743c-4967-90fa-999db1d80f7a       /mnt/scratch    ext4  defaults  0   0" >> $DEST_DIR/etc/fstab
fi
{% endhighlight %}

__Note__ : To use btrfs (or LVM...) you need to install needed packages :

{% highlight bash %}
if [[ $_CLUSTER =~ parasilo|paravance|paranoia ]]; then
  parted -s /dev/sdb mklabel msdos
  parted -s /dev/sdb --align optimal mkpart primary ext4 0 100%
  export http_proxy=http://proxy:3128
  apt-get update && apt-get install btrfs-tools
  mkfs.btrfs /dev/sdb1 > /dev/null
  mkdir $DEST_DIR/mnt/scratch
  echo "/dev/sdb1       /mnt/scratch    btrfs  defaults  0   0" >> $DEST_DIR/etc/fstab
fi
{% endhighlight %}

### Build the new post-install and create a custom environment

{% highlight console %}
$ mkdir -p ~/public/kadeploy/postinstall
$ cd ~/envs/postinstall/debian-x64-custom-2.4-post
$ tar cvzf ~/public/kadeploy/postinstall/debian-x64-custom-2.4-post.tgz *
$ mkdir ~/public/kadeploy/envs
$ cd ~/public/kadeploy/envs
$ kaenv3 -p jessie-x64-base > jessie-x64-custom.dsc
{% endhighlight %}

Edit the `postinstall` section of the file `jessie-x64-custom.dsc` with :

{% highlight yaml %}
postinstalls:
- archive: http://public.rennes.grid5000.fr/~pmorillo/kadeploy/postinstall/debian-x64-custom-2.4-post.tgz
{% endhighlight %}

## Deploy the new environment

{% highlight console %}
$ oarsub -I -t deploy -l {"cluster='parasilo'"}/nodes=1
$ kadeploy3 -a "http://public.rennes.grid5000.fr/~pmorillo/kadeploy/envs/jessie-x64-custom.dsc" -f $OAR_NODEFILE -k
Deployment #D-07ac6fbc-97fc-4772-b864-bc02b9d34d85 started
Grab the key file /home/pmorillo/.ssh/authorized_keys
Grab the tarball file server:///grid5000/images/jessie-x64-base-0.1.tgz
Grab the postinstall file http://public.rennes.grid5000.fr/~pmorillo/kadeploy/postinstall/debian-x64-custom-2.4-post.tgz
Launching a deployment on parasilo-13.rennes.grid5000.fr
...
The deployment is successful on nodes
parasilo-13.rennes.grid5000.fr
$ ssh root@parasilo-13
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda3        15G  995M   13G   8% /
udev             10M     0   10M   0% /dev
tmpfs            26G  9.0M   26G   1% /run
tmpfs            63G     0   63G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs            63G     0   63G   0% /sys/fs/cgroup
/dev/sdb1       551G   70M  522G   1% /mnt/scratch
/dev/sda5       525G   70M  498G   1% /tmp
{% endhighlight %}


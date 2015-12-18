---
layout: post
comment: true
title:  "Create a persistent database service on Grid'5000"
date: 2015-12-18 09:17:00
tags:
- Grid5000
- Tutorial
- Ceph
- KVM
---
Tagged with {{ page.tags | join: ' - ' }}

## Introduction

Usually on [Grid'5000](https://www.grid5000.fr), in order to deploy our own system we use [Kadeploy](http://kadeploy3.gforge.inria.fr).
The lifetime of the system is very short, the duration of the [OAR](https://oar.imag.fr) job.
To keep system modifications, you need to update the deployed Kadeploy environment before the end of OAR job.
This is sufficient in most simple cases, but for systems hosting a lot of datas usable through multiple experiments, it's not practical.

In this tutorial we will create a persistent system hosting a database, hosted on a node as a virtual machine with [KVM](http://www.linux-kvm.org), without deployment (only 20 secondes needed to start the service).
This virtual machine will be stored on the [Ceph](http://www.ceph.com) distributed object store available at Rennes.

![schema](/images/g5k_permanent_db_tuto_img1_v2.png)

## Manage your Ceph object store

### Generate your Ceph Key and create an RBD pool

On [Grid'5000 Ceph UI](https://api.grid5000.fr/sid/storage/ceph/ui/), click on `Generate Ceph key` (if no key appears) and create a Ceph pool name `username_rbd`.

![Ceph pool creation](/images/g5k_permanent_db_tuto_img2.png)

**Note** : A replication size of 3 is required for a safe storage.

## Create the virtual machine stored on a Rados Block Device

### Configure Ceph client and authentication

* From host `frontend.rennes.grid5000.fr` :

{% highlight console %}
G5K ❯ /home/pmorillo » mkdir ~/.ceph
G5K ❯ /home/pmorillo » cat > ~/.ceph/config <<EOF
[global]
mon host = ceph0,ceph1,ceph2
keyring = /home/user_login/.ceph/ceph.client.user_login.keyring
EOF
G5K ❯ /home/pmorillo » curl -k https://api.grid5000.fr/sid/storage/ceph/auths/$USER.keyring > ~/.ceph/ceph.client.$USER.keyring
{% endhighlight %}

__Notes__ : Replace `user_login` by your username.

### Create a Rados Block Device based on Debian 7 (Wheezy)

* Submit an interactive OAR job :

{% highlight console %}
G5K ❯ /home/pmorillo » oarsub -I -l walltime=2
[ADMISSION RULE] Modify resource description with type constraints
Generate a job key...
OAR_JOB_ID=736746
Interactive mode : waiting...
Starting...

Connect to OAR job 736746 via the node paranoia-7.rennes.grid5000.fr
G5K(736746) ❯ /home/pmorillo »
{% endhighlight %}

* From the node :

{% highlight console %}
G5K(736746) ❯ /home/pmorillo » export CEPH_CONF=~/.ceph/config
G5K(736746) ❯ /home/pmorillo » qemu-img convert -f qcow2 -O raw /grid5000/virt-images/wheezy-x64-base-1.4.qcow2 rbd:pmorillo_rbd/debian7-mysql:id=pmorillo
G5K(736746) ❯ /home/pmorillo » rbd --pool pmorillo_rbd --id pmorillo ls
debian7-mysql
{% endhighlight %}


### Start the virtual machine

{% highlight console %}
G5K(736746) ❯ /home/pmorillo » screen kvm -m 1024 -drive format=raw,file=rbd:pmorillo_rbd/debian7-mysql:id=pmorillo -nographic -device e1000,netdev=user.0 -netdev user,id=user.0,hostfwd=tcp::2222-:22
{% endhighlight %}

From host `frontend.rennes.grid5000.fr` :

{% highlight console %}
G5K ❯ /home/pmorillo » ssh -p 2222 root@paranoia-7
...
root@vm-mysql:~#
{% endhighlight %}


## Configure Virtual Machine and MySQL

### Set hostname

Here `apt-get update` command will fail because the system cannot resolve host `proxy`. [SLIRP](http://wiki.qemu.org/Documentation/Networking#User_Networking_.28SLIRP.29)
does not send by default DNS search field. Add FQDN in `/etc/hostname` file will quickly solve the problem.

Log in as root/grid5000, and :

{% highlight console %}
root@(none):~# echo "vm-mysql.rennes.grid5000.fr" > /etc/hostname
root@(none):~# sh /etc/init.d/hostname.sh

{% endhighlight %}


### Install and configure MySQL

{% highlight console %}
root@vm-mysql:~# apt-get update && apt-get install mysql-server
root@vm-mysql:~# mysql -u root
mysql> create database grid5000_test;
mysql> grant all privileges on grid5000_test.* to 'grid5000'@'%' identified by 'grid5000';
Query OK, 0 rows affected (0.00 sec)

mysql> exit
{% endhighlight %}

Edit `/etc/mysql/my.cnf`, comment line `bind-address = 127.0.0.1` and restart MySQL service :

{% highlight console %}
root@vm-mysql:~# service mysql restart
Stopping MySQL database server: mysqld.
Starting MySQL database server: mysqld ..
Checking for tables which need an upgrade, are corrupt or were 
not closed cleanly..
root@vm-mysql:~# shutdown -h now
{% endhighlight %}

### Restart VM with MySQL port forwarding

* from the node :

{% highlight console %}
G5K(736746) ❯ /home/pmorillo » screen kvm -m 1024 -drive format=raw,file=rbd:pmorillo_rbd/debian7-mysql:id=pmorillo -nographic -device e1000,netdev=user.0 -netdev user,id=user.0,hostfwd=tcp::13306-:3306
{% endhighlight %}


### Test

From host `frontend.rennes.grid5000.fr` :

{% highlight console %}
G5K ❯ /home/pmorillo » mysql -u grid5000 -h paranoia-7 --port 13306 -p                                                                                                                                                                                      frennes.rennes.grid5000.fr  1 ↵ 
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 37
Server version: 5.5.44-0+deb7u1 (Debian)

Copyright (c) 2000, 2014, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| grid5000_test      |
+--------------------+
2 rows in set (0.00 sec)

mysql>
{% endhighlight %}


## Conclusion

Now you can use a persistent database during multiple experiments.
Network performances are not optimal in this KVM configuration, but it's very simple to use, without privileges.


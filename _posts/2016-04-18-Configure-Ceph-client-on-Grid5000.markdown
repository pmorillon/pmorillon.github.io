---
layout: post
comment: false
title:  "Configure Ceph client on Grid5000"
date: 2016-04-18 19:49:00
tags:
- Grid5000
- Tutorial
- Ceph
---
Tagged with {{ page.tags | join: ' - ' }}

## Introduction

Ceph is installed on all Grid'5000 frontends and nodes (standard environment). Ceph clusters are available at Rennes and Nantes.

Ceph requires authentication to access to your data storage.

## On the frontend

Create a directory with Ceph configuration (below for Rennes, if you are at Nantes, replace rennes by nantes into the URL):

{% highlight console %}
G5K ❯ /home/pmorillo » mkdir ~/.ceph
G5K ❯ /home/pmorillo » cat > ~/.ceph/config <<EOF
[global]
mon host = ceph0,ceph1,ceph2
keyring = ${HOME}/.ceph/ceph.client.${USER}.keyring
EOF
G5K ❯ /home/pmorillo » curl -k https://api.grid5000.fr/sid/sites/rennes/storage/ceph/auths/$USER.keyring > ~/.ceph/ceph.client.$USER.keyring
G5K ❯ /home/pmorillo » export CEPH_CONF=~/.ceph/config
G5K ❯ /home/pmorillo » export CEPH_ARGS="--id ${USER}"
{% endhighlight %}

### Check the Ceph cluster status

{% highlight console %}
G5K ❯ /home/pmorillo » ceph -s
    cluster 6ee4c970-6c0d-43d6-82e4-e6d9881fd21d
     health HEALTH_OK
     monmap e3: 3 mons at {ceph0=172.16.111.30:6789/0,ceph1=172.16.111.31:6789/0,ceph2=172.16.111.32:6789/0}, election epoch 16, quorum 0,1,2 ceph0,ceph1,ceph2
     osdmap e3326: 16 osds: 16 up, 16 in
      pgmap v2886627: 8384 pgs, 59 pools, 2051 GB data, 516 kobjects
            4222 GB used, 4132 GB / 8802 GB avail
                8382 active+clean
                   2 active+clean+scrubbing+deep
{% endhighlight %}

### Configure your shell

Into your ~/.bashrc or ~/.zshrc, puts this two lines :

{% highlight console %}
export CEPH_CONF=~/.ceph/config
export CEPH_ARGS="--id ${USER}"
{% endhighlight %}

You can now use _ceph_ and _rbd_ commands from the frontend and nodes.

{% highlight console %}
G5K ❯ /home/pmorillo » oarsub -I
G5K(752498) ❯ /home/pmorillo » rbd --pool pmorillo_rbd ls
debian7-mysql
elasticsearch
jessie-x64-base
wheezy-x64-base
G5K(752498) ❯
{% endhighlight %}

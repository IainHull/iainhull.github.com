---
layout: post
title: "Centos in VirtaulBox on MacOSX"
description: ""
category: 
tags: [virtualbox,centos]
---
{% include JB/setup %}

I normally prefer to use Ubuntu server (I find more things work out of the box, software is easier to find and install and the command line has been pimped to be a little friendlier). However I need to explore Chef for work which means using Centos, so my first job was to set up some Centos virtual machines to play with.

First off I tried [Vagrant](http://www.vagrantup.com/about.html). This looks very good, but I had some problems with bridged networking and the Centos box I was using. So I moved on to plain old [VirtualBox](https://virtualbox.org/) and created my virtual machines from scratch.

This wasn’t as straightforward as I had hoped.  Some of this is probably because I am not as familiar with Centos, but some of this is just much harder than it should be.  I had to do a lot of searching for answers but I finally found some blogs that got me almost there.

Here is a list of things I had to do to get my machines working the way I wanted them.

## VirtualBox

Download and install it from here 

[http://dlc.sun.com.edgesuite.net/virtualbox/4.2.10/VirtualBox-4.2.10-84104-OSX.dmg](http://dlc.sun.com.edgesuite.net/virtualbox/4.2.10/VirtualBox-4.2.10-84104-OSX.dmg)

## CentOS

Then download CentOS and install it in a new virtual machine

[http://mirror.stshosting.co.uk/centos/6.4/isos/x86_64/CentOS-6.4-x86_64-minimal.iso](http://mirror.stshosting.co.uk/centos/6.4/isos/x86_64/CentOS-6.4-x86_64-minimal.iso)

## DHCP Networking

I don't know how many chef nodes I will need in the future, so I didn't want to manage static ip.  Here is what I had to do to get DHCP networking up starting from a minimal CentOS server

Set the hostname in `/etc/sysconfig/network`

    NETWORKING=yes
    HOSTNAME=chef-server
    DHCP_HOSTNAME=chef-server

Update `/etc/sysconfig/network-scripts/ifcfg-eth0`.  I had to change `ONBOOT` and `BOOTPROTO`.

    DEVICE=eth0
    HWADDR=08:00:27:EA:84:97
    TYPE=Ethernet
    UUID=f8a485aa-91cb-4ca8-b372-4397e8f45114
    ONBOOT=yes
    NM_CONTROLLED=yes
    BOOTPROTO=dhcp

Restart the networking and test

    # services network restart
    # ping google.com

## Zeroconf DNS

I want to access my vms by name however I don't want to setup a domain name server or use static ip addresses and /etc/hosts, so I configured [Zeroconf](http://en.wikipedia.org/wiki/Zero_configuration_networking) and [mDNS](http://en.wikipedia.org/wiki/Multicast_DNS).

This was a lot more difficult in CentOS than I expected but I finally found a good blog that helped me through it.

[http://theengguy.blogspot.ie/2013/02/mdns-centos-63.html](http://theengguy.blogspot.ie/2013/02/mdns-centos-63.html)

Short steps:

    # yum -y install avahi
    # rpm --import http://packages.atrpms.net/RPM-GPG-KEY.atrpms
    # rpm -ivh http://dl.atrpms.net/all/atrpms-repo-6-6.el6.x86_64.rpm
    # yum -y --enablerepo=atrpms install nss-mdns
    # service messagebus restart
    # chkconfig avahi-daemon on

Edit /etc/nsswitch.conf make sure your hosts line looks like this.

    hosts:      files mdns4_minimal dns

Reboot.  This almost works.

    # getent hosts chef-server.local
    # ping chef-server.local
    ping: unknown host chef-server.local

However iptables is blocking it.  I currently don't require iptables so I am just going to disable it.

    # service iptables stop
    iptables: Flushing firewall rules:                         [  OK  ]
    iptables: Setting chains to policy ACCEPT: filter          [  OK  ]
    iptables: Unloading modules:                               [  OK  ]
    # chkconfig iptables off

Now try again

    # getent hosts chef-server.local
    192.168.0.190   chef-server.local
    # ping chef-server.local
    PING chef-server.local (192.168.0.190) 56(84) bytes of data.
    64 bytes from 192.168.0.190: icmp_seq=1 ttl=64 time=0.942 ms
    64 bytes from 192.168.0.190: icmp_seq=2 ttl=64 time=0.320 ms
    ^C
    --- chef-server.local ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 1265ms
    rtt min/avg/max/mdev = 0.320/0.631/0.942/0.311 ms

### Installing Guest Additions

The guest additions enable the shared folders feature.  However installing this is not as straight forward as I hoped either.  Again I found a blog that got me almost there.

[http://www.if-not-true-then-false.com/2010/install-virtualbox-guest-additions-on-fedora-centos-red-hat-rhel/](http://www.if-not-true-then-false.com/2010/install-virtualbox-guest-additions-on-fedora-centos-red-hat-rhel/) (I had to add perl to the list of dev tools to install)

    # yum -y update kernel*
    # reboot
    # yum -y install gcc kernel-devel kernel-headers dkms make bzip2 perl
    
For some reason the symlink for uname -r didn't work

    # export KERN_DIR=/usr/src/kernels/`uname -r`
    
So I just used the name of the directory

    # export KERN_DIR=/usr/src/kernels/2.6.32-358.2.1.el6.x86_64/
    
Then continue

    # mount /dev/scd0 /mnt
    # /mnt/VBoxLinuxAdditions.run

The last few lines from the output look like this

    Building the VirtualBox Guest Additions kernel modules
    Building the main Guest Additions module                   [  OK  ]
    Building the shared folder support module                  [  OK  ]
    Building the OpenGL support module                         [FAILED]
    (Look at /var/log/vboxadd-install.log to find out what went wrong)
    Doing non-kernel setup of the Guest Additions              [  OK  ]
    Installing the Window System drivers                       [FAILED]
    (Could not find the X.Org or XFree86 Window System.)

I am not useing OpenGL and X so I am ignoring these, so reboot and try it out.
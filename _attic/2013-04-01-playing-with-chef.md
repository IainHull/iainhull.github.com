---
layout: post
title: "Playing with Chef"
description: ""
category: 
tags: [chef,devops]
---
{% include JB/setup %}

## chef-server

wget https://opscode-omnibus-packages.s3.amazonaws.com/el/6/x86_64/chef-server-11.0.6-1.el6.x86_64.rpm
yum -y localinstall /vagrant/chef-server-11.0.6-1.el6.x86_64.rpm
chef-server-ctl reconfigure

failed!!!

http://tickets.opscode.com/browse/CHEF-3837?focusedCommentId=33877&page=com.atlassian.jira.plugin.system.issuetabpanels:comment-tabpanel#comment-33877

/etc/hosts
    127.0.0.1   chef-server localhost localhost.localdomain localhost4 localhost4.localdomain4

chef-server-ctl reconfigure

now works

http://docs.opscode.com/install_workstation.html


# yum -y install git

install git


http://workstuff.tumblr.com/post/50911984233/some-tips-on-getting-started-with-vagrant-and-chef




---
layout: post
title: "Installing Ansible with Homebrew"
description: ""
category: 
tags: [ansible, homebrew]
---
{% include JB/setup %}

{% highlight bash %}

$ brew search ansible
No formula found for "ansible". Searching open pull requests...

$ brew tap mnot/stuff
Cloning into '/usr/local/Library/Taps/mnot-stuff'...
remote: Counting objects: 85, done.
remote: Compressing objects: 100% (58/58), done.
remote: Total 85 (delta 31), reused 81 (delta 27)
Unpacking objects: 100% (85/85), done.
Tapped 4 formula


$ brew install python


$ pip install paramiko
Downloading/unpacking paramiko
  Downloading paramiko-1.11.0.tar.gz (842kB): 842kB downloaded
  Running setup.py egg_info for package paramiko

Successfully installed paramiko pycrypto
Cleaning up...



$ sudo pip install jinja2
Downloading/unpacking jinja2
  Downloading Jinja2-2.7.tar.gz (377kB): 377kB downloaded
  Running setup.py egg_info for package jinja2

Successfully installed jinja2 markupsafe
Cleaning up...



$ brew install ansible
==> Installing ansible dependency: pkg-config
==> Downloading https://downloads.sf.net/project/machomebrew/Bottles/pkg-config-0.28.lion.bottle.tar.gz
######################################################################## 100.0%
==> Pouring pkg-config-0.28.lion.bottle.tar.gz
   /usr/local/Cellar/pkg-config/0.28: 10 files, 636K
==> Installing ansible dependency: sqlite
==> Downloading http://sqlite.org/2013/sqlite-autoconf-3071700.tar.gz
######################################################################## 100.0%
==> ./configure --prefix=/usr/local/Cellar/sqlite/3.7.17 --enable-dynamic-extensions



==> /usr/local/bin/python setup.py build
   /usr/local/Cellar/ansible/1.2: 186 files, 1.7M, built in 8 seconds






See: https://github.com/mxcl/homebrew/wiki/Homebrew-and-Python
Warning: Could not link python. Unlinking...
Error: The `brew link` step did not complete successfully
The formula built, but is not symlinked into /usr/local
You can try again using `brew link python'

Possible conflicting files are:
/usr/local/bin/pip-2.7
/usr/local/bin/pip
==> Summary
   /usr/local/Cellar/python/2.7.5: 5200 files, 80M, built in 3.9 minutes
==> Installing ansible
==> Downloading https://github.com/ansible/ansible/archive/v1.2.tar.gz
######################################################################## 100.0%
==> /usr/local/bin/python setup.py build
Failed to execute: /usr/local/bin/python

READ THIS: https://github.com/mxcl/homebrew/wiki/troubleshooting


$ mv /usr/local/bin/pip /usr/local/bin/pip.old
$ mv /usr/local/bin/pip-2.7 /usr/local/bin/pip-2.7.old

$ brew link python
Linking /usr/local/Cellar/python/2.7.5... 34 symlinks created

{% endhighlight %}

---
layout: post
title:  "Phabricator配置Git服务器"
date:   2016-11-29 11:17:22 +0800
categories: phabricator
---

> Phabricator集成多个版本控制工具，今天我们就来配置下Git。
> 
> 此文档参考官方说明： [https://secure.phabricator.com/book/phabricator/article/diffusion_hosting/](https://secure.phabricator.com/book/phabricator/article/diffusion_hosting/) 整理而成。


### 1. 设置用户
官方的建议是创建三个不用的系统用户帐号www-user,daemon-user,vcs-user
其中的daemon,在安装phabricator默认是用root，这个直接用root,也没问题，www-user 用的是nginx.

至于vcx-user,就是git的用户，如果服务器没有这个，创建一个:

	useradd git

> 注意（至关重要）： 
> 
> 1.要修改 /etc/shadow 找到git 把!! 修改为 NP (nopassword) ,
> 
> 如：git:NP:16918:0:99999:7:::
> 
> 2.确保 /etc/passwd 中git的指定shell 为 /bin/bash
> 
> 如：git:x:504:504::/home/git:/bin/bash


### 2. 配置Phabricator

	./bin/config set phd.user root
	
	./bin/config set diffusion.ssh-user git

	

### 3. 配置sudo

	sudoedit /etc/sudoers

找到以下这行，确保已被注释掉

	#Defaults    requiretty

在文件最后添加：

	nginx ALL=(root) SETENV: NOPASSWD:/bin/ls,/usr/bin/git
	git ALL=(root) SETENV: NOPASSWD:/bin/git,/usr/bin/git,/usr/bin/git-upload-pack,/usr/bin/git-receive-pack


### 4. 配置 SSH 服务器

> 注意openssh的版本要在6.2以上，否则，会启动不成功。
> 
> 系统是centos 6.5 的，openssh 的默认版本是5.3，不满足需求，所以我另外安装了一个最新版本的7.1
> 
> 安装的目录是 /usr/local/bin, 对应的sshd 是 /usr/local/sbin/sshd

配置端口

	./bin/config set diffusion.ssh-port 2222

复制模板文件phabricator/resources/sshd/phabricator-ssh-hook.sh到 /usr/libexec/phabricator-ssh-hook.sh
	
	cp .//resources/sshd/phabricator-ssh-hook.sh /usr/libexec/phabricator-ssh-hook.sh

确认权限

	chown root /usr/libexec/
	chown root /usr/libexec/phabricator-ssh-hook.sh
	chmod 755 /usr/libexec/phabricator-ssh-hook.sh

修改 /usr/libexec/phabricator-ssh-hook.sh,（第一次没有修改这个文件，走了不少弯路） 内容如下：

	#!/bin/sh

	# NOTE: Replace this with the username that you expect users to connect with.
	VCSUSER="git"
	
	# NOTE: Replace this with the path to your Phabricator directory.
	ROOT="/home/max/review/phabricator"
	
	if [ "$1" != "$VCSUSER" ];
	then
	  exit 1
	fi

需要修改的，都在NOTE里面了。


复制文件 phabricator/resources/sshd/sshd_config.phabricator.example 到 /etc/ssh/sshd_config.phabricator

	cp /resources/sshd/sshd_config.phabricator.example /etc/ssh/sshd_config.phabricator

修改文件内容如下

	# NOTE: You must have OpenSSHD 6.2 or newer; support for AuthorizedKeysCommand
	# was added in this version.
	
	# NOTE: Edit these to the correct values for your setup.
	
	AuthorizedKeysCommand /usr/libexec/phabricator-ssh-hook.sh
	AuthorizedKeysCommandUser git
	AllowUsers git
	
	# You may need to tweak these options, but mostly they just turn off everything
	# dangerous.
	
	Port 2222
	Protocol 2
	PermitRootLogin no
	AllowAgentForwarding no
	AllowTcpForwarding no
	PrintMotd no
	PrintLastLog no
	PasswordAuthentication no
	AuthorizedKeysFile none

启动ssh服务

	/usr/local/sbin/sshd -f /etc/ssh/sshd_config.phabricator

如需调试，可以这样

	/usr/local/sbin/sshd -d -d -d -f /etc/ssh/sshd_config.phabricator


#### 5.问题
如果出现了奇怪的问题，请检查

---
layout: post
title:  "Phabricator+Git Review环境部署"
date:   2016-07-24 11:50:22 +0800
categories: review
---

> Phabricator是一套基于Web的软件开发协作工具，包括代码审查工具Differential，资源库浏览器Diffusion，变更监测工具Herald，Bug跟踪工具Maniphest和维基工具Phriction。Phabricator可与Git、Mercurial和Subversion集成使用。其可在Apache许可证第2版下作为自由软件分发。
> 
> Phabricator最初是Facebook的一个内部工具，主要开发者为Evan Priestley。Evan Priestley离开Facebook后，在名为Phacility的新公司继续Phabricator的开发。

###1. 背景

总结开发经验，PHP的项目开发比较混乱，特别是BOSS,需求杂乱无章，开发时间紧，大部份需求，提出后都需要在两三天之内完成。 也有很多的需求开发完成后，根本就没有上线或是只用一两次就没有再用了。

开发工程师为了满足这些奇葩的需求，不得不放弃一些考虑，快速地完成开发。虽然开发的效率提高上去了，但这也会带来一些问题，如代码的安全性及代码质量问题。

出于更好的把控代码质量及安全性，必须执行代码审查。 目前BOSS项目经过将近一年的开发，工程师也换了几批，代码量实在是太大了，一个个文件地去看代码有点不大现实，必须要找到一个工具作为辅助。对工具的要求是能够友好地显示代码的变更，能够跟代码仓库进行关联。

###2. 寻求方案

对于Review工具，有两种选择：

	1. 基于目前现有的自己开发的EdocPHP进行扩展，优化代码展示界面。
	2. 选择开源Review工具。

对于1，尝试优化了下功能，发现如果单纯查看代码，Review工作量还是太大，另外就是前端的展示没有找到更好的样式，还得考虑highlight功能，要进行开发，可能还需要大量的工作。还有就是不能跟代码仓库关联，想要查看代码的变更，还要进行大量的开发。所以选择了第2种方式。

###3. 工具的选择

以前曾经尝过用reviewboard,但是环境搭建折腾了好久，感觉晦涩难用，最张放弃了。

最终找到了[ Phabricator ](https://www.phacility.com/)，花了一个下午的时间搭建好环境，尝试了下，感觉可以。

首先看看知乎上对Phabricator的问答 [这里](https://www.zhihu.com/question/19977889), Phabricator的评价还是相当不错的，非常重要一点还是用PHP开发，顿时好感倍增，所以就非它莫属了！

###4. 部署

####4.1. 环境要求

官方是这样说的：

> 
>     You will need a computer. Options include:
> 
> 	A Normal Computer: This is strongly recommended. Many installs use a VM in EC2. Phabricator installs properly and works well on a normal computer.
> 	A Shared Host: This may work, but is not recommended. Many shared hosting environments have restrictions which prevent some of Phabricator's features from working. Consider using a normal computer instead. We do not support shared hosts.
> 	A SAN Appliance, Network Router, Gaming Console, Raspberry Pi, etc.: Although you may be able to install Phabricator on specialized hardware, it is unlikely to work well and will be difficult for us to support. Strongly consider using a normal computer instead. We do not support specialized hardware.
> 	A Toaster, Car, Firearm, Thermostat, etc.: Yes, many modern devices now have embedded computing capability. We live in interesting times. However, you should not install Phabricator on these devices. Instead, install it on a normal computer. We do not support installing on noncomputing devices.
> 	To install the Phabricator server software, you will need an operating system on your normal computer which is not Windows. Note that the command line interface does work on Windows, and you can use Phabricator from any operating system with a web browser. However, the server software does not run on Windows. It does run on most other operating systems, so choose one of these instead:
> 	
> 	Linux: Most installs use Linux.
> 	Mac OS X: Mac OS X is an acceptable flavor of Linux.
> 	FreeBSD: While FreeBSD is certainly not a flavor of Linux, it is a fine operating system possessed of many desirable qualities, and Phabricator will install and run properly on FreeBSD.
> 	Solaris, etc.: Other systems which look like Linux and quack like Linux will generally work fine, although we may suffer a reduced ability to support and resolve issues on unusual operating systems.
> 	Beyond an operating system, you will need a webserver.
> 	
> 	Apache: Many installs use Apache + mod_php.
> 	nginx: Many installs use nginx + php-fpm.
> 	lighttpd: lighttpd is less popular than Apache or nginx, but it works fine.
> 	Other: Other webservers which can run PHP are also likely to work fine, although these installation instructions will not cover how to set them up.
> 	PHP Builtin Server: You can use the builtin PHP webserver for development or testing, although it should not be used in production.
> 	You will also need:
> 	
> 	MySQL: You need MySQL. We strongly recommend MySQL 5.5 or newer.
> 	PHP: You need PHP 5.2 or newer, but note that PHP 7 is not supported.

说白就是你要有一台正常配置的电脑，装*inx系统的，要是你的装是的 window 那就不好意思了，不支持。

别外需要注意的就是MySQL版本要在5.5以上，PHP 5.2以上，但还没支持到 PHP 7。

####4.2. 配置

4.2.1 Nginx配置，添加一个host,加上：
	
	
	server {
	  server_name 访问域名;
	  root        代码目录/webroot;
	
	  location / {
	    index index.php;
	    rewrite ^/(.*)$ /index.php?__path__=/$1 last;
	  }
	
	  location = /favicon.ico {
	    try_files $uri =204;
	  }
	
	  location /index.php {
	    fastcgi_pass   localhost:9000;
	    fastcgi_index   index.php;
	
	    #required if PHP was built with --enable-force-cgi-redirect
	    fastcgi_param  REDIRECT_STATUS    200;
	
	    #variables to make the $_SERVER populate in PHP
	    fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;
	    fastcgi_param  QUERY_STRING       $query_string;
	    fastcgi_param  REQUEST_METHOD     $request_method;
	    fastcgi_param  CONTENT_TYPE       $content_type;
	    fastcgi_param  CONTENT_LENGTH     $content_length;
	
	    fastcgi_param  SCRIPT_NAME        $fastcgi_script_name;
	
	    fastcgi_param  GATEWAY_INTERFACE  CGI/1.1;
	    fastcgi_param  SERVER_SOFTWARE    nginx/$nginx_version;
	
	    fastcgi_param  REMOTE_ADDR        $remote_addr;
	  }
	}


4.2.2 设置MySQL连接用户及密码,官方的说明是这样的：

	phabricator/ $ ./bin/storage upgrade --user <user> --password <password>

但我第一次试的问题，总是会报错，后来就直接去改配置文件了，配置文件在这里
	
	src/applications/config/option/PhabricatorMySQLConfigOptions.php 

配置其实不复杂，它跟一个普通的PHP项目没什么区别，但它的强大在于，很多设置可以通过命令行的方式去实现，在它的 ./bin 目录下有很多的脚本，有空可以慢慢研究它，有很多可以参考应用到我们的开发项目中。

###5.Web访问

浏览器上输入配置好的域名即可访问，首次访问会提示设置管理员帐号和密码。然后进去就可以看到这样的界面了：

![](http://i.imgur.com/BzENz15.png)


本文就先写到这里，以后再别行文记录下如果添加关联到现有的Git仓库以及如何进行Review。






	








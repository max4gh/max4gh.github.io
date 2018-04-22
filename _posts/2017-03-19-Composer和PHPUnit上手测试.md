---
layout: post
title:  "Composer和PHPUnit上手测试"
date:   2017-03-19 17:29:22 +0800
categories: review
---

> Composer是 PHP 用来管理依赖（dependency）关系的工具。你可以在自己的项目中声明所依赖的外部工具库（libraries），Composer 会帮你安装这些依赖的库文件.
> 
> Composer 相关文档和教程可以参考 [http://www.phpcomposer.com/](http://www.phpcomposer.com/)
> 本文记录下它在项目中的简单使用。

>PHPUnit 是 PHP 的单元测试工具，安装可以参考 [https://phpunit.de/getting-started.html](https://phpunit.de/getting-started.html), 还有中文文档 [https://phpunit.de/manual/current/zh_cn/installation.html](https://phpunit.de/manual/current/zh_cn/installation.html)

### 1. 目的

本文的目的是用 composer 帮助我们在项目中生成自动加载器，另外就是管理依赖库。然后再集成PHPUnit 测试功能                  

### 2. 创建项目

创建一个项目目录 my-unit-test
项目的目录结构如下：

	src(代码目录)
  		App
			Models
				Model.php(模型类)
	tests(测试目录)
		MyTest.php（测试类）
	composer.json(项目配置文件)
	phpunit.xml（phpunit测试配置文件）

composer.json 文件如下：
	
	{
    "name": "mars/my-unit-test",
    "description": "my unit test",
    "type": "project",
	"autoload":{
		"psr-4": {
			"App\\": "src/App"
		}
	},
    "require-dev": {
        "phpunit/phpunit": "^6.0"
    },
    "license": "MIT",
    "authors": [
        {
            "name": "Max",
            "email": "512796875@qq.com"
        }
    ]
	}	

这个文件关注一点：

	"autoload":{
		"psr-4": {
			"App\\": "src/App"
		}
	},

这是指定一个自动加载器，把 APP目录下的文件，设置为自动加载，只要这个目录下的文件名符合psr-4的标准（如:A.php,BxxxCxxx.php），都可以自动加载到程序中。

phpunit.xml 文件如下：
	<?xml version="1.0" encoding="UTF-8"?>
	<phpunit backupGlobals="false"
	         backupStaticAttributes="false"
	         bootstrap="vendor/autoload.php"
	         colors="true"
	         convertErrorsToExceptions="true"
	         convertNoticesToExceptions="true"
	         convertWarningsToExceptions="true"
	         processIsolation="false"
	         stopOnFailure="false">
	    <testsuites>
	        <testsuite name="MyTests">
	            <directory suffix="Test.php">./tests</directory>
	        </testsuite>
	    </testsuites>	    
	</phpunit>

这里说明下

	bootstrap="vendor/autoload.php"

这是告诉PHPUnit，开始测试前，先执行vendor/autoload.php文件，这个文件是自动加载器，在MyTest中用到Model的时候，需要到它指定的目录去找相应的类。

另三行是：

	<testsuite name="MyTests">
	     <directory suffix="Test.php">./tests</directory>
	</testsuite>

这里是定义一个测试的套件，名称是 "MyTest", 测试的文件是当前目录下的 tests 目录，后缀为"Test.php" 的文件。

创建以上文件好，执行以下命令
	
	composer intall 

执行成功之后，在根目录下会生成一个 vendor 目录，

最后来写下 Model.php 

	<?php
	
	namespace App\Models;
	
	class Model{
		public function modelMethod(){
			return true;
		}
	}

这个类并没有什么意义，只有一个测试的方法，一会我们会用到。

再看看 MyTest.php

	<?php
	namespace Test;
	
	use PHPUnit\Framework\TestCase;
	use App\Models\Model;
	
	class MyTest extends TestCase{
		protected $model;
		
		protected function setUp()
	    {
	        $this->model = new Model();
	    }
		
		public function testModelMethod()
		{
			$this->assertTrue($this->model->modelMethod());
		}
	}

这个测试类的功能是测试 Model类的方法是否能正确返回一个true的值。

### 3. 测试
都写好之后，接下来就开始运行单元测试了,直接在根目录下执行：

	phpunit

如果运行正确，可以得到以下测试通过结果：
	
	PHPUnit 6.0.9 by Sebastian Bergmann and contributors.

	.                                                                  1 / 1 (100%)
	
	Time: 242 ms, Memory: 8.00MB
	
	OK (1 test, 1 assertion)

测试下把 Model 中的方法返回 false:
	
	public function modelMethod(){
		return false;
	}

再测试下：
	
	phpunit

得到以下测试失败的结果：

	PHPUnit 6.0.9 by Sebastian Bergmann and contributors.

	F                                                                  1 / 1 (100%)
	
	Time: 263 ms, Memory: 8.00MB
	
	There was 1 failure:
	
	1) Test\MyTest::testModelMethod
	Failed asserting that false is true.
	
	E:\wamp\www\my-unit-test\tests\MyTest.php:17
	
	FAILURES!
	Tests: 1, Assertions: 1, Failures: 1.


Composer 和 PHPUnit 的简单使用就是这样，在 composer.json 的

	"require-dev": {
        "phpunit/phpunit": "^6.0"
    },

当我们的系统中没有安装到 phpunit 命令的时候，在测试时可以在根目录中可以使用这个命令测试：
	
	./vendor/bin/phpunit


示例的代码已上传到：[https://coding.net/u/max5/p/my-unit-test/git](https://coding.net/u/max5/p/my-unit-test/git)





---
layout: post
title:  "使用Laravel构建CBAdmin后台（一）"
date:   2017-2-28 10:10:25 +0800
categories: php
---
## 1. 创建 Laravel 项目

执行以下命令：

	> laravel new cbadmin
	
本地环境配置好相关的host，如：www.cbadmin.com
浏览器访问：http://www.cbadmin.com/，如看到正常页面，即项目创建成功

## 2. 数据库配置：
修改根目录下的.env文件，数据库配置部分如下：

	DB_CONNECTION=mysql
	DB_HOST=127.0.0.1
	DB_PORT=3306
	DB_DATABASE=cbadmin
	DB_USERNAME=root
	DB_PASSWORD=root

在数据库中创建与配置对应的数据库，如上面配置的 'cbadmin'
	
## 3. 添加用户认证
使用 Laravel 开箱即用的用户认证系统
	
执行以下命令：
	
	> cd cbadmin.local.fcuh.com
	> php artisan make:auth
	Authentication scaffolding generated successfully.

创建用户认证相关数据表

	> php artisan migrate
	Migration table created successfully.
	Migrated: 2014_10_12_000000_create_users_table
	Migrated: 2014_10_12_100000_create_password_resets_table

## 4. 集成 AdminLTE 后台界面

相关步骤参考 [https://github.com/jeroennoten/Laravel-AdminLTE](https://github.com/jeroennoten/Laravel-AdminLTE)

### 4.1. 使用composer添加包：

	> composer require jeroennoten/laravel-adminlte

### 4.2 在配置文件 config/app.php 添加 service provider 到 providers :
	JeroenNoten\LaravelAdminLte\ServiceProvider::class,

### 4.3 发布资源：
	> php artisan vendor:publish --provider="JeroenNoten\LaravelAdminLte\ServiceProvider" --tag=assets
	
这一步完成后，将会把 admilte 使用到的 css 及 js 等资源 复制到 \public\vendor\adminlte 目录

### 4.4 替换认证模块页面
	
	> php artisan make:adminlte


### 4.5 配置

	>　php artisan vendor:publish --provider="JeroenNoten\LaravelAdminLte\ServiceProvider" --tag=config
	
以上命令会复制一份配置文件至 \config\adminlte.php，里面有菜单及标题等等 配置，可根据需要修改

### 4.5 多语言配置

	> php artisan vendor:publish --provider="JeroenNoten\LaravelAdminLte\ServiceProvider" --tag=translations
	
以下命令会复制一份语言配置文件至 \resources\lang\vendor\adminlte 目录下，可以根据需要定制修改，了可以在目录下增加语言种类如zh（中文），默认语言可以在\config\app.php中修改，如修改为中文：
	
	'locale' => 'zh',

添加文件 \resources\lang\vendor\adminlte\zh\adminlte.php:
 
	<?php

	return [
	
	    'full_name'                   => '姓名',
	    'email'                       => '邮箱地址',
	    'password'                    => '密码',
	    'retype_password'             => '确认密码',
	    'remember_me'                 => '记住我',
	    'register'                    => '注册',
	    'register_a_new_membership'   => '注册一个新帐号',
	    'i_forgot_my_password'        => '忘记密码',
	    'i_already_have_a_membership' => '已有帐号',
	    'sign_in'                     => '登录',
	    'log_out'                     => '退出',
	    'toggle_navigation'           => '导航开关',
	    'login_message'               => '登录系统',
	    'register_message'            => '注册新帐号',
	    'password_reset_message'      => '重置密码',
	    'reset_password'              => '重置密码',
	    'send_password_reset_link'    => '发送重置密码链接',
	];

### 4.6 页面定制

	> php artisan vendor:publish --provider="JeroenNoten\LaravelAdminLte\ServiceProvider" --tag=views
	
以上命令会复制 AdminLte 的模板到 /resources/views/vendor/adminlte 目录，可以通过编辑这些文件修改页面

## 5. 创建数据表

### 5.1 创建角色表
	
	> php artisan make:migration create_roles_table --create=roles
	Created Migration: 2017_02_28_084121_create_roles_table
以上命令执行成功后，会生成一个文件 /database/migrations/2017_02_28_084121_create_roles_table.php
	
修改以上文件：
	<?php

	use Illuminate\Support\Facades\Schema;
	use Illuminate\Database\Schema\Blueprint;
	use Illuminate\Database\Migrations\Migration;
	
	class CreateRolesTable extends Migration
	{
	    /**
	     * Run the migrations.
	     *
	     * @return void
	     */
	    public function up()
	    {
	        Schema::create('roles', function (Blueprint $table) {
	            $table->increments('id');
	            $table->string('name')->comment('名称');
	            $table->smallInteger('sort')->defalut(0)->comment('排序');
	            $table->timestamps();
	        });
	    }
	
	    /**
	     * Reverse the migrations.
	     *
	     * @return void
	     */
	    public function down()
	    {
	        Schema::dropIfExists('roles');
	    }
	}

### 5.2 创建用户角色关系表
	
	> php artisan make:migration create_user_role_table --create=user_role
	Created Migration: 2017_02_28_084536_create_user_role_table

修改内容如下：

	<?php

	use Illuminate\Support\Facades\Schema;
	use Illuminate\Database\Schema\Blueprint;
	use Illuminate\Database\Migrations\Migration;
	
	class CreateUserRoleTable extends Migration
	{
	    /**
	     * Run the migrations.
	     *
	     * @return void
	     */
	    public function up()
	    {
	        Schema::create('user_role', function (Blueprint $table) {
	            $table->increments('id');
	            $table->unsignedInteger('user_id')->comment('用户ID对应users表')->index();
	            $table->unsignedInteger('role_id')->comment('角色id对应roles表');
	            $table->timestamps();
	        });
	    }
	
	    /**
	     * Reverse the migrations.
	     *
	     * @return void
	     */
	    public function down()
	    {
	        Schema::dropIfExists('user_role');
	    }
	}

### 5.3 创建菜单表
	
	> php artisan make:migration create_menus_table --create=menus
	Created Migration: 2017_02_28_085720_create_menus_table

修改文件内容如下：

	<?php

	use Illuminate\Support\Facades\Schema;
	use Illuminate\Database\Schema\Blueprint;
	use Illuminate\Database\Migrations\Migration;
	
	class CreateMenusTable extends Migration
	{
	    /**
	     * Run the migrations.
	     *
	     * @return void
	     */
	    public function up()
	    {
	        Schema::create('menus', function (Blueprint $table) {
	            $table->increments('id');
	            $table->integer('pid')->default(0)->comment('父级菜单');
	            $table->string('name')->default('')->comment('菜单名称');
	            $table->string('icon')->default('')->comment('图标');
	            $table->string('permission')->default('')->comment('菜单对应的权限');
	            $table->string('url')->default('')->comment('菜单链接地址');
	            $table->string('active')->default('')->comment('菜单高亮地址');
	            $table->string('description')->default('')->comment('描述');
	            $table->tinyInteger('sort')->default(0)->comment('排序');
	            $table->timestamps();
	        });
	    }
	
	    /**
	     * Reverse the migrations.
	     *
	     * @return void
	     */
	    public function down()
	    {
	        Schema::dropIfExists('menus');
	    }
	}

### 5.4 创建角色与菜单关系表

	> php artisan make:migration create_role_menu_table --create=role_menu
	Created Migration: 2017_02_28_090206_create_role_menu_table

修改文件内容如下： 

	<?php

	use Illuminate\Support\Facades\Schema;
	use Illuminate\Database\Schema\Blueprint;
	use Illuminate\Database\Migrations\Migration;
	
	class CreateRoleMenuTable extends Migration
	{
	    /**
	     * Run the migrations.
	     *
	     * @return void
	     */
	    public function up()
	    {
	        Schema::create('role_menu', function (Blueprint $table) {
	            $table->increments('id');
	            $table->unsignedInteger('role_id')->comment('角色id对应roles表')->index();
	            $table->unsignedInteger('menu_id')->comment('菜单ID对应menus表');
	            $table->timestamps();
	        });
	    }
	
	    /**
	     * Reverse the migrations.
	     *
	     * @return void
	     */
	    public function down()
	    {
	        Schema::dropIfExists('role_menu');
	    }
	}


### 5.5 执行创建数据库

	> php artisan migrate
	Migrated: 2017_02_28_084121_create_roles_table
	Migrated: 2017_02_28_084536_create_user_role_table
	Migrated: 2017_02_28_085720_create_menus_table
	Migrated: 2017_02_28_090206_create_role_menu_table

至此权限系统所用到的数据表已经创建完毕。
---
layout: post
title: 在 Ubuntu 服务器上搭建 Rack, Rails 生产环境
date: 2012-08-20
categories: ruby
tags: guide rack rails server ubuntu passenger apache nginx postgresql
---


## 注意事项

 * 服务器必须可以链接到外网, 下面的安装操作需要走网络下载对应的安装包, 在中国可能会存在某些网络无法访问的情况, 需自行准备梯子.
 * 再没有说明的情况下所有的命令都使用 root 用户运行, 如果不是 root 用户可以通过 `sudo su -` 切换到 root 用户进行安装.
 * 文章中的软件基于软件版本基于: `ubuntu-12.04`, `rvm-1.15.5`, `ruby-1.9.3-p194`, `postgresql-9.1` 版本可能会过期, 如果过期的话记得把命令替换成对应版本.


## 一. 基本环境安装

### 1. 确保 `en_US.UTF-8` 为默认语言. 

编辑 `/etc/default/locale` 确保其内容是 `LANG="en_US.UTF-8"` 当然可以替换成其他语言, 
不过对于服务器来说还是推荐用 "`en_US.UTF-8`" 以减少麻烦.


### 2. 更新服务器到最新的系统

    apt-get update
    apt-get dist-upgrade
    do-release-upgrade

较新系统可以减少很多不必要的兼容性问题, 而且 apt 的包也不是很新很新的, 所以基本不会出什么大问题所以 大胆的更新吧.

### 3. 设置 `/etc/bash.bashrc` 和 `/etc/profile`

<!--more-->

设置 bashrc 和 profile 主要是为了解决 no-login 和 non-interactive 模式下 shell 的 PATH 问题. 顺带的开启下自
动补全之类的. 具体的可以看 <https://wido.me/sunteya/understand-bashrc-and-profile>

编辑 `/etc/profile` 文件, 在文件头加入

    export system_profile_loaded=1

并且找到

	if [ "$PS1" ]; then
	  if [ "$BASH" ] && [ "$BASH" != "/bin/sh" ]; then
	    # The file bash.bashrc already sets the default PS1.
	    # PS1='\h:\w\$ '
	    if [ -f /etc/bash.bashrc ]; then
	      . /etc/bash.bashrc
	    fi
	  else
	    if [ "`id -u`" -eq 0 ]; then
	      PS1='# '
	    else
	      PS1='$ '
	    fi
	  fi
	fi

修改成

	if [ "$BASH" ] && [ "$BASH" != "/bin/sh" ]; then
	  [ -f /etc/bash.bashrc ] && . /etc/bash.bashrc
	fi

编辑 `/etc/bash.bashrc` 在并开头加入
    
	[ -n "${system_bashrc_running}" ] && return
	system_bashrc_running=1
	[ -z "${system_profile_loaded}" ] && source /etc/profile
	unset system_bashrc_running
	system_bashrc_runned=1

接着安装 bash 自动补全 `apt-get install -y bash-completion` , ubuntu 12.04 以后修改了 profile.d 和 bashrc 的启动顺序
所以可以不用修改 bashrc 的 bash_completion 的载入.

## 二. 安装 Ruby 环境

[RVM](http://rvm.beginrescueend.com/) 是目前最好的, 也是最常用的管理和编译 ruby 的工具, 使用 rvm, 可以减少以
后 ruby 升级带来的麻烦, 更平滑的迁移到新版本.

### 1. 在服务器上安装 RVM, 注意的是 这里使用的是 root 用户

	apt-get install -y curl git-core
	curl -L get.rvm.io | bash -s stable
{:lang='bash'}

最后记得重新登入或者重启一下服务器, 使得 rvm 生效. 测试的话可以输入

    type rvm | head -1
    rvm is a function
{:lang='bash'}

如果看到 `rvm is a function` 就算成功的安装好了. 另外由于 rvm 是 root 用户安装的, 所以其他用户如果没有 sudo 权限将
无法安装 gem, 需要将其加入 rvm 组才可以, 后面会详细说明.

### 2. 使用 RVM 安装和编译 Ruby

安装好 rvm 以后 我们可以使用 rvm 来对 ruby 进行编译, 以前曾经推荐使用 ree 版本的 ruby, 但随着 ruby 1.9.2 的发布,
ree ruby 已经在性能上没有任何优势了, 所以目前只推荐安装官方的 1.9.2+ 版本的 Ruby, 但在编译 Ruby 前首先是需要安装
该 Ruby 所需要的类库, 所以我们输入:
    
    rvm requirements

程序会列出不同的 Ruby 所需要的系统类库, 从中找出 `MRI` 也就是

	Additional Dependencies:
	# For Ruby / Ruby HEAD (MRI, Rubinius, & REE), install the following:
	  ruby: /usr/bin/apt-get install build-essential openssl libreadline6 libreadline6-dev curl git-core zlib1g zlib1g-dev libssl-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt-dev autoconf libc6-dev ncurses-dev automake libtool bison subversion pkg-config

这里列出了安装 官方 Ruby 所需要安装的所有类库, 所以直接在终端里复制粘贴, 当然请记得每次安装前都使用 `rvm requirements` 
查看下所需要的类库:

	apt-get install build-essential openssl libreadline6 libreadline6-dev curl git-core zlib1g zlib1g-dev libssl-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt-dev autoconf libc6-dev ncurses-dev automake libtool bison subversion pkg-config

等待安装完成后就可以编译 Ruby 了.

安装 Ruby 由于使用了 RVM 所以只需要在终端里 输入:

	rvm install 1.9.3

就会自动的编译和安装 Ruby, 如果需要安装其他版本的 ruby 也可以通过命令

	rvm list known

查看目前可以使用的 Ruby 版本, 其中 `[]` 内的部分可以省略.

等待 Ruby 1.9.2 安装好以后, 我们可以通过 `rvm list` 命令来查看所有通过 rvm 安装的 ruby 版本.

### 3. 设置 Ruby 环境

首先, 我们通过 rvm 安装完以后, 新版本 rvm 会自动的把当前安装的 ruby 设置成默认 ruby, 如果没有可以用下面命令设置

    rvm use 1.9.3 --default

来将我们安装的 Ruby 设置为系统默认的 Ruby 环境. 接着我们就可以通过 `ruby -v` 来验证刚刚的设置是否成功.

另外, 在服务器上由于不是开发环境, 所以 rdoc 和 ri 基本是毫无作用的, 所以我们最好默认就不让 rake 去安装 rdoc 和 ri, 
所以我们还需要编辑 `/usr/local/rvm/rubies/ruby-1.9.3-p194/etc/gemrc`, (如果没有就自己建出来, 记得先建目录) 其
内容为:

    gem: -V --no-ri --no-rdoc

这样设置完以后, 每次使用 gem 命令就不会下载并安装 ri 和 rdoc 了, 另外该命令只对 1.9.2+ 的版本才有效, 如果是 1.8.7 的话
需要编辑的文件是 `/etc/gemrc`

至此 Ruby 就算基本安装完成了

## 三. 安装数据库及其驱动

由于我基本只用 Postgresql, 所以这里只安装 Postgresql, 如果要安装 Mysql或者其他数据库, 请自行 google 安装文档.

### 1. 安装 Postgresql Server (可选)

安装 postgresql server, 这里如果有多台机器作为服务器, 显然只需要一台机器安装数据库就可以了.

    apt-get install postgresql

安装完成以后, 编辑 `/etc/postgresql/9.1/main/postgresql.conf` 找到 
    
    #listen_addresses = 'localhost'

将其修改成
    
    listen_addresses = '*'

这样就可以通过远程来访问该数据库了, 当然还需要开启对应的权限才可以. 接着我们编辑 `/etc/postgresql/9.1/main/pg_hba.conf` 
在文件的最后可以看到:


    # Database administrative login by UNIX sockets
    local   all         postgres                          peer

    # TYPE  DATABASE    USER        CIDR-ADDRESS          METHOD

    # "local" is for Unix domain socket connections only
    local   all         all                               peer
    # IPv4 local connections:
    host    all         all         127.0.0.1/32          md5
    # IPv6 local connections:
    host    all         all         ::1/128               md5


这样的内容, 如果是开发或者测试的服务器的话, 我会修改成:

    # Database administrative login by UNIX sockets
    # local   all         postgres                          peer

    # TYPE  DATABASE    USER        CIDR-ADDRESS          METHOD

    # "local" is for Unix domain socket connections only
    local   all         all                               md5
    # IPv4 local connections:
    host    all         all         0.0.0.0/0             md5
    # IPv6 local connections:
    host    all         all         ::1/0                 md5
    
这样任何人在任何的地方都可以通过 ip 来访问数据库, 方便测试和调试, **但如果是生产环境的话, 请务必明白每一行代码的效果后在修改.**

由于默认 postgres 用户密码是随机的, 如果需要使用 postgres 通过远程访问的话, 按照我们上述修改的话, 由于不知道密码还无法登入.
所以我们还需要在命令行输入:

    sudo su - postgres -c psql postgres
    ALTER USER postgres WITH PASSWORD 'postgres';
    \q

修改 postgres 用户的密码, 我这里修改成的密码是 "postgres", 实在是不安全, 所以请大家务必要修改成其他的密码.

最后输入 `/etc/init.d/postgresql restart` 重启 postgresql 服务器使配置文件生效.


### 2. 安装 Postgresql 客户端和 Ruby 驱动

    apt-get install -y postgresql-client libpq-dev
    gem install pg

这个很简单直接 apt 和 gem 就可以了.


## 四. 安装 Apache Web 服务器 (选择一)

一般来说有 2种选择 Apache + Passenger 或者 Nginx + Passenger, 如果不是追求极致性能. 我是比较推荐 Apache + Passenger 的. 因为升级维护起来会比 Nginx 简单很多. 当然如果需要 nginx 的配置步骤请往下看 (选择二).

### 1. 安装 apache 服务器

在命令行输入

    apt-get install -y apache2-mpm-prefork apache2-prefork-dev

安装完成以后, 用浏览器访问服务器的80端口确保可以看到 "It works!". 再运行

    a2dissite default
    /etc/init.d/apache2 restart
    rm /var/www/index.html

关闭刚刚那个无用的默认的网站.

### 2. 设置 apache 用户

apache 默认使用的是 www-data 用户, 而 Rails 程序也是使用该用户发布, 所以要修改 www-data 的相关配置信息, 允许这个
用户安装 Gem, 也就是加入 rvm 组
    
    usermod --shell=/bin/bash -a -G rvm www-data
    chown -R www-data:www-data /var/www

完成以后 Apache 基本也就安装好了

### 3. 安装 Passenger

在终端中输入

	gem install passenger

会自动下载并安装 passenger, 安装完 passenger 后, 输入

	apt-get install libcurl4-openssl-dev

安装编译 passenger 的必须的 curl 的 ssl dev包, 接着输入

	passenger-install-apache2-module

等编译完成以后, 会看到这样的提示

	Please edit your Apache configuration file, and add these lines:
	
	   LoadModule passenger_module /usr/local/rvm/gems/ruby-1.9.2-p290/gems/passenger-3.0.15/ext/apache2/mod_passenger.so
	   PassengerRoot /usr/local/rvm/gems/ruby-1.9.2-p290/gems/passenger-3.0.15
	   PassengerRuby /usr/local/rvm/wrappers/ruby-1.9.2-p290/ruby

于是我们就按照他的提示编辑 `/etc/apache2/mods-available/passenger.load` 输入

	LoadModule passenger_module /usr/local/rvm/gems/ruby-1.9.3-p194/gems/passenger-3.0.15/ext/apache2/mod_passenger.so

再编辑 `/etc/apache2/mods-available/passenger.conf` 输入

	PassengerRoot /usr/local/rvm/gems/ruby-1.9.3-p194/gems/passenger-3.0.15
	PassengerRuby /usr/local/rvm/wrappers/ruby-1.9.3-p194/ruby

接下来让 apache 启用该模块, 输入

    a2enmod passenger
    /etc/init.d/apache2 restart

至于 Passenger 的性能调优, 请参考 [passenger 的官方文档](http://www.modrails.com/documentation.html)

## 五. 安装 Nginx Web 服务器 (选择二)

至于遇到一个纯 API 的项目, 每台服务器设计必须能处理 800请求/秒. 当然真真性能瓶颈是在 CPU 上. 不过同样的配置下 Apache 能到 500请求/秒, 而 Nginx 能上 2000请求/秒. 当然这是最极端情况, 正常情况估计也有一倍以上的差距.

所以还是有必要整理安装步骤, 只是 Nginx 升级和维护比较麻烦而已, 每次都需要重新编译.

### 1. 安装 passenger

只续简单输入

	gem install passenger

等待自动下载并安装完成. 输入如下命令

	cd $(ruby -e 'gem "passenger"; print Gem.loaded_specs["passenger"].full_gem_path')
	rake nginx:clean nginx

用来编译 Passenger 的 Nginx 的相关文件.


### 2. 下载和编译 nginx 

首先需要下载一些 Nginx 和 Passenger 的编译依赖包

	apt-get install -y libpcre++-dev libcurl4-openssl-dev

然后输入如下命令

	cd /usr/local/src/
	wget http://nginx.org/download/nginx-1.3.4.tar.gz
	tar xvf nginx-1.3.4.tar.gz
	cd nginx-1.3.4

解压 nginx 并进入 nginx 目录后, 输入

	./configure --with-http_ssl_module --with-http_gzip_static_module \
	--add-module="$(ruby -e 'gem "passenger"; print Gem.loaded_specs["passenger"].full_gem_path')/ext/nginx" \
	--pid-path=/var/run/nginx.pid

我一般会把 Nginx 代替系统全局的 Nginx 所以会把他的 pid 文件放在系统的 `/var/run` 下. 之后只需要

	make && make install

接着还需要安装对应的 manpage, 输入

	mkdir -p /usr/local/share/man/man8/
	install objs/nginx.8 /usr/local/share/man/man8/

接着需要将其连接到系统的配置和日志目录下, 输入

	ln -s /usr/local/nginx/sbin/nginx /usr/local/sbin/
	ln -s /usr/local/nginx/conf /etc/nginx
	ln -s /usr/local/nginx/logs /var/log/nginx

### 3. 配置 nginx

一般来说默认编译出的配置并不适合生产环境的配置, 比较倾向于 ubuntu 自带的区分 sites-enabled 和 conf.d 的样子, 所以这里需要. 输入

	mkdir -p /etc/nginx/conf.d
	mkdir -p /etc/nginx/sites-available
	mkdir -p /etc/nginx/sites-enabled
	usermod --shell=/bin/bash -a -G rvm www-data
	mkdir -p /var/www
	chown -R www-data:www-data /var/www

稍微解释下 conf.d 放扩展的配置文件, sites-available 放服务器配置, 启用服务后需要将配置连接到 sites-enabled 下, 接着需要修改 `/etc/nginx.conf` 成如下内容

	user www-data;
	worker_processes 4;
	pid /var/run/nginx.pid;

	events {
		worker_connections 768;
		# multi_accept on;
	}
	
	http {
		# Basic Settings
		sendfile on;
		tcp_nopush on;
		tcp_nodelay on;
		keepalive_timeout 65;
		types_hash_max_size 2048;
		# server_tokens off;
		
		# server_names_hash_bucket_size 64;
		# server_name_in_redirect off;
		
		include mime.types;
		default_type application/octet-stream;
		
		# Logging Settings
		access_log logs/access.log;
		error_log logs/error.log;
		
		# Gzip Settings
		gzip on;
		gzip_disable "msie6";
		
		# gzip_vary on;
		# gzip_proxied any;
		# gzip_comp_level 6;
		# gzip_buffers 16 8k;
		# gzip_http_version 1.1;
		# gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
		
		
		# Virtual Host Configs
		include conf.d/*.conf;
		include sites-enabled/*;
	}

### 4. 配置 passenger

虽然编译了 passenger 的模块, nginx 还需要配置 passenger 的配置. 配置前首先需要获取 passenger 的安装目录, 输入

	root@server # ruby -e 'gem "passenger"; puts Gem.loaded_specs["passenger"].full_gem_path'
	/usr/local/rvm/gems/ruby-1.9.3-p194/gems/passenger-3.0.15

这里的 `/usr/local/rvm/gems/ruby-1.9.3-p194/gems/passenger-3.0.15` 就是 passenger 的安装目录, 接着编辑 `/etc/nginx/conf.d/passenger.conf` 文件, 输入如下内容

	passenger_root /usr/local/rvm/gems/ruby-1.9.3-p194/gems/passenger-3.0.15;
	passenger_ruby /usr/local/rvm/wrappers/ruby-1.9.3-p194/ruby;

保存退出后, 再在 `/etc/nginx/sites-available` 下新建一个叫 `default` 的文件, 内容如下:

	server {
		listen   80; ## listen for ipv4; this line is default and implied
		# listen   [::]:80 default ipv6only=on; ## listen for ipv6
		
		server_name _;
		
		root /var/www/default/htdocs;
		autoindex on;
		index index.html index.htm;
		
		access_log logs/default-access.log;
		error_log logs/default-error.log;
	}

完成后输入

	ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/default

启用配置, 接着输入

	mkdir -p /var/www/default/htdocs
	echo world > /var/www/default/htdocs/hello.txt
	chown -R www-data:www-data /var/www

新建测试文件后, 可以通过命令 `nginx` 启动 nginx, 然后命令行输入 `curl http://localhost/hello.txt` 就可以看到其返回的结果是 "world" 了, 然后通过命令 `nginx -s stop` 停止 nginx 服务器.

### 5. 配置 Nginx 随系统自动启动

之前命令虽然已经可以启动 nginx 了, 但不会随系统一起启动. 为了让他随着系统启动我们还需要编辑 `/etc/init.d/nginx` 文件, 输入

	#!/bin/sh
	
	### BEGIN INIT INFO
	# Provides:          nginx
	# Required-Start:    $local_fs $remote_fs $network $syslog
	# Required-Stop:     $local_fs $remote_fs $network $syslog
	# Default-Start:     2 3 4 5
	# Default-Stop:      0 1 6
	# Short-Description: starts the nginx web server
	# Description:       starts nginx using start-stop-daemon
	### END INIT INFO

	PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
	DAEMON=/usr/local/sbin/nginx
	NAME=nginx
	DESC=nginx
	
	# Include nginx defaults if available
	if [ -f /etc/default/nginx ]; then
		. /etc/default/nginx
	fi
	
	test -x $DAEMON || exit 0
	
	set -e
	
	. /lib/lsb/init-functions
	
	test_nginx_config() {
		if $DAEMON -t $DAEMON_OPTS >/dev/null 2>&1; then
			return 0
		else
			$DAEMON -t $DAEMON_OPTS
			return $?
		fi
	}
	
	case "$1" in
		start)
			echo -n "Starting $DESC: "
			test_nginx_config
			# Check if the ULIMIT is set in /etc/default/nginx
			if [ -n "$ULIMIT" ]; then
				# Set the ulimits
				ulimit $ULIMIT
			fi
			start-stop-daemon --start --quiet --pidfile /var/run/$NAME.pid \
			    --exec $DAEMON -- $DAEMON_OPTS || true
			echo "$NAME."
			;;
			
		stop)
			echo -n "Stopping $DESC: "
			start-stop-daemon --stop --quiet --pidfile /var/run/$NAME.pid \
			    --exec $DAEMON || true
			echo "$NAME."
			;;
			
		restart|force-reload)
			echo -n "Restarting $DESC: "
			start-stop-daemon --stop --quiet --pidfile \
			    /var/run/$NAME.pid --exec $DAEMON || true
			sleep 1
			test_nginx_config
			start-stop-daemon --start --quiet --pidfile \
			    /var/run/$NAME.pid --exec $DAEMON -- $DAEMON_OPTS || true
			echo "$NAME."
			;;
			
		reload)
			echo -n "Reloading $DESC configuration: "
			test_nginx_config
			start-stop-daemon --stop --signal HUP --quiet --pidfile /var/run/$NAME.pid \
			    --exec $DAEMON || true
			echo "$NAME."
			;;

		configtest|testconfig)
			echo -n "Testing $DESC configuration: "
			if test_nginx_config; then
				echo "$NAME."
			else
				exit $?
			fi
			;;

		status)
			status_of_proc -p /var/run/$NAME.pid "$DAEMON" nginx && exit 0 || exit $?
			;;
		*)
			echo "Usage: $NAME {start|stop|restart|reload|force-reload|status|configtest}" >&2
			exit 1
			;;
	esac
	
	exit 0

完成后还需要附上执行权限

	chmod +x /etc/init.d/nginx

都完成后就可以通过命令 `service nginx start|stop|status` 来启动,停止和查看 nginx 了. 之后还需要通过命令

	update-rc.d nginx defaults

设置 nginx 可以随着系统自动启动.

### 6. Nginx 后记

之前的只是安装 nginx 步骤, 之后如果要升级 nginx 或者 passenger 的话, 还需要重新编译 nginx 的. 还有比如需要集成 php 也比 apache 麻烦多了, 所以一般情况下并不是很推荐 nginx 虽然他确实很快.

还需要说明的一点是: 我这里的 Nginx 主旨是用来替换系统自带的 nginx, 因为自带的没法集成 passenger, 所以会把配置日志之类的都会放到系统的默认的地方.

## 六. 最后

至此一台基本的 Rack 服务器也基本搭建完成了, 后续的工作就是发布和配置项目了.

一般只需要按照 [passenger 的官方文档](http://www.modrails.com/documentation.html) 配置项目, 接着重启 web 服务器就好了.

<!-- , 具体的可以参考: <https://wido.me/sunteya/deploy-a-rails-project> -->


## 相关连接 {#further-reading}

 * <https://wido.me/sunteya/understand-bashrc-and-profile>
 * <https://wido.me/sunteya/deploy-a-rails-project>
 * <https://rvm.beginrescueend.com/rvm/install/>


## 更新历史 {#changes}

 *  <span>2012-08-20</span>增加 nginx 的配置步骤
 *  <span>2012-05-02</span>根据 ubuntu-12.04, 更新安装步骤.
 *  <span>2012-01-30</span>更新软件版本, `rvm-1.10.2` 解决了安装权限问题.
 *  <span>2011-12-23</span>更新软件版本到 `ubuntu-11.10`, `rvm-1.10.0`, `postgresql-9.1.1`. 提示 `rvm-1.10.0` 会存在权限问题.


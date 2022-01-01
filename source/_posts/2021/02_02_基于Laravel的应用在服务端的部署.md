---
title:  基于Laravel的应用在服务端的部署
date:   2021-02-02 16:21:14 +0800
tags:
  - sql
  - coding
categories:
  - 折腾
---

>一个已经去世的应用重获新生，需要在服务端重新部署，因此就有了这篇文章。

### 服务器环境

- 系统：Ubuntu 20.04
- CPU：4核 Intel(R) Xeon(R) Gold 6278C CPU @ 2.60GHz
- 内存：8G

### 前置工作

Ubuntu 20.04只有php7.2，但我至少需要php7.4，因此先更新php源：

```sh
sudo add-apt-repository ppa:ondrej/php
```

然后安装HTTP服务器、Mysql、PHP、以及Laravel的一些依赖。
```sh
sudo apt install apache2 \
    mysql-server \
    php7.4 \
    libapache2-mod-php7.4 \
    php7.4-mysql \
    php7.4-opcache php7.4-mbstring \
    php-ssh2 php-tokenizer \
    php7.4-xml php7.4-json \
    php7.4-bcmath php7.4-zip \
    php7.4-curl php7.4-gd
```

将项目文件解压到`/var/www/`目录下
```sh
unzip finelyteam.zip
sudo mv finelyteam /var/www/finelyteam
```

### 配置Apache
#### 配置VirtualHost

配置Apache网站文件`/etc/apache2/sites-available/finelyteam.conf`：
```apache
<VirtualHost *:80>
	ServerAdmin root@localhost
	ServerName dev.finely-team.com
	DocumentRoot /var/www/finelyteam/public

	ErrorLog ${APACHE_LOG_DIR}/error.finely-team.log
	CustomLog ${APACHE_LOG_DIR}/access.finely-team.log combined
</VirtualHost>
```

#### 启用VirtualHost

```sh
sudo a2ensite finelyteam.conf
```

#### 启用Apache模块

```sh
sudo a2enmod php7.4 ssl rewrite
```

#### 编辑apache全局设置

编辑文件`/etc/apache2/apache2.conf`，注意`AllowOverride`选项变更为`all`

```apache
<Directory /var/www/>
	Options Indexes FollowSymLinks
	AllowOverride all
	Require all granted
</Directory>
```

#### 重启 Apache 服务

```
sudo service apache2 restart
```

### 配置MySQL

#### 生成管理员用户

为了让PHP访问数据库，我们需要生成一个MySQL用户。
MySQL初始只允许root免密登录。
这是不安全的，而且Laravel应用无法使用这个账户。

在MySQL cli里执行以下指令以生成用户并赋予所有权限。

```sql
CREATE USER laravel_admin@localhost IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON * . * TO laravel_admin@localhost;
FLUSH PRIVILEGES;
```

#### 添加数据库

```sql
CREATE DATABASE finelyteam;
```

### 配置PostgreSQL

#### 安装PHP驱动

如果你安装的不是mysql而是pgsql，则需要安装php驱动：
```sh
sudo apt install php7.4-pgsql
```

进入cli时不能使用root用户，只能使用特定用户：`postgres`

```
su postgres
psql
```

#### 创建管理员用户

```
create user laravel_admin superuser password '123456';
```

#### 创建数据库

```sql
CREATE DATABASE laravel;
```

### 配置PHP

#### 安装Composer

Composer是PHP的包管理工具。

```sh
sudo apt-get remove composer # 不要使用 Ubuntu 库中的Composer
cd # 回家
curl -sS https://getcomposer.org/installer | php
mv ./composer.phar /usr/local/bin/composer
# 如果提示 'Failed to decode zlib stream' 则执行
# sudo apt install zlib
```

#### Composer换源与安装

```sh
composer config -g repo.packagist composer https://mirrors.aliyun.com/composer/ # 切换国内 Composer 镜像
cd /var/www/finelyteam
composer update # 自动安装
```

### 配置Laravel

#### DOT ENV

现在来到Laravel项目的文件夹下，先生成`.env`文件。

```sh
cd /var/www/finelyteam
mv .env.example .env
php artisan key:generate # 生成 APP KEY
```

如果之前使用的是pgsql，则需要修改数据库连接，尤其是端口：

```env
DB_CONNECTION=pgsql
DB_HOST=127.0.0.1
DB_PORT=5432
DB_DATABASE=laravel
DB_USERNAME=laravel_admin
```

#### 目录权限

为以下目录分配权限以保证访问正常。

```sh
sudo chmod -R 777 public storage bootstrap/cache
```

#### 数据库迁移

```sh
php artisan migrate --seed
```

All done.

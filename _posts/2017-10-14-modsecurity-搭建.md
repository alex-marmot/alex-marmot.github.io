---
layout: post
title: "Modsecurity 搭建"
date: 2017-10-14
categories: Note
---
# Modsecurity 项目简介

ModSecurity是一个入侵探测与阻止的引擎.它主要是用于Web应用程序所以也可以叫做Web应用程序防火墙.它可以作为Apache Web服务器的一个模块或单独的应用程序来运行.ModSecurity的目的是为增强Web应用程序的安全性和保护Web应用程序避免遭受来自已知与未 知的攻击.

## 概述

在阅读项目文档[https://github.com/SpiderLabs/ModSecurity](https://github.com/SpiderLabs/ModSecurity)时，我发现有 V2,V3 两个版本存在。差别如下所示:

>All Apache dependences have been removed #移除了 Apache 相关依赖
>Higher performance # 更高的性能
>New features # 新特性
>New architecture # 新架构

本着尝新的原则，我直接使用 V3 来进行部署。实验环境 Ubuntu 16.04 LTS

## 安装过程

### 安装依赖

`apt-get install apache2-dev autoconf automake build-essential bzip2 checkinstall devscripts flex g++ gcc git graphicsmagick-imagemagick-compat graphicsmagick-libmagick-dev-compat libaio-dev libaio1 libass-dev libatomic-ops-dev libavcodec-dev libavdevice-dev libavfilter-dev libavformat-dev libavutil-dev libbz2-dev libcdio-cdda1 libcdio-paranoia1 libcdio13 libcurl4-openssl-dev libfaac-dev libfreetype6-dev libgd-dev libgeoip-dev libgeoip1 libgif-dev libgpac-dev libgsm1-dev libjack-jackd2-dev libjpeg-dev libjpeg-progs libjpeg8-dev liblmdb-dev libmp3lame-dev libncurses5-dev libopencore-amrnb-dev libopencore-amrwb-dev libpam0g-dev libpcre3 libpcre3-dev libperl-dev libpng12-dev libpng12-0 libpng12-dev libreadline-dev librtmp-dev libsdl1.2-dev libssl-dev libssl1.0.0 libswscale-dev libtheora-dev libtiff5-dev libtool libva-dev libvdpau-dev libvorbis-dev libxml2-dev libxslt-dev libxslt1-dev libxslt1.1 libxvidcore-dev libxvidcore4 libyajl-dev make openssl perl pkg-config tar texi2html unzip zip zlib1g-dev`

### 下载 Modsecurity

```bash
cd /opt/ 
git clone https://github.com/SpiderLabs/ModSecurity 
cd ModSecurity 
git checkout -b v3/master origin/v3/master 
sh build.sh 
git submodule init 
git submodule update 
./configure 
make
make install
```

### 下载 Modsecurity-nginx connector

```bash
cd /opt/
git clone https://github.com/SpiderLabs/ModSecurity-nginx.git
```

### 下载 Nginx

[http://nginx.org/en/download.html](http://nginx.org/en/download.html)

``` bash
cd /opt
wget http://nginx.org/download/nginx-1.12.0.tar.gz 
tar -zxf nginx-1.12.0.tar.gz 
cd nginx-1.12.0
```

### 安装配置 Nginx

```bash
./configure --user=www-data --group=www-data --with-pcre-jit --with-debug --with-http_ssl_module --with-http_realip_module --add-module=/opt/ModSecurity-nginx 
make 
make install
```

#### 复制规则

`cp /opt/ModSecurity/modsecurity.conf-recommended /usr/local/nginx/conf/modsecurity.conf`

#### 创建 Nginx 相关目录

```bash
ln -s /usr/local/nginx/sbin/nginx /bin/nginx
mkdir /usr/local/nginx/conf/sites-available 
mkdir /usr/local/nginx/conf/sites-enabled 
mkdir /usr/local/nginx/conf/ssl 
mkdir /etc/nginx
ln -s /usr/local/nginx/conf/ssl /etc/nginx/ssl
```

#### 修改 Nginx 配置文件

`vi /usr/local/nginx/conf/nginx.conf`

添加

`include /usr/local/nginx/conf/sites-enabled/*;`

修改 user `user www-data;`

#### 添加 Nginx 的控制脚本

```bash
wget https://raw.github.com/JasonGiedymin/nginx-init-ubuntu/master/nginx -O /etc/init.d/nginx 
chmod +x /etc/init.d/nginx
update-rc.d nginx defaults
```

#### 安装 OWASP ModSecuirty Core Rule Set

```bash
cd /opt/ 
git clone https://github.com/SpiderLabs/owasp-modsecurity-crs.git 
cd owasp-modsecurity-crs/ 
cp -R rules/ /usr/local/nginx/conf/ 
cp /opt/owasp-modsecurity-crs/crs-setup.conf.example /usr/local/nginx/conf/crs-setup.conf
```

#### 修改 modsecurity.conf

`vi /usr/local/nginx/conf/modsecurity.conf`

添加

```BASH
Include crs-setup.conf 
Include rules/*.conf 
```

#### 修改 Nginx 配置文件

`vi /usr/local/nginx/conf/nginx.conf`

配置样例:

>server {    
>.....   
>modsecurity on;   
>location / {    
>modsecurity_rules_file /usr/local/nginx/conf/modsecurity.conf;    
>.....   
>}   
>}

#### 启动 Nginx 并测试

`service nginx start`

#### 引用资料
[https://www.howtoforge.com/tutorial/nginx-with-libmodsecurity-and-owasp-modsecurity-core-rule-set-on-ubuntu-1604/](https://www.howtoforge.com/tutorial/nginx-with-libmodsecurity-and-owasp-modsecurity-core-rule-set-on-ubuntu-1604/)

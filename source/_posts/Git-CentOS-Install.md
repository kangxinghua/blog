---
title: Git CentOS 源码安装
date: 2016-11-02 13:42:14
tags: 笔记
---
## 安装依赖包
``` shell
yum install gcc
yum install curl
yum install curl-devel
yum install zlib-devel
yum install openssl-devel
yum install perl
yum install cpio
yum install expat-devel
yum install gettext-devel
yum install libcurl-devel
yum install autoconf
yum install perl-ExtUtils-Embed
```

## 下载最新的git包 
在 https://github.com/git/git/releases 找到你需要的源码包,我现在最新的是2.10.2版本
``` shell
wget https://github.com/git/git/archive/v2.10.2.tar.gz
tar xzvf v2.10.2.tar.gz
cd git-2.10.2
autoconf
./configure
make
sudo make install
```

``` shell
git --version
```

## 收工



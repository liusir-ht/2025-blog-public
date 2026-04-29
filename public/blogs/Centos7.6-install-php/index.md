## 背景
  业务上有一个新服务，环境与其它环境差异比较大，需要使用php去执行，默认情况下的php版本太低，不符合业务要求，来安装高版本的php

## 环境介绍
|组件名称|组件版本|组件介绍|
|--|--|--|
|Centos| 7.6 | 开源的Linux操作系统|
|PHP| 7.3.33| 开源的PHP语言|


## 核心步骤

1. 安装EPEL和Remi仓库
首先需要安装EPEL（Extra Packages for Enterprise Linux）和Remi仓库，Remi是提供最新PHP版本的第三方仓库：
```
# 安装EPEL仓库
yum install epel-release -y

# 安装Remi仓库（CentOS 7版本）
yum install https://rpms.remirepo.net/enterprise/remi-release-7.rpm -y
```

2.  安装yum-utils并启用PHP 7.3仓库
```
# 安装yum-utils，提供yum-config-manager命令
yum install -y yum-utils

# 禁用默认的PHP 5.4仓库
yum-config-manager --disable remi-php54

# 启用PHP 7.3仓库
yum-config-manager --enable remi-php73
```

3. 安装PHP 7.3.33,启用仓库后，直接安装PHP即可获得7.3.33版本：

```
yum install php -y
```
4. 安装常用PHP扩展（可选但推荐）
```
yum install php-cli php-common php-gd php-mbstring php-mysqlnd php-xml php-zip php-opcache php-fpm -y
```
常用扩展说明：

`php-cli`：命令行支持

`php-mysqlnd`：MySQL原生驱动

`php-fpm`：FastCGI进程管理器

`php-opcache`：性能加速

`php-gd`：图像处理

`php-mbstring`：多字节字符串处理


## 结果验证
```
[root@xxxx ~]# php -v
PHP 7.3.33 (cli) (built: Jun  5 2024 05:42:20) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.3.33, Copyright (c) 1998-2018 Zend Technologies
```
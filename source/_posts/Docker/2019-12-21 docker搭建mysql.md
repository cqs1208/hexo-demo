---

layout: post
title: docker搭建MySQL
tags:
- docker
categories: docker
description: docker
---

docker搭建MySQL

<!-- more --> 

## 1 下载镜像

```shell
docker pull mysql:5.7
```

## 2 挂载和配置文件

```shell
# 新建MySQL数据目录
mkdir -p /docker/mysql/data
chmod 777 -R /docker/mysql/data
# 新建MySQL日志目录
mkdir -p /docker/mysql/log
chmod 777 -R /docker/mysql/log
# 新建MySQL配置文件目录
mkdir -p /docker/mysql/conf
```

新建并编辑配置文件

```shell
# 新建并编辑配置文件
vi /docker/mysql/conf/mysqld.cnf

# 编辑内容
[client]
#该目录下的内容常用来进行localhost登陆，一般不需要修改
# 端口号
port = 3306 

[mysql]
#包含一些客户端mysql命令行的配置
# 默认不自动补全 auto-rehash自动补全
auto-rehash 

[mysqld]
#mysql优化的配置目录，除硬件和环境配置外，全部优化在此配置，一般服务器安装只有此配置目录
#默认启动用户，一般不需要修改，可能出现启动不成功
user = mysql 
#端口号
port = 3306 
#设置数据库服务器默认编码 utf-8
character-set-server = utf8 
#数据库目录，数据库目录切换时需要用到
datadir =/var/lib/mysql 
#mysql进程文件，可指定自己的进程文件
pid-file =/var/log/mysql/mysqld.pid 
#外部锁定(非多服务器可不设置该选项，默认skip-external-locking)
external-locking = FALSE 
#跳过外部锁定 （避免多进程环境下的external locking 锁机制）
skip-external-locking 
#跳过主机名解析，直接IP访问，可提升访问速度
skip-name-resolve 
#错误日志文件
log-error =/var/log/mysql/mysqld.log 
#默认为1,表示启用警告信息记录日志,不需要置0即可,大于1时表示将错误或者失败连接记录日志
log-warnings 
#全局只读变量,文件描述符限制 注：上限其实为OS文件描述符上限，小于OS上限时生效 可用lsof查看限制并修改相应配置
open_files_limit = 10240 
#mysql支持的基本语法及校验规则
sql_mode = STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION
```

## 3 启动

```shell
docker run -p 3306:3306 \
--name mysql \
-v /docker/mysql/conf/mysqld.cnf:/etc/mysql/mysql.conf.d/mysqld.cnf \
-v /docker/mysql/log:/var/log/mysql \
-v /docker/mysql/data:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=chen1208 \
-id mysql:5.7

--name   # 设置容器名
-v  # 挂载文件
-e MYSQL_ROOT_PASSWORD=chen1208   # 设置密码
```

## 4 dockerFile配置

 。。。
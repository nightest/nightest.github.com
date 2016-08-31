---
layout: post
title: CentOS6迁移NOdebb与Mon个DB的升级
category: 技术
tags: [tech, linux, centos, nodebb, mongodb]
---

* Content
{:toc}


## 1. 常见问题1：mongodb 启动不了

** mongodb 启动不了**
[报错](http://www.dataguru.cn/thread-107361-1-1.html):**child process failed, exited with error number 100**

检查启动日志发现：

**************
Unclean shutdown detected.
Please visit http://dochub.mongodb.org/core/repair for recovery instructions.
*************
Sat Apr 20 09:40:31.286 [initandlisten] exception in initAndListen: 12596 old lock file, terminating

于是：
```
#删除db文件夹下的mongod.lock文件
rm /usr/local/mongodb/db/mongod.lock
#然后以repair的模式启动：
/usr/local/mongodb/bin/mongod -f /usr/local/mongodb/bin/mongodb.conf --repair
#然后接着在启动一次
/usr/local/mongodb/bin/mongod -f /usr/local/mongodb/bin/mongodb.conf
#现在就可以查看到mongod的进程存在了
ps -ef |grep mongo
```
## 2. 常见问题2：如何正常关闭MongoDB

#### **方法1：**

先通过shell连上服务器：
mongo
use admin
db.shutdownServer()

#### **方法2：**

ps -ef |grep mongo
直接kill -15 &lt;pid\>;,注意kill -9 可能会导致数据文件损坏

## 3. MongoDB 升级与数据库迁移

[参考:http://www.tuicool.com/articles/f2aaquq](http://www.tuicool.com/articles/f2aaquq)

 > 注意：MongoDB不能从2.4一步升级到3.2；必须先从2.4升级到2.6，再升级到3.0，最后升级到3.6。
 > 本文只进行了到2.6的升级。
 > **另外：新版本3.2的`auth=true`选项已经失效，必须改为`authorization: enabled`

**3.1 停止老版本MongoDB**

```
ps -ef |grep mongod
kill -15 pid
```

**3.2 下载并解压新版MongoDB**

下载地址：[http://downloads.mongodb.org/linux/mongodb-linux-x86_64-2.6.0.tgz](http://downloads.mongodb.org/linux/mongodb-linux-x86_64-2.6.0.tgz)

```
wget http://downloads.mongodb.org/linux/mongodb-linux-x86_64-2.6.0.tgz
tar xvf mongodb-linux-x86_64-2.6.0.tgz
mv mongodb-linux-x86_64-2.6.0 mongodb2.6

```

**3.3 创建logs与database文件夹**

```
cd mongodb2.6
mkdir db
mkdir logs
```

新建配置文件`vim ./bin/mongodb.conf`:
```
bind_ip=127.0.0.1
dbpath=/usr/local/mongodb2.6/db
logpath=/usr/local/mongodb2.6/logs/mongodb.log
port=27017
fork=true
nohttpinterface=true
auth=true
```

**3.4 数据库备份与重新导入**

备份：

```
./bin/mongod --dbpath ../mongodb/db/ &
./bin/mongodump --out bak
```

然后关闭数据库，重新启动,并修复数据：
```
#如需指定WiredTiger引擎，请加参数--storageEngine wiredTiger
./bin/mongod --dbpath ./db/ &
./bin/mongorestore ./bak/
```

然后关闭数据库，修改配置文件mongodb.conf，并重新启动。
`/usr/local/mongodb2.6/bin/mongod --config /usr/local/mongodb2.6/bin/mongodb.conf &`

**3.5 验证数据库，创建用户**

通过mongo shell进入数据库命令行，注意，mongo shell版本与MongoDB不一致，就重新修改.bashrc文件：
**验证不通过的主要原因即为：mongo shell版本与MongoDB不一致，**
```
mongo
> show dbs
> use nodebb
> db.auth('nodebb', 'bioi......')
#正确返回1，错误返回error，如果错误重新设置用户名密码
> db.createUser( { user: "nodebb", pwd: "<Enter in a secure password>", roles: [ "readWrite" ] } )
> #For earlier versions of MongoDB (if the above throws an error)
> db.addUser( { user: "nodebb", pwd: "<Enter in a secure password>", roles: [ "readWrite" ] } )
```

如果保证数据正确，可以登录，那么重新启动nodebb就OK了!!!


## 4. 设置开机启动

```
nohup /usr/local/mongodb2.6/bin/mongod --config /usr/local/mongodb2.6/bin/mongodb.conf &
vim /etc/rc.d/rc.local
/usr/local/mongodb2.6/bin/mongod --config /usr/local/mongodb2.6/bin/mongodb.conf
```

## 5. MongoDB 3.2配置文件

MongoDB3.2版本的配置文件有较大变动，示例如下：
```
systemLog:
   destination: file
   path: "/usr/local/mongodb3.2/logs/mongod.log"
   logAppend: true
storage:
   journal:
      enabled: false
   dbPath: "/usr/local/mongodb3.2/db"
   directoryPerDB: false
   engine: wiredTiger
   wiredTiger:
      engineConfig:
         cacheSizeGB: 4
         directoryForIndexes: false
         journalCompressor: zlib
      collectionConfig:
         blockCompressor: zlib
      indexConfig:
         prefixCompression: true
net:
   port: 27017
processManagement:
   fork: true
security:
   authorization: enabled
```

## 参考

 - [http://www.tuicool.com/articles/f2aaquq](http://www.tuicool.com/articles/f2aaquq)
 - [http://www.dataguru.cn/thread-107361-1-1.html](http://www.dataguru.cn/thread-107361-1-1.html)
 - [http://m.blog.csdn.net/article/details?id=51111255](http://m.blog.csdn.net/article/details?id=51111255)
 - [http://www.cnblogs.com/kgdxpr/p/3519352.html](http://www.cnblogs.com/kgdxpr/p/3519352.html)

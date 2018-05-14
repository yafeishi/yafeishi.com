>ora2pg 工具安装说明
>author: yafeishi
>tags: AntDB,ora2pg,oracle
> AntDB:[github_url](https://github.com/ADBSQL/AntDB),基于postgresql的高性能分布式数据库

## ora2pg 工具安装说明
###	前言
#### 编写目的
本手册是对ADB产品ora2pg数据导出导入工具的操作说明，指导维护或开发人员了解该工具的使用维护。
#### 适用范围
适用于使用ADB产品的人员。

## 产品简介
#### 官方说明 
Ora2pg数据导出导入工具是perl语言编写的免费工具，可以用于将数据从oracle侧迁移至adbql数据库兼容的模式。
Ora2pg可以从oracle侧抽取结构或数据，组装成sql脚本的形式，加载到adbql数据库。

Ora2Pg is a free tool used to migrate an Oracle database to a adbQL compatible schema. It connects your Oracle database, scan it automaticaly and extracts its structure or data, it then generates SQL scripts that you can load into adbQL.
#### 工具版本号
Ora2pg版本号：ora2pg for adb 18.1
适用的ADB版本号：All

#### 	功能和用途
Ora2pg支持如下功能：
1.	导出全量数据库模式
2.	导出用户权限
3.	导出函数、存储过程、触发器
4.	支持数据类型转换
5.	支持oracle FDW
6.	支持oracle分区表

更多新功能可以参考官网说明。


### 系统运行环境
#### 软件环境

| 项目 | 值 |
|--------|--------|
|操作系统 |Red Hat Enterprise Linux Server release 6.3 (Santiago)+|
|内核版本 |2.6.32-279.el6.x86_64+|
|ora2pg版本|17.3|
|ADB版本 |All|
|Oracle版本|9以上|

### 配置概述
#### 安装概述
Ora2pg使用过程中需要使用oracle客户端，因此要求使用oracle系统用户进行安装。当然与ADB用户合设，也是允许的。
安装ora2pg需要使用sudo命令安装，因此需要配置ora2pg安装用户的sudo权限。
安装ora2pg工具，需要在系统提前安装完成如下软件包：
1. yum -y install cpan
2. yum -y install perl-Time-HiRes

#### 升级概述
全新安装
#### 配置概述
Ora2pg配置项概要说明：
具体配置项说明请参考 [http://ora2pg.darold.net/documentation.html](http://ora2pg.darold.net/documentation.html)


| 配置项| 配置值| 配置说明|
|------|---|---|
|ORACLE_HOME|/home/adb/oracle/instantclient_11_2|Oracle客户端根目录|
|ORACLE_DSN|dbi:Oracle:host=192.168.111.138;port=1521;sid=ADB|Oracle服务端连接串信息|
|ORACLE_USER|cmbase|Oracle用户|
|ORACLE_PWD|123456|Oracle密码|
|SCHEMA|cmbase|导出的模式|
|TYPE|TABLE,VIEW,SEQUENCE,GRANT,PARTITION|导出的类型|
|ALLOW||希望导出的对象|
|EXCLUDE|不希望导出的对象|
|WHERE|rownum <= 10000|当导出表的时候，运行添加一个where子句|
|INDEXES_SUFFIX|_idx|添加索引后缀信息|
|PREFIX_PARTITION |1|分区表是否以父表为前缀|
|FILE_PER_CONSTRAINT|0|是否每个约束一个文件|
|FILE_PER_INDEX|0|是否每个索引一个文件|
|FILE_PER_TABLE|0|是否每个表一个文件|
|DATA_TYPE||Oracle与pg数据类型|
|PG_NUMERIC_TYPE|0|Numeric类型转换|
|PG_INTEGER_TYPE|0|Numeric类型转换|
|DEFAULT_NUMERIC|float|numeric转换到pg默认的数据类型为float|
|DATA_LIMIT|100000|每次从oracle抽取的数据行数限制|
|LOG_ON_ERROR|0|当导入出错时，是否忽略错误继续导入|
|JOBS|5|多线程。并行导入pg时的线程数设置。线程数不超过CPU的核数。|
|ORACLE_COPIES|1|多线程。从oracle导出单张表数据时的线程数设置。线程数不超过CPU的核数。|
|PARALLEL_TABLES|1|多线程。从oracle导出数据时一次读取多少张表数据。一次读表的数量不超过CPU的核数。该配置与ORACLE_COPIES同时只能一个配置项生效。|
|DEFINED_PK|TABLE:COLUMN TABLE:ROUND(COLUMN)|与ORACLE_COPIES一起使用，定义单表以哪个字段取模并行导出。|


###	安装步骤
#### 环境准备
安装人员资格要求：熟悉Linux基本操作，有安装用户权限。
软件产品检查：检查设备的软件系统是否符合要求。
运行环境检查要点：检查设备当前CPU使用率、内存剩余等符合安装要求。
网络连通性检查要点：部署规划中涉及到所有节点网络连通，稳定。
#### 安装操作
##### 创建ora2pg安装目录并上传安装文件
注：本文把ora2pg、oracle客户端和ADB数据库安装在同一个用户的不同目录下。
举例：这里假设ADB安装用户为adb，根目录为/home/adb
以adb用户新建ora2pg、oracle客户端的根目录：

```shell
mkdir -p /home/adb/oracle/setup
```
以adb用户二进制方式上传ora2pg安装文件至上述目录

上传至如下目录：
- /home/adb/oracle/
- instantclient-basic-linux.x64-11.2.0.4.0.zip
- instantclient-sdk-linux.x64-11.2.0.4.0.zip
- instantclient-sqlplus-linux.x64-11.2.0.4.0.zip

上传至如下目录：
/home/adb/oracle/setup
- YAML-0.84.tar.gz
- DBI-1.634.tar.gz
- DBD-Oracle-1.75_2.tar.gz
- DBD-Pg-3.5.3.tar.gz
- ora2pg-17.3-1.tar.bz2

安装文件的下载链接请参考下载链接。
##### 安装oracle客户端
以adb用户进入oracle安装目录

```shell
cd /home/adb/oracle/
```
解压缩后生成instantclient_11_2目录，即为oracle精简客户端目录

```shell
unzip instantclient-basic-linux.x64-11.2.0.4.0.zip
unzip instantclient-sdk-linux.x64-11.2.0.4.0.zip
unzip instantclient-sqlplus-linux.x64-11.2.0.4.0.zip
```
新建tnsnames.ora文件

```shell

mkdir -p /home/adb/oracle/instantclient_11_2/network/admin
vim /home/adb/oracle/instantclient_11_2/network/admin/tnsnames.ora
```
编辑tnsnames.ora，增加对oracle数据库的监听配置
注：以下只是举例，实际配置以现场情况为准

```shell
ADB =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.111.138)(PORT = 1521))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = adb.cos01.com)
    )
  )
```

添加adb用户针对oracle客户端的环境变量

```shell
export ORACLE_HOME=/home/adb/oracle/instantclient_11_2
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$ORACLE_HOME
export PATH=$PATH:$ORACLE_HOME/bin:$ORACLE_HOME
```
让环境变量生效

```shell
source /ome/adb/.bash_profile
```
测试oracle连接

```shell
sqlplus system/manager@adb
SQL> select * from v$version;
```
返回信息，表示oracle客户端安装成功。


##### 配置sudo权限及安装必备软件包
###### 配置sudo权限
以root用户编辑/etc/sudoers文件,在root下方添加一行adb用户

```shell
adb ALL=(ALL) ALL
```
保存退出即可。
安装必备软件包
###### 以root用户执行安装：

```shell
yum -y install cpan
yum -y install perl-Time-HiRes
```
###### 解压3个tar包
以adb用户进入setup目录并解压

```shell
cd /home/adb/oracle/setup
tar -zxvf YAML-0.84.tar.gz
tar -zxvf DBI-1.634.tar.gz
tar -zxvf DBD-Oracle-1.75_2.tar.gz
```
###### 安装3个tar包
为root用户配置环境变量

```shell
vim /root/.bash_profile
```
添加或调整如下3个变量

```shell
export ORACLE_HOME=/home/adb/oracle/instantclient_11_2
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$ORACLE_HOME
export PATH=$PATH:$ORACLE_HOME/bin:$ORACLE_HOME
```
让变量生效

```shell
Source /root/.bash_profile
```
以root用户进入解压后的目录，分别安装

```shell
cd /home/adb/oracle/setup/YAML-0.84/
perl Makefile.PL 
make;make install

cd /home/adb/oracle/setup/DBI-1.634/
perl Makefile.PL 
make;make install

cd /home/adb/oracle/setup/DBD-Oracle-1.75_2/
perl Makefile.PL 
make;make install
```
###### 验证安装情况
编写checkperl.pl脚本

```perl
#!/usr/bin/perl
use strict;
use ExtUtils::Installed;

my $inst=ExtUtils::Installed->new();

my @modules = $inst->modules();

foreach(@modules){
        my $ver = $inst->version($_) || "???";
        printf("%-12s -- %s\n",$_,$ver);
}
```
执行该脚本

```shell
perl checkperl.pl
```
正常显示perl各模块的版本信息。
#####	安装ora2pg工具
###### 以adb用户进入ora2pg安装目录

```shell
cd /home/adb/oracle/setup
```
###### 解压缩安装文件

```shell
tar xf ora2pg-17.3-1.tar.bz2
```
###### 以root用户进入解压目录
```
cd /home/adb/oracle/setup/ora2pg-17.3/
```
###### 开始安装

```
perl Makefile.PL
make && make install
```
##### 安装验证
以adb用户执行下述命令，查看是否返回ora2pg的版本信息，如正确返回版本号，说明ora2pg安装成功。

```
ora2pg -v
Ora2Pg for ADB v18.1
```
#### 配置操作
##### Ora2pg的配置
###### 拷贝配置文件模板至/home/adb/oracle/目录

```
cp /home/adb/oracle/setup/ora2pg-17.3/ora2pg.conf.dist /home/adb/oracle/ora2pg.conf
```
###### 编辑配置文件/home/adb/oracle/ora2pg.conf
具体配置可以参考 `配置概述` 的配置说明。
##### ORACLE的配置
从oracle导出数据时，连接用户需要有dba权限，否则部分数据导出失败。
## 使用维护
#### ora2pg.conf配置示例

```
ORACLE_DSN	dbi:Oracle:host=192.168.111.138;port=1521;sid=ADB
ORACLE_USER	cmbase
ORACLE_PWD	123456
SCHEMA		channel
TYPE		TABLE,VIEW,SEQUENCE,GRANT,PARTITION
PKEY_IN_CREATE		1
INDEXES_SUFFIX		_idx
```
#### ora2pg配置项检查
show_report模式

```
ora2pg -c /home/adb/oracle/ora2pg.conf -n cmbase -t show_report -o /home/adb/oracle/cmbase_struct.sql
```
show_report模式不会实际去执行导出，但是可以检测oracle连接信息、导出的信息提供一个报告。
#### 从oracle抽取数据

```
ora2pg -c /home/adb/oracle/ora2pg.conf -n cmbase -o /home/adb/oracle/cmbase_struct.sql &
```
#### 导入数据至ADB

```
psql -d adb -h 192.168.111.138 -p 5432 -U cmbase -f /home/adb/oracle/cmbase_struct.sql &
```

### 注意事项
1.	为了提高数据迁移效率，ora2pg提供了一些性能调优参数。
比如 DATA_LIMIT、ORACLE_COPIES、PARALLEL_TABLES等，实际操作过程中，请合理设置这些参数。
2.	LOG_ON_ERROR参数可以控制从oracle抽取数据生成的sql文件，当导出时遇到问题，是否退出执行或忽略错误继续执行。


### 下载链接
Ora2pg最新工具下载链接URL    
[https://github.com/ADBSQL/adb-ora2pg](https://github.com/ADBSQL/adb-ora2pg)

oracle精简客户端工具下载链接URL     
http://www.oracle.com/technetwork/database/features/instant-client/index-097480.html    

安装ora2pg工具必备的系统软件包下载链接URL    
http://search.cpan.org/~mstrout/YAML-0.84/lib/YAML.pm    
http://search.cpan.org/~timb/DBI-1.634/DBI.pm    
http://search.cpan.org/~pythian/DBD-Oracle-1.74/    

###	参考链接
Ora2pg工具安装配置使用说明链接URL：    
http://ora2pg.darold.net/documentation.html     
http://blog.csdn.net/kiwi_kid/article/details/50846593    







>使用Ora2pg和psql copy方式进行数据迁移
>author: yafeishi
>tags: AntDB,ora2pg,oracle
> AntDB:[github_url](https://github.com/ADBSQL/AntDB),基于postgresql的高性能分布式数据库

# 使用Ora2pg和psql copy方式进行数据迁移

##	准备工作
### 使用本文档的前提
本文档指导如何使用ora2pg进行oracle到ADB的数据迁移，但是在参照本文档操作之前，有以下条件必须满足：
- ADB集群安装并可以正常使用
- ADB集群中的机器可以正确访问源端Oracle数据库
- ADB集群中的机器上都正确安装了ora2pg
- ADB集群中的机器上有充足的空间存放导出的sql文件
- ADB集群中的机器上有充足的空间存放需要导入的数据

另外需要说明的是，按照本文档的操作方式，ora2pg导出的数据文件为copy方式，可以直接通过psql -f 方式执行。示例如下：

```sql
COPY table_name (col1,col2,col3,col4) FROM STDIN;
data data data data
....
\.
```
### 确定迁移范围

|序号|基表名|分表条件 1:年月分表 _YYYYMM（原历史库）2:地市分表 _21X，3:地市年月分表 21X_YYYYMM（原历史库），4:非分表|
|---|---|---|
|1|INS_ACCREL|4|
|2|INS_OFFER|2|
|...|...|...|

注：带年月的表导出范围为：

```sql
expire_date >= to_date('2015/06/01 00:00:00','yyyy/MM/dd hh24:mi:ss') and expire_date < to_date('2016/07/01 00:00:00','yyyy/MM/dd hh24:mi:ss')
```
###	数据量大小查询@oracle
在oracle侧查询迁移数据的大小，包括表大小和索引大小。如果给的用户没有查询dba_视图的权限，所以需要通过user_视图分用户查询：
表大小：

```sql
select sum(b.bytes)/1024/1024/1024 as table_size
from user_tables a ,user_segments b
where 1=1
and a.table_name=b.segment_name
and a.table_name like 'INS_OFFER_21%'
;
```

索引大小：

```sql
select round(sum(b.bytes)/1024/1024/1024,2) as index_size
from user_indexes a ,user_segments b
where 1=1
and a.index_name=b.segment_name
and a.table_name like 'INS_OFFER_21%'
;
```
编写查询脚本，传参为用户名（大写）和基表名称（大写）：

```shell
#!/bin/bash
username=$1
tablelike=$2

oraconn="$username/password@10.10.123.123/crm"

index_size=`sqlplus -S /nolog <<EOF
set heading off feedback off pagesize 0 verify off echo off
conn $oraconn
select round(sum(b.bytes)/1024/1024/1024,2) 
from user_indexes a ,user_segments b
where 1=1
and a.index_name=b.segment_name
and a.table_name like '$tablelike%'
;
exit
EOF`

table_size=`sqlplus -S /nolog <<EOF
set heading off feedback off pagesize 0 verify off echo off
conn $oraconn
select round(sum(b.bytes)/1024/1024/1024,2) as index_size
from user_tables a ,user_segments b
where 1=1
and a.table_name=b.segment_name
and a.table_name like '$tablelike%'
;
exit
EOF`

echo "$username table $tablelike  table_size $table_size,index_size:$index_size"
```

分用户编写查询脚本，然后后台执行：

```shell
sh s.sh user1  INS_OFFER_21

sh s.sh user2  INS_ACCREL_21

nohup sh s_user1.sh > s_user1.out 2>&1 &
nohup sh s_user2.sh > s_user2.out 2>&1 &
```

查询结果：

|用户|基表名|表大小(GB)|索引大小(GB)|
|---|---|---|---|
|user1|INS_OFFER|10|5|
|user2|INS_ACCREL|25|10|
|...|...|...|...|

##	导出导入对象
### 创建相关目录：

```shel;
mkdir -p /adbdata/ora2pg/{scripts,conf,ddl,data,log}
```
- scripts 存放导出导入操作需要使用的shell脚本。
- conf 存放导出对象和数据时候需要使用的配置文件。
- ddl 存放导出的对象。
- data存放导出的数据。
- log 存放导出导入进程产生的日志。


我们使用ora2pg导出迁移范围内的表对象和数据。
### 导出对象
#### 编写配置文件
ora2pg 导出对象的配置文件如下：

```shell
cat > crm_user1_ddl.conf  << EOF
ORACLE_HOME /home/postgres/oracle/instantclient_11_2  
ORACLE_DSN  dbi:Oracle:host=10.10.123.123;sid=crm
ORACLE_USER user  
ORACLE_PWD  password  
SCHEMA  user1
TYPE TABLE
USER_GRANTS 1
PKEY_IN_CREATE  1
UKEY_IN_CREATE  1
DISABLE_SEQUENCE 1
INDEX_RENAMING 0
STOP_ON_ERROR 0
ALLOW INS_OFFER_21.*
EOF
```
建议配置文件放在 ora2pg/conf 目录下。
#### 执行导出操作

```shelll
nohup ora2pg -c /adbdata/ora2pg/conf/crm_user1_ddl.conf  -o /adbdata/ora2pg/ddl/crm_user1_ins.sql -d  > /adbdata/ora2pg/log/crm_user1_ins.log 2>&1 &
```
#### 查看导出日志

```
tail -f /adbdata/ora2pg/log/crm_user1_ins.log
```
###	导入对象
导出的sql文件可以直接通过psql端执行，如下：

```
psql -p 5432 -d crm -u user1 -f /adbdata/ora2pg/ddl/crm_user1_ins.sql
```

### 对象稽核
对象稽核主要是看两端数据库中在迁移范围内，表数量和索引数量是否一致。
#### Oracle侧查询
Oracle通过 dba_tables/dba_indexes 或者 user_table/user_indexes来查询：

```shell
vi check_obj_ora.sh
#!/bin/bash


sqldir="/data/ora2pg/data/crm"
owner=$1
type=$2
tablelike=(ORD_BUSI_PRICE_F
ORD_CUST_F ORD_OFFER_F )


usertns="username/passwd@10.10.123.123/crm"

for t in ${tablelike[@]}
do
sqlplus -S /nolog <<EOF
set heading off feedback off pagesize 0 verify off echo off
conn $usertns
select '$t',owner,count(*)
from $type a
where 1=1
and a.owner=upper('$owner')
and a.table_name like '$t%'
group by owner;
EOF
done
```
用法：sh check_obj_ora.sh user1 dba_index
#### ADB侧查询
ADB侧通过 pg_tables/pg_indexes来查询：

```
vi check_obj_adb.sh
#!/bin/bash


sqldir="/data/ora2pg/data/crm"
owner=$1
type=$2
tablelike=(ORD_BUSI_PRICE_F
ORD_CUST_F ORD_OFFER_F)


psqlconn="psql -d shhis -U $owner -q -t"


for t in ${tablelike[@]}
do
$psqlconn << EOF
select '$t',schemaname,count(*)
from $type a
where 1=1
and a.schemaname='$owner'
and upper(a.tablename) like '$t%'
group by schemaname;
EOF
done
```
用法：sh check_obj_adb.sh user1 pg_tables



## 导出导入数据
### 导出数据
#### 编写配置文件
ora2pg 导出数据的配置文件如下：

```
cat > crm_user1_data.conf << EOF
ORACLE_HOME /home/adb/oracle/instantclient_11_2  
ORACLE_DSN  dbi:Oracle:host=10.10.123.123;sid=crm  
ORACLE_USER user1  
ORACLE_PWD  password  
SCHEMA  user
TYPE COPY 
OUTPUT_DIR  /adbdata/ora2pg/data/user1
USER_GRANTS 1
FILE_PER_TABLE 1
SPLIT_FILE 1
DISABLE_SEQUENCE 1
STOP_ON_ERROR 0
ORACLE_COPIES 10
DATA_LIMIT  100000
SPLIT_LIMIT  10000000
WHERE expire_date >= to_date('2015/06/01 00:00:00','yyyy/MM/dd hh24:mi:ss') and expire_date < to_date('2016/07/01 00:00:00','yyyy/MM/dd hh24:mi:ss')
EOF
```
OUTPUT_DIR 指定了导出sql文件的存放目录。
WHERE 指定了导出时候对数据的过滤条件，如果需要全表数据导出，则去掉该参数。
#### 创建相关目录

```
mkdir -p /adbdata/ora2pg/data/{user1,user2}/import_done
```
import_done 存放各个用户下，导入完成后的sql文件。
#### 表数量比较少
如果迁移的表数量比较少(小于20张)，那么直接执行ora2pg命令即可：

```
nohup ora2pg -c /adbdata/ora2pg/conf/crm_user1_data.conf -a"INS_OFFER_21.*" -d > /adbdata/ora2pg/log/ora2pg_crm_user1_INS_OFFER_21.log 2>&1 &
```
其中 -a参数指定匹配的表名，"INS_OFFER_21.*"表示以INS_OFFER_21开头的所有表。
也可以封装在脚本中：

```
vi exec_export.sh
usenname=$1
configfile="/adbdata/ora2pg/conf/crm_${usenname}_data.conf"
logdir=/adbdata/ora2pg/log
tablelike=$2
nohup ora2pg -c $configfile -a"$tablelike.*" -d > ${logdir}/ora2pg_${usenname}_${tablelike}.log 2>&1 &
```
执行封装脚本：

```
sh exec_export.sh user1   INS_OFFER_21
```
执行后，查看日志：

```
tail -f /adbdata/ora2pg/log/ora2pg_crm_user1_INS_OFFER_21.log
```
导出的文件会存放配置文件中的OUTPUT_DIR参数值中。正在导出的文件以exporting_table_name_1.sql.tmp命名，导出完成的文件以table_name_1.sql 命名。
#### 表数量比较多
在迁移涉及到的表数量比较多的时候，建议采用表的方式来记录迁移表的信息，方便查询表的迁移状态。
##### 创建配置表
创建配置表用来存放迁移表的信息：

```sql
create table t_ora2adb_tableinfo
(
dbname text,
owner text,
tablename text,
batch_tag text,
o_cnt numeric,
a_cnt numeric,
o_cnt_time timestamp,
a_cnt_time timestamp,
o_minus_a numeric,
a_minus_o numeric,
o_size_m numeric,
a_size_m numeric,
is_export numeric,
export_time timestamp,
where_filter text
);
```

#####	初始化配置表

```sql
insert into t_ora2adb_tableinfo
(dbname,owner,tablename,is_export)
select 'shhis',a.tableowner,a.tablename,0
from pg_tables a
where 1=1
and a.tableowner not in ('postgres','') 
and a.schemaname not in ('public','pg_catalog','information_schema')
and a.tablename like 'sec%';
;
```

#####	更新配置表字段
字段is_export的值在初始化的时候设置为0，需要更新该字段为1或者更新其他字段，则通过update语句来操作：
`update t_ora2adb_tableinfo set xx=xx where xx ..`

##### 编写导出脚本

```shell
vi ora2pg_export.sh
#!/bin/bash 
# ora2pg export 
# ora2pg config: shhis_data.conf
# sh ora2pg_export.sh owner tablenamelike tablecnt

# sh ora2pg_export.sh configfile username tablename  logdir tablecnt
 
function check_ora_conn
{
conn_tag=`sqlplus -S /nolog <<EOF
set heading off feedback off pagesize 0 verify off echo off
conn $oraconn
select * from dual;
exit
EOF`
if [ "x$conn_tag" = "x" ]; then
  echo "the oracle connection is invalid!"	
	exit(0)
else
	echo "the oracle connection is ok!"	
fi
}


function check_ora2pg
{
  which ora2pg
  if [ $? -eq 0 ]; then
        return 0
  else
        echo "ora2pg execute program not find!"
        exit        
  fi
}


function check_ora2pgcfg
{
  if [ -f "$configfile" ]; then
        return 0
  else
        echo "ora2pg config file does not exist!"
        exit        
  fi
}

function init_tablecnt
{
if [ "x$tablecnt" = "x" ]; then
  echo "tablecnt apply init value:10"
  tablecnt=10
fi
}


function ora2pg_background
{
selectsql="select tablename 
from t_ora2adb_tableinfo 
where dbname='$dbname' 
and owner='$tableowner' 
and tablename like '%$tablelike%'
and is_export=0
limit $tablecnt;"

tables=(`$psqlconn -c "$selectsql"`)
for t in ${tables[@]}
do
 updatesql="update t_ora2adb_tableinfo set is_export=1,export_time=now() where owner='$tableowner' and tablename='$t'"
 $psqlconn -c "$updatesql"
 ora2pg -c $configfile -n $tableowner -a"$t" -d > ${logdir}/"ora2pg_"$tableowner_$t.log 2>&1 &
 echo "ora2pg to export table: $tableowner.$t"
done
}


# init parameter

configfile=$1
username=$2
tablelike=$3
logdir=$4
tablecnt=$5

tableowner="$username"
dbname="crm"

# Make sure this option is correct,If the oracle database is not connected, the program will exit 
oraconn="$username/password@10.10.123.123/crm"


psqlconn="psql -d $dbname -U $tableowner -q -t"

check_ora2pg
check_ora2pgcfg
init_tablecnt
ora2pg_background
```


##### 执行导出操作

```shell
sh ora2pg_export.sh /adbdata/ora2pg/conf/crm_user1_data.conf user1 ord_user_ext_f_210_201612 /adbdata/ora2pg/log 3

```
#####	查看导出日志
导出日志存放在脚本指定的日志目录中，每张表一个日志文件：
`/adbdata/ora2pg/log/ora2pg_ins_offer_f_210_201612.log`
### 导入数据
导出的数据文件为copy命令的sql文件，可以直接使用`psql -f`参数执行。但是在文件数量比较多的时候，建议采用脚本封装的方式，可控的进行数据导入。
####	编写导入脚本
该脚本通过命令行参数来控制是通过表名查找文件导入还是按照文件大小排序后取一定的文件数来进行导入。用法如下：

```
sh psqlcopy.sh -u user1 -d /adbdata/ora2pg/data/user1  -l /adbdata/ora2pg/log -s 11
```
按照文件大小正序排序后取前11个文件

```
sh psqlcopy.sh -u user1 -d /adbdata/ora2pg/data/user1  -l /adbdata/ora2pg/log -S 11
```
按照文件大小正序倒序后取前11个文件

```
sh psqlcopy.sh -u user1 -d /adbdata/ora2pg/data/user1  -l /adbdata/ora2pg/log -t  INS_DES_ACCREL
```
导入以INS_DES_ACCREL开头的表文件

脚本内容如下：

```shell
vi psqlcopy.sh
#!/bin/bash

source /home/postgres/.bashrc

function check_dir
{
 if [ ! -d "$2" ]; then
     echo "the $1 direcory $2 does not exist!"
     exit 0
 fi
}


function usage
{
  echo "Usage: "
  echo "    `basename $0` -u username -d datadir -t tablename -l logdir"
  echo "    `basename $0` -u username -d datadir -s|S filecnt -l logdir"
  exit 0
}


function psql_copy_file
{
$psqlconn -f ${datadir}/$file"_importing" > ${logdir}/"psqlcopy_"$file.log 2>&1 
errcnt=`grep -E "ERROR|FATAL" ${logdir}/"psqlcopy_"$file.log | wc -l`
if [ "$errcnt" -gt 0 ];then
 echo `date "+%Y-%m-%d %H:%M:%S" `": file ${datadir}/$file"_importing" copy failed,please check log ${logdir}/"psqlcopy_"$file.log!" >> ${logdir}/"psqlcopy_error".log
elif [ "$errcnt" -eq 0 ]; then
 mv ${datadir}/$file"_importing" ${datadir}/import_done/$file"_done"
fi
}



function exe_psql_copy
{
if [ "x$tablelike" != "x" ]; then
   files=`ls -rt $datadir  |grep -E '*sql$' |grep -v tmp | grep -v done |grep -v import |grep $tablelike`
elif [ $filecnt -gt 0 ]; then
   if [ $file_sort -eq 0 ]; then 
      lsopt="ls -rS"
   elif [ $file_sort -eq 1 ]; then
      lsopt="ls -S"
   fi       
   files=`$lsopt $datadir  |grep -E '*sql$' |grep -v tmp | grep -v done |grep -v import |head -$filecnt`
fi

# rename file to filename_importing
for file in ${files[@]}
do 
  mv ${datadir}/$file ${datadir}/$file"_importing"
done 

# execute copy, after move file to done dir
for file in ${files[@]}
do 
  echo `date "+%Y-%m-%d %H:%M:%S"` " execute file: $file ,file size :`du -sh ${datadir}/$file"_importing" |awk '{print $1}'`"
  #echo `date "+%Y-%m-%d %H:%M:%S"` " execute file: $file ,file size :`du -sh ${datadir}/$file |awk '{print $1}'`"

  # sh /data/ora2pg/scripts/psqlcopyfile.sh  $dbname $tableowner $file_dir $file >> ${logdir}/"psqlcopy".log 2>&1 &
  psql_copy_file &
done
}


function check_adb_conn
{
result==`$psqlconn -q -t -c "select 1"`
if [ "x$result" = "x" ];then
    echo "connect to adb is invalid,please check!"
    exit 1
}


function dir_stat
{
exporting_cnt=`ls -rt $datadir  |grep  'exporting' |grep -v done |wc -l`
importing_cnt=`ls -rt $datadir  |grep  'importing' |grep -v done |wc -l`
noimport_cnt=`ls -rt $datadir  |grep -E '*sql$' |grep -v tmp | grep -v done |grep -v import |wc -l`
done_cnt=`ls -rt $datadir/import_done |grep "done" |wc -l`
echo `date "+%Y-%m-%d %H:%M:%S"` "data directory $datadir stat: exporting:$exporting_cnt,noimport:$noimport_cnt,importing:$importing_cnt,import_done:$done_cnt"
}

while getopts 'u:d:t:s:S:l:' OPT; do
    case $OPT in
        u)
            username="$OPTARG";;
        d)
            datadir="$OPTARG";;
        t)
            tablelike="$OPTARG";;
        s)
            file_sort=0
            filecnt="$OPTARG";;
        S)
            file_sort=1
            filecnt="$OPTARG";;            
        l)
            logdir="$OPTARG";;            
        ?)
            usage
    esac
done

check_dir "data" $datadir
check_dir "log" $logdir


# should use -t or -s|S option.
if [ "x$tablelike" != "x" -a "x$filecnt" != "x" ]; then
   echo "you should use -t or -s|S option."
   usage
fi   

dbname='crm'
hostname='10.10.123.123'
port="5432"
psqlconn="psql -h $hostname -p $port -d $dbname -U $username "

echo "username:$username,datadir:$datadir,tablelike:$tablelike,file_sort:$file_sort,filecnt:$filecnt,logdir:$logdir"
check_adb_conn
exe_psql_copy
dir_stat
```

####	查找文件
提供一个脚本，根据文件名查找文件是否存在：

```
vi findfile.sh
echo  "-------------------------------------`date`"
dir=$1
find $dir -maxdepth 2 -name "*$2*"  -exec ls -lhr {} \;
```
因为导出文件是以表名命名的，所以，根据表名(大写)查找出来的文件，通过标识(exporting，done等)可以知道表的状态(导出中、导出完成、导入中、导入完成)。
#### 查看日志
日志文件在指定的日志目录中存放，命名方式为：
`psqlcopy_filename.log `
在日志目录中，会有一个总的记录导入失败的文件的日志：`psqlcopy_error.log`，记录导入失败的文件，并提供对应的日志文件全路径。

## 资源监控
在执行导出导入过程中，要随时查看进程所在主机的资源。提供一个简单脚本，在操作过程可以查看资源情况：

```
vi stat.sh
#/bin/bash 
# stat.sh

echo  "-------------------------------------`date`"
echo ""
echo "connect to oracle processes: `ps -ef|grep "ora2pg - querying table" |grep -v grep |wc -l`"
echo "sending data to file processes: `ps -ef|grep "ora2pg - sending data from" |grep -v grep | wc -l`"
echo "psql copy processes: `ps -ef|grep COPY |grep -v local| grep -v grep | wc -l`"
dstat -am 1 3
sar -r 1 3
iostat -d sdb -x -k 1 3
```
在资源充足的时候，可以多启动几个进程，资源不足的时候，就要慎重决定启动的进程数量。做到充分利用硬件资源，但不可过度使用硬件资源，造成资源竞争激烈的话，耽误的时间就得不偿失了。
##	数据稽核
### 两边数据库计算count(*)结果
同样，表少的情况下，可以直接使用`select count(*)` 在两边数据库查询。
表多的时候，依然建议将查询结果存表。
#### Oracle侧查询
oracle侧查询脚本，默认在Oracle起三个并行进行查询，通过配置表取表名，然后更新配置表的o_cnt字段：

```
vi o_cnt.sh
#/bin/bash
# sh o.sh user1 ins_des_user_ext_210 > o_cnt.log 2>&1 &
dbname="crm"
owner=$1
tablename=$2

psqlconn="psql -d $dbname -U adb -q -t"
selectsql="select tablename 
from t_ora2adb_tableinfo_cnt
where dbname='$dbname' 
and owner='$owner' 
and upper(tablename) like '$tablename%'
--and o_cnt is null or o_cnt=0
"

usertns="username/password@10.10.123.123/crm"
oracleparallel=3


tables=(`$psqlconn -c "$selectsql"`)
# oracle count
for t in ${tables[@]}
do 
# oracle count
o_cnt=`sqlplus -S /nolog <<EOF
set heading off feedback off pagesize 0 verify off echo off
conn $usertns
select /*+ parallel($oracleparallel)*/ count(*) from $owner.$t;
exit
EOF`
# oracle size 
o_size_m=`sqlplus -S /nolog <<EOF
set heading off feedback off pagesize 0 verify off echo off
conn $usertns
select round(sum(bytes)/1024/1024,2)  
from dba_segments
--from user_segments
where 1=1
and owner=upper('$owner')
and segment_name=upper('$t');
exit
EOF`

echo `date "+%Y-%m-%d %H:%M:%S"` "table_cnt on oracle: $owner:$t:$o_cnt:$o_size_m MB"
$psqlconn << EOF
update t_ora2adb_tableinfo_cnt set o_cnt=$o_cnt,o_size_m=$o_size_m,o_cnt_time=now() 
where 1=1
and owner='$owner' 
and tablename like '$t'
;
EOF
done
```

####	ADB侧查询
同样通过配置表取表名，然后更新配置表的a_cnt字段：

```
vi a_cnt.sh
#!/bin/bash
# sh a.sh user1 ins_des_user_ext_210 > o_cnt.log 2>&1 &

source /home/adb/.bashrc
# init parameter
dbnname="crm"
owner=$1
tablename=$2

# oracle connect info

# adb connect info
updatepsqlconn="psql -d $dbnname -U adb -q -t"
cntpsqlconn="psql -d $dbnname -U $owner -q -t"
selectsql="select tablename 
from t_ora2adb_tableinfo_cnt
where dbname='$dbnname' 
and owner='$owner' 
and upper(tablename) like '$tablename%'
--and o_cnt>0
--and (a_cnt is null or a_cnt=0 or o_minus_a <> 0)
"

tables=(`$updatepsqlconn -c "$selectsql"`)
# oracle count
for t in ${tables[@]}
do 
a_cnt=`$cntpsqlconn << EOF 
select count(*) from $owner.$t;
EOF`
a_size_m=`$cntpsqlconn << EOF 
select pg_relation_size('$owner.$t')/1024/1024;
EOF`
echo `date "+%Y-%m-%d %H:%M:%S"` "table_cnt on adb: $owner:$t:$a_cnt:$a_size_m MB"
$updatepsqlconn << EOF
update t_ora2adb_tableinfo_cnt set a_cnt=$a_cnt,a_size_m=$a_size_m,a_cnt_time=now() 
where 1=1
and owner='$owner' 
and tablename like '$t'
;
EOF
done 
$updatepsqlconn -c "update t_ora2adb_tableinfo_cnt set o_minus_a=o_cnt-a_cnt;"
```
两个脚本中获取表名的sql为selectsql，可以根据实际情况进行修改。
## 其他会用到的脚本
### 索引相关
#### 生成创建删除索引的脚本

```shell
vi generate_create_index.sh
#!/bin/bash
# sh 1.sh user1 ins_des_user_ext_21

dbname="shhis"

tableowner=$1
tablelike=$2
filedir=$3
psqlconn="psql -d $dbname -U $tableowner -q -t"
selectsql="select tablename from pg_tables where 1=1 and tableowner='$tableowner' and tablename like '$tablelike%'"
#selectsql="select tablename
#from t_ora2adb_tableinfo
#where 1=1
#and o_cnt>0
#--and a_cnt>0
#and o_minus_a<>0
#and owner='$tableowner'
#and tablename like '$tablelike%'
#order by o_size_m desc"


#echo "$selectsql"

tables=(`$psqlconn -c "$selectsql"`)

for t in ${tables[@]}
do 
echo `date "+%Y-%m-%d %H:%M:%S"` "----generate  create index sql on  table $tableowner.$t"
$psqlconn << EOF > ${filedir}/'create_index_'$tableowner'_'$t'.sql' 2>&1 
\echo select now();
\echo '\\\timing on'
select a.indexdef||';'
from pg_indexes a,pg_index b
where 1=1
and a.indexname::regclass=b.indexrelid::regclass
and a.tablename::regclass=indrelid::regclass
and b.indisunique=false
and b.indisprimary=false
and a.schemaname not in ('pg_catalog','public')
and a.schemaname='$tableowner'
and a.tablename='$t'
;
\echo select now();
EOF
done
```
```shell
vi  generate_drop_index.sh
#!/bin/bash
# sh 1.sh user1 ins_des_user_ext_21

dbname="shhis"

tableowner=$1
tablelike=$2
filedir=$3
psqlconn="psql -d $dbname -U $tableowner -q -t"
selectsql="select tablename from pg_tables where 1=1 and tableowner='$tableowner' and tablename like '$tablelike%'"
#selectsql="select tablename
#from t_ora2adb_tableinfo
#where 1=1
#and o_cnt>0
#--and a_cnt>0
#and o_minus_a<>0
#and owner='$tableowner'
#and tablename like '$tablelike%'
#order by o_size_m desc"

#echo "$selectsql"

tables=(`$psqlconn -c "$selectsql"`)

for t in ${tables[@]}
do 
echo `date "+%Y-%m-%d %H:%M:%S"` "----generate  drop index sql on  table $tableowner.$t"
$psqlconn << EOF > ${filedir}/'drop_index_'$tableowner'_'$t'.sql' 2>&1
select 'drop index '||a.schemaname||'.'||a.indexname||';'
from pg_indexes a,pg_index b
where 1=1
and a.indexname::regclass=b.indexrelid::regclass
and a.tablename::regclass=indrelid::regclass
and b.indisunique=false
and b.indisprimary=false
and a.schemaname not in ('pg_catalog','public')
and a.schemaname='$tableowner'
and a.tablename = '$t'
EOF
done
```

#### 执行创建删除索引的脚本

```shell
vi execute_drop_index.sh
#!/bin/bash
# sh 1.sh user1 ins_des_user_ext_21

dbname="crm"
port=5433
tableowner=$1
tablelike=$2
filedir=$3
#psqlconn="psql -d $dbname -U $tableowner -q -t -p $port"
psqlconn="psql -d $dbname -U user1 -q -t -p $port -h 10.11.123.123"
selectsql="select tablename from pg_tables where 1=1 and tableowner='$tableowner' and tablename like '$tablelike%'"
#selectsql="select tablename
#from t_ora2adb_tableinfo
#where 1=1
#and o_cnt>0
#--and a_cnt>0
#and o_minus_a<>0
#and owner='$tableowner'
#and tablename like '$tablelike%'
#order by o_size_m desc"

tables=(`$psqlconn -c "$selectsql"`)

for t in ${tables[@]}
do 
echo `date "+%Y-%m-%d %H:%M:%S"` "----on port $port --- drop index on  table $tableowner.$t"
$psqlconn -f ${filedir}/'drop_index_'$tableowner'_'$t'.sql' >> drop_index_${port}_$tableowner.log 2>&1 
done
```
```shell
vi execute_create_index.sh
#!/bin/bash
# sh 1.sh sodang ins_des_user_ext_21

dbname="shhis"

tableowner=$1
tablelike=$2
filedir=$3
psqlconn="psql -d $dbname -U $tableowner -q -t"
selectsql="select tablename from pg_tables where 1=1 and tableowner='$tableowner' and tablename like '$tablelike%'"

tables=(`$psqlconn -c "$selectsql"`)

for t in ${tables[@]}
do 
$psqlconn -f ${filedir}/'create_index_'$tableowner'_'$t'.sql' >> create_index_$tableowner.log 2>&1 &
done
```








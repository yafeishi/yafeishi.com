
>python 连接AntDB进行数据库操作
>author: yafeishi
>tags: AntDB,python

AntDB基于postgres开发，所以使用连接postgres的python驱动即可。
python连接postgre的驱动也有好几个，具体可看：[https://wiki.postgresql.org/wiki/Python](https://wiki.postgresql.org/wiki/Python)
本例中我们选择流行度比较高的psycopg2。文档地址：[http://initd.org/psycopg/docs/](http://initd.org/psycopg/docs/)，py2、py3均可使用。

#### 安装psycopg2：
在centos下，直接执行：   

```
yum install python-psycopg2
```

如果是内网环境，且没有yum源，可以去github上下载源码进行安装，源码地址：[https://github.com/psycopg/psycopg2](https://github.com/psycopg/psycopg2),按照readme的说明操作即可。

安装好psycopg2之后，进行测试：

```
python -c "import psycopg2"
```
返回空，表示安装成功。

#### 连接AntDB：

如果有使用python连接postgres的经验，那在AntDB中操作是完全一样的。
首先需要import psycopg2，然后使用psycopg2提供的conn方法创建连接即可。代码示例如下：

```python
import psycopg2
conn = psycopg2.connect(database="postgres", user="dang31", password="123", host="10.1.226.201", port="17613")
```
此时就创建好了一个连接，供后续使用。
在进行数据库操作之前，需要先建立一个curosr，在cursor中进行相应的操作：

```
cur = conn.cursor()
```
创建表：

```
cur.execute("create t_test (id int,name text)")
```
插入数据：

```
cur.insert("insert into t_test values (1,'name')")
```
删除数据：

```
cur.execute("delete from t_test where id=%s",(3,)) 
```
更新数据：

```
cur.execute("update  t_test set name=%s where id=%s",('c',1)) 
```
查询数据：

```
cur.execute("select * from t_test;")
rows = cur.fetchall()
pprint.pprint(rows)
```

完整的脚本如下：

```python
#!/usr/bin/env python
# -*- coding: UTF-8 -*-

import psycopg2
import sys
import pprint

#adb_conn="dbname=postgres user=dang31 password=123 host=10.11.6.20 port=17322"
adb_conn="dbname=postgres user=dang31 password=123 host=10.1.226.201 port=17613"
pg962_conn="dbname=postgres user=danghb password=123 host=10.21.16.14 port=7632"

#conn = psycopg2.connect(database="postgres", user="dang31", password="123", host="10.1.226.201", port="17613")

#conn = psycopg2.connect("dbname=postgres user=sh2.2 password=123 host=10.1.226.201 port=17322")


#host=localhost port=5432 dbname=mydb user=postgres password=123456"


try:
   conn = psycopg2.connect(adb_conn)
except psycopg2.Error as e:
    print"Unable to connect!"
    print e.pgerror
    print e.diag.message_detail
    sys.exit(1)
else:
    print"Connected!"    
#conn = psycopg2.connect(database="postgres", user="dang31", password="123", host="10.1.226.201", port="17613")
    cur = conn.cursor()
#该程序创建一个光标将用于整个数据库使用Python编程。
print ("version:")
cur.execute("select version();")  
rows = cur.fetchall()
pprint.pprint(rows)
print ("create  table")
cur.execute("create table t_test (id int,name text);")   
print ("insert into table")
cur.execute("insert into t_test (id,name) values (%s,%s)",(1,'a'))
cur.statusmessage
cur.execute("insert into t_test (id,name) values (%s,%s)",(3,'b')) 
cur.mogrify("insert into t_test (id,name) values (%s,%s)",(3,'b')) 
cur.execute("select * from t_test;")
print ("fetchone")
row = cur.fetchone()
pprint.pprint(row)
cur.execute("select * from t_test;")
rows = cur.fetchall()
print ("fetchall")
pprint.pprint(rows)
print ("delete from table")
cur.execute("delete from t_test where id=%s",(3,)) 
cur.execute("select * from t_test;")
rows = cur.fetchall()
pprint.pprint(rows)
print ("update  table")
cur.execute("update  t_test set name=%s where id=%s",('c',1)) 
cur.execute("select * from t_test;")
rows = cur.fetchall()
pprint.pprint(rows)
print ("drop  table")
cur.execute("drop table if EXISTS  t_test ");

conn.commit()
#connection.commit() 此方法提交当前事务。如果不调用这个方法，无论做了什么修改，自从上次调用#commit()是不可见的，从其他的数据库连接。
conn.close() 
#connection.close() 此方法关闭数据库连接。请注意，这并不自动调用commit（）。如果你只是关闭数据库连接而不调用commit（）方法首先，那么所有更改将会丢失
```

输出结果为：

```
Connected!
version:
[('PostgreSQL 9.6.2 ADB 3.1devel d4afccf on x86_64-pc-linux-gnu, compiled by clang version 3.4.2 (tags/RELEASE_34/dot2-final), 64-bit',)]
create  table
insert into table
fetchone
(1, 'a')
fetchall
[(3, 'b'), (1, 'a')]
delete from table
[(1, 'a')]
update  table
[(1, 'c')]
drop  table
```




>python 离线环境安装python第三方库
>author: yafeishi
>tags: AntDB,python

python对于运维工作确实方便了很多，但很多比较实用的库都是第三方提供，在os自带的iso中并没有，离线环境下安装第三方库是一件很痛苦的事情，无止境的依赖会让你崩溃。能不能在离线环境中像在线环境一样通过pip来解决依赖问题呢？答案是可以的(让我一个一个安装依赖是不可能的，这辈子都不可能的)。

以在离线环境中安装paramiko为例进行说明：
paramiko虽然好用，但是依赖很多，多到甚至我想放弃使用，直到找到了pip 离线安装解决依赖的方法，感觉世界又充满了阳光。
大致思路是：
1. 离线环境中安装setuptools和pip
2. 在在线环境通过pip将paramiko的依赖下载到一个文件夹里
3. 在离线环境中，通过pip访问该文件夹来解决依赖问题，顺利安装。

首先，离线环境中，需要安装setuptools和pip：

```
unzip setuptools-39.1.0.zip
cd setuptools-39.1.0 && $SUDO python setup.py install
cd ..
tar xzvf pip-10.0.1.tar.gz
cd pip-10.0.1 && $SUDO python setup.py install
cd ..
```

在在线环境中，下载paramiko的依赖：

```
pip download -d /tmp/paramiko paramiko
[emea@intel175 tmp]$ cd paramiko
[emea@intel175 paramiko]$ ll
total 3968
-rw-r--r--. 1 root root  101571 May  2 18:04 asn1crypto-0.24.0-py2.py3-none-any.whl
-rw-r--r--. 1 root root   57401 May  2 18:04 bcrypt-3.1.4-cp27-cp27mu-manylinux1_x86_64.whl
-rw-r--r--. 1 root root  407338 May  2 18:04 cffi-1.11.5-cp27-cp27mu-manylinux1_x86_64.whl
-rw-r--r--. 1 root root 2161273 May  2 18:04 cryptography-2.2.2-cp27-cp27mu-manylinux1_x86_64.whl
-rw-r--r--. 1 root root   12427 May  2 18:04 enum34-1.1.6-py2-none-any.whl
-rw-r--r--. 1 root root   56450 May  2 18:04 idna-2.6-py2.py3-none-any.whl
-rw-r--r--. 1 root root   18155 May  2 18:04 ipaddress-1.0.22-py2.py3-none-any.whl
-rw-r--r--. 1 root root  194536 May  2 18:04 paramiko-2.4.1-py2.py3-none-any.whl
-rw-r--r--. 1 root root   71642 May  2 18:04 pyasn1-0.4.2-py2.py3-none-any.whl
-rw-r--r--. 1 root root  245897 May  2 18:04 pycparser-2.18.tar.gz
-rw-r--r--. 1 root root  696879 May  2 18:04 PyNaCl-1.2.1-cp27-cp27mu-manylinux1_x86_64.whl
-rw-r--r--. 1 root root   10702 May  2 18:04 six-1.11.0-py2.py3-none-any.whl
tar zcvf paramiko.tar.gz /tmp/paramiko
```

上传paramiko.tar.gz 到离线环境，在离线环境通过pip 安装：

```
tar xzvf paramiko.tar.gz 
pip install --no-index --ignore-installed six --find-links=tmp/paramiko paramiko

[root@adb01 ~]# pip install --no-index --ignore-installed six --find-links=tmp/paramiko paramiko
Looking in links: tmp/paramiko
Collecting six
Collecting paramiko
Collecting pyasn1>=0.1.7 (from paramiko)
Collecting bcrypt>=3.1.3 (from paramiko)
Collecting cryptography>=1.5 (from paramiko)
Collecting pynacl>=1.0.1 (from paramiko)
Collecting cffi>=1.1 (from bcrypt>=3.1.3->paramiko)
Collecting idna>=2.1 (from cryptography>=1.5->paramiko)
Collecting enum34; python_version < "3" (from cryptography>=1.5->paramiko)
Collecting ipaddress; python_version < "3" (from cryptography>=1.5->paramiko)
Collecting asn1crypto>=0.21.0 (from cryptography>=1.5->paramiko)
Collecting pycparser (from cffi>=1.1->bcrypt>=3.1.3->paramiko)
Installing collected packages: six, pyasn1, pycparser, cffi, bcrypt, idna, enum34, ipaddress, asn1crypto, cryptography, pynacl, paramiko
  Running setup.py install for pycparser ... done
Successfully installed asn1crypto-0.24.0 bcrypt-3.1.4 cffi-1.11.5 cryptography-2.2.2 enum34-1.1.6 idna-2.6 ipaddress-1.0.22 paramiko-2.4.1 pyasn1-0.4.2 pycparser-2.18 pynacl-1.2.1 six-1.11.0
You have mail in /var/spool/mail/root
[root@adb01 ~]# python -c 'import paramiko'
[root@adb01 ~]# 
```

从最后的结果可以看到，paramiko已经顺利安装。

这个问题解决之后，我就可以开心的AntDB中引入python的包来扩展运维工具了。

参考链接：
https://pip.pypa.io/en/stable/reference/pip_download/



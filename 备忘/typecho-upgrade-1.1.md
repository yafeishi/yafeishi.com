>升级typecho到1.1
>author: yafeishi
>tags: typecho，vps

typecho已经好几年没有更新了，17年中作者又开始更新了，目前建议的稳定版本是1.1-17.10.30,下载地址：[1.1-17.10.30]( http://typecho.org/downloads/1.1-17.10.30-release.tar.gz)

升级过程很简单，下载压缩包之后，进行解压：

```
tar xzvf 1.1-17.10.30-release.tar.gz
```
移除老版本的系统文件,当然移除前请先备份:

```
rm -rf yafeishi.com/admin yafeishi.com/var yafeishi.com/index.php yafeishi.com/install.php yafeishi.com/install
```

添加新版本的系统文件到网站目录：

```
cp -R build/admin yafeishi.com/admin
cp -R build/var yafeishi.com/var
cp build/index.php yafeishi.com/index.php
cp build/install.php yafeishi.com/install.php
cp -R build/install yafeishi.com/ipnstall
```

替换完成后，重新刷新后台即可。
在后台底部可以看到版本号：
![](https://ws4.sinaimg.cn/large/006tNc79ly1fo4djgksaej30je054gmb.jpg)

参考链接：

* [https://blog.izgq.net/archives/913/](https://blog.izgq.net/archives/913/)
* [http://docs.typecho.org/upgrade](http://docs.typecho.org/upgrade)


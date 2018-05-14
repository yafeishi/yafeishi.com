# git filter-branch 使用
刚好有个需求：
adb_tools.git 通过不同的分支对应了不同的文件夹，现在想把文件夹下面的内容push到adbsql对应的仓库中，
git log只保留子文件夹相关的，应该怎么做？
pro git中提到了方法: [重写历史] (https://www.git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E9%87%8D%E5%86%99%E5%8E%86%E5%8F%B20)
具体到本次需求，解决方法如下：
将一个子目录设置为新的根目录
假设你完成了从另外一个代码控制系统的导入工作，得到了一些没有意义的子目录（trunk, tags等等）。
如果你想让trunk子目录成为每一次提交的新的项目根目录，filter-branch也可以帮你做到：

```
git filter-branch --subdirectory-filter Ora2Pg HEAD
➜  Ora2Pg git:(Ora2Pg) git filter-branch --subdirectory-filter Ora2Pg HEAD
Rewrite 22a5add0f8975a19d935ea00fbd2e0462e649f7f (21/30) (1 seconds passed, remaining 0 predicted)
Ref 'refs/heads/Ora2Pg' was rewritten
子目录已经变成了work目录，切与该目录无关的git log都已经清除。
接下来就是push到github上了

git remote add yafeishi https://github.com/yafeishi/adb-ora2pg.git
git push -u yafeishi Ora2Pg:master


git remote add adbsql https://github.com/adbsql/adb-ora2pg.git
git push -u adbsql master
```


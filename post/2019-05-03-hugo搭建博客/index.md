
在编译时报错如下：
```
Building sites … ERROR 2019/05/03 10:36:45 Failed to read Git log: fatal: Not a git repository (or any of the parent d irectories): .git

Building sites … ERROR 2019/05/03 10:45:16 Failed to read Git log: fatal: your current branch 'master' does not have a ny commits yet
```
![](/img/bugo-error-git1.png)
![](/img/bugo-error-git2.png)
需要先在站点根目录执行以下命令：
```
git init
git add .
git commit -m 'test'
```

config.toml中的baseUrl为http://liabio.github.io 不修改，调试时使用命令行参数指定baseUrl：为 http://localhost:1313/
```
hugo server -D -t even --baseUrl="http://localhost:1313/"
```
![](/img/bugo-error-git3.png)



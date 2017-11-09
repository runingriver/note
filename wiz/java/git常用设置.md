## git配置原理
git配置有两个地方: 1. 位于用户目录`/home/zongzhehu/.gitconfig`; 2. 位于git的仓库目录`.git/config`; 他们是继承关系,类似maven父子工程.
公司gitlab全局配置:
```
[push]
default = simple
[core]
autocrlf = input
quotepath = off
editor = nano
[gui]
encoding = utf-8
[user]
name = zongzhe.hu
email = zongzhe.hu@qunar.com
[https]
[http]
```

## 常见问题解决：
1. git命令；`git command --help`可以查看该命令的使用说明
2. 如果远程分支a和本地分支a1不一致：比如，其他分支a2也提交了修改到远程a：
答：先将远程pull下来，然后查看不同，修改，在提交！
3. `git commit --amend `会进入nano编辑器，修改备注，然后`Ctrl+O`保存，---提示文件名，这里直接Enter，`Ctrl+X`退出！
4. `git commit `和`git add .`不能加入某些文件,提示之前的`commit`操作....去`.git`文件夹中删除`index.lock`文件即可.
5. 每次push都要输入密码，是因为你使用https的方式pull下来的，所以，最好的办法就是去工程目录下.git文件夹下修改`config`文件里面的内容为ssh地址。

## 理解
1. origin什么意思。
答；就是在本地的远程分支的名称。用来代表我要提交到远程的哪个分支！

2. 本地新建的分支和远程分支的区别
答：本地新建分支与远程分支没有关系，如果从远程分支拉取了一个分支到本地（只能拉取master分支），那么这个分支名在本地叫做：`origin\master`，这个时候我们要新建一个分支a1，然后将`origin\master`的内容merge到a1.我们就在a1上开发，到提交的时候，如果：`git push origin a1`,远程没有这个分支，就会在远程创建这样一个分支！
3. 把修改提交远程branch分中，而不是提交到master分支。
4. 输出太长按q，回到命令行，`Ctrl+z`，退出当前命令行任务

## 设置代理
```
#设置,如果下面不能走代理,可以设置:`socks5://127.0.0.1:1080`
git config --global http.proxy http://127.0.0.1:1080
git config --global https.proxy https://127.0.0.1:1080
#取消
git config --global --unset http.proxy
git config --global --unset https.proxy
#详见:https://gist.github.com/laispace/666dd7b27e9116faece6
```
根据git配置原理,可以设置局部代理,即在对应的git项目目录中`.git/config`中加入即可!
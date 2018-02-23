---
title: Java排查工具之jvisualvm,jps,jmap,jstack
date: 2017/03/08 11:12:22
toc: false
list_number: false
categories:
- Java
tags:
- JVM
- 开发工具
---

**前言：**总结java开发中常用到的一些工具，这里都是实际操作命令，很多时候在服务器上操作需要注意的地方，如果有疑问建议加google来看，这里精简描述。

## 1. `jps` 查看jvm中运行的Java进程
参数说明: 
`-q ` 不输出类名、Jar名和传入main方法的参数
`-m` 输出传入main方法的参数
`-l` 输出main类或Jar的全限名
`-v` 输出传入JVM的参数
常用:` sudo jps -l` 和 `sudo jps -v`

## 2. `jmap` jvm内存使用情况(强烈建议使用`jmap -h`查看使用方法)
注: 使用格式中 `[]`表示可选, `<>`表示必选. 比如: `jmap [option] <pid>`中option内容是可选, pid是必选.
1. `sudo jmap -heap <pid>`  查看jvm运行各个区的情况.
2. `sudo jmap -F -histo  <pid>` 打印Java对象堆. `-F`表示强制打印, 不能强制打印`-histo:live`
3.  `jmap -F -dump:format=b,file=dumpFileName pid` dump内存.
如: `sudo jmap -F -dump:format=b,file=./dump.dat 24192` 首先想内存dump到`dump.dat`文件中
然后使用`sudo jhat -port 8998 ./dump.dat`命令分析内存对象. 服务器上执行(运行一个http server),本地浏览器可查看!
注: 如果`dump.dat`文件比较大 , 执行如下命令: `sudo jhat -J-Xmx512m -port 8998 ./dump.dat`

## 3. `jstack` 打印堆栈信息,多用于分析死锁
- `sudo jstack -F <pid>`  打印堆栈
-  `sudo jstack -F -l <pid>` 打印加上锁信息 

## 4. `jstat` 查看jvm每个区域情况
1. `jsta -options` 查看可用参数
2. `sudo jstat -class <pid>1000 3` 显示class加载数量和空间, 间隔1s, 打印3次.
3. `sudo jstat -gcutil <pid> 1000 5`  显示各个分区使用百分比,间隔1s,打印5次.  `-gc` 打印字节数
其中: S0,S1表示Survivor, E表示Eden, O表示Old, P表示Perm; YGC , FGC 表示Minor GC , FULL GC触发次数. YGCT,FGCT分别表是两种GC总时间,GCT表示所有GC总时间.


## 5. `jvisualvm`远程连接,监控`jvm`
1. 首先`jvisualvm` 确保已经安装`Visual GC`插件
2. 服务器创建一个文件`sudo touch visualvm.remote.policy` 文件名任意,添加如下内容:
 ```
grant codebase "file:${java.home}/../lib/tools.jar" {
    permission java.security.AllPermission;
};
 ```
3. 服务器上运行 `sudo jstatd -J-Djava.security.policy=./visualvm.remote.policy -p 9000`  注意要加`sudo`，且注意相对`visualvm.remote.policy`目录．
4. 打开`jvisualvm`,右键`Remote`, 添加`Host Name`主机名, 打开`Advanced Settings`修改端口为`9000`, 设置刷新间隔.
5. ok!


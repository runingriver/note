---
title: Java排查利器之BTrace，Greys
date: 2017/03/08 11:12:22
toc: false
list_number: false
categories:
- Java
tags:
- JVM
- 开发工具
---

**前言：** 关于btrace和greys的使用方法，网上不计其数。这里根据我的使用整理了个精简版，零复制纯手打，高度概括，适用快速入门。萝卜青菜各有所爱，greys好用太多有木有！


# 一. Greys
## 安装
安装非常简单
1. 直接安装：`mkdir greys;cd greys`，`curl -sLk http://ompc.oss.aliyuncs.com/greys/install.sh|bash` 然后运行`greys.sh <pid>`即可！
2. 手动下载安装：`http://ompc.oss.aliyuncs.com/greys/release/greys-stable-bin.zip` 解压：`unzip greys-stable-bin.zip`然后：`cd greys; ./install-local.sh`
如果没有权限，给目录权限即可：`chmod 777 -R greys`

## 使用

### 使用问题
greys出现：
1. `attach to target jvm(11313) failed.`
2. `start greys failed, because : well-known file is not secure`
3. `attach to target jvm(10241) failed.`
4. `permission denied, /home/xxx is not writeable.`
5. `start greys failed, because : Unable to open socket file: target process not responding or HotSpot VM not loaded`
6. `start greys failed, because : Operation not permitted`
等等，以上问题，解决：`sudo -u <app_user> -H ./greys.sh <pid>`
如：`sudo -utomcat -H ./greys.sh 1024`，其中tomcat是jvm运行的用户，1024是进程pid。
也可以这样尝试下：`sudo -H ./greys.sh 1024`

### 远程调试
比如你没有A机器的权限，小伙伴需要你帮忙定位A机器上程序的问题时：
 1. 先在A机器运行`greys.sh 24449`脚本，并执行`jvm`命令，获得`MACHINE-NAME : 24449@xxx.xxx.xxx`
 其中，`24449`表示A机器上应用程序的PID，`xxx.xxx.xxx`表示A机器名。

 2. 本地用greys连接上A机器：`sudo ./greys.sh 24449@xxx.xxx.xxx:3658`
 其中，`3658`是greys默认的通信端口，如果在第一步中指定了特点的端口（如：`./greys.sh 24449@127.0.0.1:2727`），这里命令应该为：`sudo ./greys.sh 24449@xxx.xxx.xxx:2727`
OK 完成 ～！

### 命令使用
首先，可以通过`help`和`help [cmd]`命令来查看帮助，其中`[cmd]`表示greys支持的命令。
另外，终止命令运行`Ctrl + D` !

## 程序排查命令
1. `sc` 查看已经加载到JVM的Class信息
 如：`sc -df com.xx.xx.xx.xxxxService` 不仅输出类的信息，还可以输出成员变量的值。
2. `sm` 查方法信息：`sm -d com.xx.xxx.service.AClass` 查看AClass类下所有方法的定义。
3. monitor监控方法的执行信息，包括执行时间，状态，时间等统计信息。
eg：`monitor -c 5 org.apache.commons.lang.StringUtils is*`监控`StringUtils`类中以`is`开头的所有方法的执行统计信息。
4. `trace` 查看`insert`方法的调用栈，并输出执行时间信息。
eg：`trace *xxxxService insert`，`[2,1ms]`的含义，`2`所代表的含义是：当前节点的整体耗时；`1`的含义是：当前节点在当前步骤的耗时；二者逗号分割，单位`ms`。
5. `ptrace` 是`trace`的增强版，命令比较强大，可以查看`help ptrace`或官方文档
eg：`ptrace -t com.xxx.xx.service.xxxService insert --path=com.xx.xx.service*` 
6. `watch` 观察方法的`入参`，`返回值`，`异常`。
 - `watch -b com.xxx.xxx.service.ConsumerService insertMapToHBase params[0]+"--"+params[1]` 输出入参`params[0]+"--"+params[1]` 参数格式！
 Tip：`target + "--" + clazz +"--" + throwExp + "--" + isThrow`，注意+前后空格隔开。
 -  `watch -s com.xxx.xxx.service.ConsumerService insertMapToHBase returnObj` 输出方法返回值！
 
7.  `tt` 同`watch`，但是`tt`能记录一段时间方法调用的现场，并支持检索调用，方便定位问题。
 1. 记录现场 
`tt -t -n 30 *DataService outboxToHBase returnObj==false`：记录DataService类下outboxToHBase返回值为false的现场，`-n 30`表示记录`30`次现场。
`tt -t -n 30 *DataService* outboxToHBase isThrow==true`；记录DataService类或其内部类,匿名内部类中方法为outboxToHBase的方法,并且抛了异常的调用的现场.
Tip：后面表达式是OGNL，eg：`params[0].mobile=="13989838402"`表示记录第一个参数对象的mobile为13989838402的现场。
Tip：重载方法，通过如下方法指定特定重载方法：`tt -t *Test print 'params[1].class == Integer.class'`，通过指定参数的类型，来判定。
 2. 过滤记录的现场
`tt -l`：列出所有记录的现场概要。
`tt -s isThrow=true`：过滤记录的现场中，抛异常的调用。
 `tt -D`：删除所有历史记录。
 3.  查看现场
 `tt -i 1766` 显示INDEX为1766的方法调用的现场！包含调用栈，方法入参，返回值，异常等信息。
 4. 重新调用现场
 `tt -i 1766 -p ` 重新执行一次1766的这次调用，其中，`1766`是`tt -l` 或`tt -s method.name=="inset"`检索获得的INDEX编号！

8. `stack` 查看方法的调用栈，确定是谁调用了某个方法！

上面，简略说明各个命令使用，最好的方法还是官方文档和`help`命令～

## 其他命令
1. `js`，现有命令可能无法满足需求，需要通过自己编写脚本来实现，`js`命令支持js脚本。
2. `session` 查看当前回话的信息，主要包含：源端口，目的端口，监控的应用PID等。
3. `quit` 仅退出greys命令行，关闭session！
4. `shutdown` 关闭socket，释放端口，重置所有被greys增强的类（即：`reset`）

最后，greys相对与btrace，真的好用太多！

- - -

# 二. BTrace

## 安装
1. `https://github.com/btraceio/btrace/tags` 下载压缩包。
2. 添加环境变量：`sudo nano /etc/profile`
 ```
export BTRACE_HOME=/usr/develop/btrace     
export PATH=$PATH:$BTRACE_HOME/bin
 ```
其中，`BTRACE_HOME`是解压路径
使配置生效`source /etc/profile`，`btrace`命令将可以在任何地方使用。
Tip：这步可以省略，上面配置仅仅保证在任何位置可以使用btrace命令；也可觉得路径方式执行。
如：`sudo -u tomcat /home/q/tools/btrace/bin/btrace 13451 /home/q/tools/btrace/samples/JInfo.java`   

3. btrace命令
`btracec script.java` 编译脚本，查看是否报错！
`sudo -u <user role> btrace <PID> <script.java>` 执行脚本。
Tip：jvisualvm也有BTrace的插件可以安装！

**Btrace参数：**
`-v` 输出详细信息，多用于，执行发生错误，查看错误的具体原因，如：`sudo -u root /home/q/tools/btrace/bin/btrace -v 32257 Memory.java`查看此此运行的详细信息！
`-o <file>`： 指定输出到文件，不再输出到控制台。文件路径默认是程序启动路径，所以**必须要指定为绝对路径**，且指定以后，所有btrace命令的输入都重定向到那里！
`-d <path>`： Dump the instrumented classes to the specified path，指定Btrace脚本编译后的路径，默认编译后的class文件放在当前目录下！

### 遇到的问题
1. 使用报错：`Invalid path 13451 specified: 2/No such file or directory`
首先 `which btrace` 查看btrace命令是否有冲突。
有，则需要安装如下命令执行：`sudo -u tomcat /home/q/tools/btrace/bin/btrace 13451 /home/q/tools/btrace/samples/JInfo.java`
2. `Port 2020 unavailable.`
当服务器上多个服务同时使用Btrace时，端口会发生冲突，BTrace默认使用`2020`端口。
所以，多个服务时，可以指定端口：`sudo -u root /home/q/tools/btrace/bin/btrace -p 6767 32257 JInfo.java`

## 使用
先看一个例子：监控一个指定方法的执行时间！
```
package com.sun.btrace.samples;

import com.sun.btrace.annotations.*;
import static com.sun.btrace.BTraceUtils.*;

@BTrace
public class Method{
    @TLS
    static long beginTime;
    @OnMethod(
            clazz = "com.qunar.sms.web.controller.outer.InterfaceController",
            method = "getNewTemplate"
    )
    public static void traceGetByNameBegin() {
        beginTime = timeMillis();
    }

    @OnMethod(
            clazz = "com.qunar.sms.web.controller.outer.InterfaceController",
            method = "getNewTemplate",
            location = @Location(Kind.RETURN)
    )
    public static void traceGetByNameEnd() {
        println(strcat(strcat("getNewTemplate 耗时:", str(timeMillis() - beginTime)), "ms"));
    }
}
```
注意点：
1. 类名和文件名要一致！
2. `package` 和`import`必须有！

最后，日常问题，根据例子来改进即可，BTrace自带的sample是学习BTrace的最好的资料,可以快速的熟悉BTrace。自带的sample基本能满足大部分需求，在其基础上更改即可。

# 参考
1. https://github.com/oldmanpushcart/greys-anatomy
2. https://github.com/btraceio/btrace

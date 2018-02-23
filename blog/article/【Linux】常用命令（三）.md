---
title: 【Linux】常用命令（三）
date: 2016/08/14 12:12:11
toc: false
list_number: false
categories:
- Linux
tags:
- Linux
---

# 知识点
- `locate a.txt` 搜索文件,其中 `updatedb` 强制更新locate数据库，默认一天更新一次
- 命令前面添加`time` 可以监控命令运行时间,eg:`sudo time lsof -i:9090`

### 常用命令
- 查看cpu核心数：`grep 'model name' /proc/cpuinfo | wc -l`
- 查看所有用户：`cat /etc/passwd |cut -f 1 -d :`
#### 查看文件夹大小
- `du -sh *`   查看当前目录下的文件夹大小
- `du -sh . `  查看当前目录大小
#### 删除swap空间内容
1. `swapoff -a`  关闭swap`/etc/fstab`中设置为swap的分区
2. `swapon -a`  打开`/etc/fstab`中设置为swap的分区
-a  将/etc/fstab文件中所有设置为swap的设备关闭

注：所有的分区记录在： `/etc/fstab`文件中
`swapon -s` 显示交换分区的信息
`mkswap` 创建swap分区

### 根据access 计算QPS
设access日志格式：`[26/Sep/2017:14:59:17 +0800] 10.88.65.39 "POST /shell/hive HTTP/1.1" 200 (6 ms)`，其中`/shell/hive`是接口。
**按分钟统计平均qps：`grep '/shell/hive' access_log.2017-09-26.log | cut -d ":" -f 2,3 |sort|uniq -c| awk '{qps=$1/60;print $2,$1,qps}'`**
按小时统计：`grep '/shell/hive' access_log.2017-09-26.log | cut -d ":" -f 2 |sort|uniq -c| awk '{qps=$1/3600;print $2,$1,qps}'`
按秒统计：`grep '/shell/hive' access_log.2017-09-26.log | cut -d ":" -f 2,3,4 |sort|uniq -c| awk '{print $2,$1}'`
**Tip：如果是`tar.gz`压缩文件，使用`zgrep`**
### 根据access 计算PV
按小时统计PV：`grep '/shell/hive' access_log.2017-09-26.log | cut -d ":" -f 2 |sort|uniq -c| awk '{print $2,$1}'`

### atnodes，fornodes，tonodes多服务器操作
1. `fornodes` 主要提供域名缩写，一般不单独使用，`atnodes`和`tonodes`默认调用！
2. 一般需要在跳板机上执行！如没安装：`sudo cpan SSH::Batch`安装
3. 使用：`atnodes ‘ls’ ‘l-xxx[5-6].wap.cn6’`
4. 配置别名使用：
 `cat > ~/.fornodesrc` 输入 `abc=l-xxx[5-6].xx.xx.xxx.com`
 `atnodes 'ls' '{abc}'`

# 文件查看，编辑
**压缩文件对应：zless、zmore、zcat 和 zgrep **

## 1. linux中查看文件的命令如下：

- `cat`： 由第一行开始显示档案内容
- `tac`： 从最后一行开始显示，可以看出 tac 是 cat 的反向显示！
- `nl`： 显示的时候，随便输出行号！
- `more`： 一页一页的显示档案内容less 与 more 类似，按h可以在输出文件内容的时候查看帮助
- `head`： 查看头几行
- `tail`： 查看尾几行
- `od`： 以二进制的方式读取档案内容

## 2. more的动作指令：

- `Enter`            向下n行，需要定义，默认为1行；
- `Ctrl+f`            向下滚动一屏；
- `空格键 `         向下滚动一屏；
- `Ctrl+b `           返回上一屏；
- `=  `                   输出当前行的行号；

## 3. less动作指令：less比more更强大，命令类似于vi
空格键 向下滚动一屏；
`f` 向下滚动一屏；前移
`b` 向上滚动一屏；后移
`d` 向下滚动半屏；
`u` 向上洋动半屏；
(也可以在`fbdu`前面按`ctrl`)
在显示内容的时候可以输入：-N显示行号；/或者？向前或者向后搜索（按n搜索下一个，N上一个）！
`G`文件末尾（shift+g），`gg`文件开头
**注: ** less在查看一些不规律文件时可能会提示`may be a binary file.  See it anyway`, 比如,`gc.log `, `xx.gz`等. 可以**`less -f -r file`**来查看!

# wget,scp,nc,

## 1. wget
wget默认占用全部可能的带宽下载！
- `wget -b [url]`  后台下载
- `wget -c [url]` 断点续传，继续一个终止的文件下载。
- `wget --limit-rate=300k [url]` 限速下载 ，`--limit-rate=10M`

实例：
1. `wget http://path.to.the/file -O newname` 下载file用newname命名
2. `wget -P path/to/directory http://path.to.the/file` 将文件下载到指定目录

### wget下载整个目录
- 下载整个目录：`wget -c -r -np -k -L -p www.xxx.com/pub/path/ `
- 下载包含外部域名的资源：`wget -np -nH -r –span-hosts www.xxx.com/pub/path/`
参数：
`-c` 断点续传 
`-r` 递归下载，下载指定网页某一目录下（包括子目录）的所有文件 
`-nd` 递归下载时不创建一层一层的目录，把所有的文件下载到当前目录 
`-np` 递归下载时不搜索上层目录，如wget -c -r www.xxx.com/pub/path/ ，如不加参数-np，就会同时下载path的上一级目录pub下的其它文件 
`-k` 将绝对链接转为相对链接，下载整个站点后脱机浏览网页，最好加上这个参数 
`-L` 递归时不进入其它主机，如wget -c -r www.xxx.com/ 会递归下载所有相关主机资源！
`-p` 下载网页所需的所有文件，如图片等 
`-A` 指定要下载的文件样式列表，多个样式用逗号分隔
`-i` 后面跟一个文件，文件内指明要下载的URL
### wget 下载指定格式的文件
` wget -r -np -nd --accept=txt http://example.com/dir/` 下载`dir`目录下所有以`txt`结尾的文件，多个后缀用逗号分割！
### wget镜像网站
`wget -m -k (-H) http://www.example.com/`， 如果网站中的图像是放在另外的站点，那么可以使用` -H` 选项。
### wget使用代理下载
`wget -Y on -p -k https://xxx.net/projects/wvware/` 
**代理可以在`环境变量`或`wgetrc`文件中设定:** 
1. 在环境变量中设定代理 
`export PROXY=http://211.90.168.94:8080/ `
2. 在~/.wgetrc中设定代理 
`http_proxy = http://proxy.xxx.com:18023/ `
`ftp_proxy = http://proxy.xxx.com:18023/ `

## 2. scp
语法：`scp [参数] [被传输的文件地址路径] [文件传输目的地路径]`
参数:
`-v`:详细方式显示输出
`-r`:递归复制整个目录
1. 上传，本地上传远程:
格式： `scp foo.txt user@example.com:remote/dir`，其中user@example.com可以改为IP
例：`scp -v ./file.txt xxusername@l-xxx.xxx.xx.xx.com:/home/xxusername` 其中，`/home/xxusername`表示远程路径
2. 下载，远程下载本地:
格式： `scp user@example.com:remote/dir/foo.txt local/dir`，其中user@example.com可以改为IP
例： `scp -v ./file.txt xxusername@100.80.xx8.xx7:/tmp`，在服务器上操作，将文件传到本地主机


## 3. nc
前言：nc是一个可以通过TCP或UDP进行数据传输，可用来**传输文件**，**聊天**，**端口扫描**，**远程执行命令**。

### nc基本参数
`-l` 使用监听模式，管控传入的资料。
`-n` 直接使用IP地址，而不通过域名服务器。
`-p<local port>` 设置本地主机使用的通信端口，通常可以这样`nc -l 1234`直接指定
`-s<destination port>` 设置本地主机送出数据包的IP地址，可以`nc 10.86.219.204 1234`直接指定IP。
`-u` 使用UDP传输协议。
`-v` 显示指令执行过程。
`-w<timeout seconds>` 设置等待连线的时间。

### 文件传输
机器A(Ip:`192.90.123.112`)向机器B(IP:`192.90.123.110`)传输文件`file.txt`
1. B: `nc -l 1567 >file.txt`，`-l`开启监听模式，`1567` Tcp端口号，输出到`file.txt`
2. A: `nc 192.90.123.110 1567 < file.txt`。

Tip：A，B两个机器任何一个监听都可以，保证流向即可！

**等价：**
1. B: `nc 192.90.123.112 1567 >file.txt`
2. A：`nc -l 1567 <file.txt`

### 其他作用
1. 端口扫描：`nc -vv <ip> 1025-65535`，扫描对应ip的端口是否开放！
2. 执行远程命令： `echo stat | nc <ip> 2181`,远程执行zk命令。
3. 聊天：A：`nc -l 1567`，B：`nc <A ip> 1567`

---
title: Linux大杂烩
date: 2016/08/13 19:22:11
toc: true
list_number: false
categories:
- Linux
tags:
- Linux
---

前言：下面总结使用Linux类系统的一些总结。

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

# vim
## 1. 模式切换
### (1) 普通模式
其他模式`2次ESC`即可进入普通模式
- 删除：
**`x` 删除光标位置字符**
**`dd`删除当前行（以后按.就可以直接删）**
`d)`删除从光标开始到句末，`d(`删除当前光标到句首，`d{`段落后删除，`d}`段落前删除
`dw，db`也是删除单词，`d0`光标到行首删除
- 常用操作
前进后退：`u`：撤销上一次操作（后退），`Ctrl+r`：撤销（重做，前进）
查找：`/`向下，`?`向上查找，n匹配后一个，N匹配前一个 
替换：`：%s/内容/新内容/`
复制：`yy` 复制光标所在行的所有字符 
粘贴：`p` 将最后一个删除或复制文本放在当前字符之后 ，`P` 光标字符之前
`.`  重复执行上一个命令
#### 光标的移动
- 上下左右：`h`，`j`，`k`，`l`
- `w`上一单词，`b` 下一单词；`e`光标单词末尾；
- `G`文件末尾，`gg`文件开头
- `H`屏幕左上角，`M`屏幕中间，`L`屏幕最后一行
- `Ctrl + f`,`Ctrl + b`；前后滚动一页；`u`，`d`前后滚动半页
- `0`当前行的行首，`shift+a`调到行尾并编辑，`$`调到行尾。
- `{}`段前段尾；`（）`句首句尾

### (2) 编辑模式
命令模式下输入以下命令，即进入编辑模式，两次ESC退出该模式！
`i`，当前位置插入；`shift + i`，当前行首插入
`a`，后一个位置插入；`shift + a` 行尾追加
`o`，光标下一行新建一行开始编辑，`shift + o`，光标上一行新建一行开始编辑；

### (3) 可视模式
命令模式下输入以下命令，即进入可视模式，两次ESC退出该模式！
`v` 进入字符可视化模式
`shift+v` 进入行可视化模式（按照行选取一块）
`Ctrl+v` 进入块可视化模式（按照列选取内容）
选择指定文本后，可以输入命令模式下的命令，对指定文件进行操作！
eg：删除(`x`)，剪切(`d`)，复制(`y`)
eg：移动文本：首先v模式下选定文本，`d`剪切，`p（贴在光标后）或P（贴在光标前）`粘贴
eg：选定的文本输出到文件：v模式下，选定文本，输入`:write file.txt`
eg：选定的文本排序：v模式下，选定文本，输入`:sort`

## 2. 输入命令：
`:5m3`：把第五行移动到第四行
`:2,5m0`  把第2-5行移动到文件头
`:<` `:>`  光标所在行移动4个空格，`:<<`  `:>>`  8个空格

### 常用操作
`back`或`2次ESC`：进入普通模式
保存退出`：wq` `shift zz`
强制退出：`:q!`
`set nu`  设置行号
`set tabstop=4` tab 长度设置为 4
`set ai` 设置自动缩进
`set all` 查看所有选项值
`split`或`sp`：窗口切成上下两部分，`ctrl+w`上下切换。`q`关闭，`vsplit`左右切分（`vs`简写命令）

## 3. 安装
1. `sudo apt-get install vim-gtk`
2. `sudo vim /etc/vim/vimrc` 修改配置文件
3.在末尾添加：
```
set nu                           // 在左侧行号
set tabstop                  //tab 长度设置为 4
set nobackup               //覆盖文件时不备份
set cursorline               //突出显示当前行
set ruler                       //在右下角显示光标位置的状态行
set autoindent             //自动缩进
```
插件安装：
`wget -qO- https://raw.github.com/ma6174/vim/master/setup.sh | sh -x`
1. 按F5可以直接编译并执行C、C++、java代码以及执行shell脚本，按“F8”可进行C、C++代码的调试
2. 自动插入文件头 ，新建C、C++源文件时自动插入表头：包括文件名、作者、联系方式、建立时间等，读者可根据需求自行更改
3. 映射“Ctrl + A”为全选并复制快捷键，方便复制代码
4. 按“F2”可以直接消除代码中的空行
5. “F3”可列出当前目录文件，打开树状文件目录
8. 按“Ctrl + P”可自动补全

# AWK
## 语法：
### 基本语法
`awk [选项参数] 'script' var=value file(s)  ` 或  `awk [选项参数] -f scriptfile var=value file(s)   执行awk脚本`
### awk脚本
三个部分:
 1. BEGIN{ 这里面放的是执行前的语句 }
 2. END {这里面放的是处理完所有的行后要执行的语句 }
 3. {这里面放的是处理每一行时要执行的语句}；可以没有BEGIN和END，这个一定有
## 用法：
### 1. 基本使用：
-  `awk '{print $1,$4}' log.txt`  每行按空格或TAB分割，输出文本中的1、4项  
### 2. 指定分隔符:
`awk -F`  `-F`相当于内置变量`FS`, 指定分割字符，多个字符分隔`-F'[a b c]'`  abc三个分隔符间以空格分开,`-F'\t'`  以tab作为分隔符
- `awk -F, '{print $1,$2}' log.txt`使用","分割 
 `awk 'BEGIN{FS=","} {print $1,$2}' log.txt` 使用内建变量 
- `awk -F '[ ,]' '{print $1,$2,$5}' log.txt`  使用多个分隔符,  先使用空格分割，然后对分割结果再使用`,`分割
- `awk -F '[, :]' '{print $1,$2,$3}'`  使用`,`和`:`分隔 
### 3. 设置变量:
- ` awk -va=1 '{print $1,$1+a}' log.txt`   其中`awk -v ` 设置变量 
### 4. 过滤:
-  `awk '$1>2' log.txt ` 过滤第一列大于2的行 
- `awk '$1==2 {print $1,$3}' log.txt`   过滤第一列等于2的行  
### 5. 正则：
-  `awk '$2 ~ /th/ {print $2,$4}' log.txt`   输出第二列包含 "th"，并打印第二列与第四列     `~ 表示模式开始。// 中是模式`
-  `awk '/re/ ' log.txt`   输出包含"re" 的行 
-   `awk 'BEGIN{IGNORECASE=1} /this/' log.txt`忽略大小写
- 模式取反
`awk '$2 !~ /th/ {print $2,$4}' log.txt`
`awk '!/th/ {print $2,$4}' log.txt`
## 脚本
```shell
#!/bin/awk -f
#运行前
BEGIN {
    math = 0
    english = 0
    computer = 0
    printf "NAME    NO.   MATH  ENGLISH  COMPUTER   TOTAL\n"
    printf "---------------------------------------------\n"
}
#运行中
{
    math+=$3
    english+=$4
    computer+=$5
    printf "%-6s %-6s %4d %8d %8d %8d\n", $1, $2, $3,$4,$5, $3+$4+$5
}
#运行后
END {
    printf "---------------------------------------------\n"
    printf "  TOTAL:%10d %8d %8d \n", math, english, computer
    printf "AVERAGE:%10.2f %8.2f %8.2f\n", math/NR, english/NR, computer/NR
}
```

- 计算文件大小
` ls -l *.txt | awk '{sum+=$6} END {print sum}'`
- 从文件中找出长度大于80的行
`awk 'lenght>80' log.txt`
- 分析access.log获得访问前10位的ip地址
`awk '{print $1}' access.log |sort|uniq -c|sort -nr|head -10`
- 文本文件中第三列的所有数字求和：`awk '{ x += $3 } END { print x }'`
- 统计netstat各个状态：`netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'`

# find,grep

## 1. find 最常见和最强大的查找命令，你可以用它找到任何你想找的文件
`find <指定目录> <指定条件> <指定动作>`
最常使用：
```
#语法
find <指令目录> <指令条件> <指定动作>

#搜索当前目录中，所有文件名以my开头的文件，并显示它们的详细信息
find . -name 'my*' -ls

#当前目录及其子目录下查找lee.log这支文件
find . -name 'lee.log'

#将build.log中所有包含error字段的行列出来并显示行号
grep -n 'error' build.log

find /home/admin -size +250000k(超过250000k的文件，当然+改成-就是小于了)
find /home/admin -atime -1  1天内访问过的文件
find /home/admin -ctime -1  1天内状态改变过的文件 
find /home/admin -mtime -1  1天内修改过的文件
find /home/admin -amin -1  1分钟内访问过的文件
find /home/admin -cmin -1  1分钟内状态改变过的文件   
find /home/admin -mmin -1  1分钟内修改过的文件
```

- ` find /home/q/www/ -maxdepth 2 -name logs` 文件名，深度查找

- find 常用参数
`-print`，find命令将匹配的文件输出到标准输出
`-exec/-ok`，find命令对匹配的文件执行该参数所给出的shell命令。相应命令的形式为`'command' {} \;`，注意`{}`和`\;`之间的空
`-ok`删除文件前,要询问是否删除!, 用`-exec` 执行指令是每次操作生产一个线程, 用`xargs`只在一个shell中处理.
eg: 删除文件大小为零的文件 方法一: `find ./ -size 0 -exec rm {} \;` 方法二: `rm -i $(find ./ -size 0)`  方法三: `find ./ -size 0 | xargs rm -f &` 
`find /etc -type d –print` 在/etc目录下查找所有的目录 
`find . ! -type d –print` 在当前目录下查找除目录以外的所有类型的文件 
`find ./ -mtime +3 -print|xargs rm -f –r` 删除3天以前的所有东西 （`find . -ctime +3 -exec rm -rf {} \;`）
`find . -type f -mtime +7 -name "*.log" -exec mv {} /tmp \;` 将7天前的`log`文件移动到`/tmp`目录下，`\;`表示命令结束。
`find ./ -size 0 | xargs rm -f &` 删除文件大小为零的文件

## 2. grep
- 查找当前目录下所有文件包含'text'的文件及行数： `grep -rn 'text' .` 其中` r`表示递归, `n`表示显示匹配字符在文件中的行!
- 只显示匹配的字符：`grep -o 'type.*groupid'` 只显示以`type`开头，以`groupid`结尾，中间任意个字符的这部分。
- AND与：grep 'parttern1' | grep 'parttern2'
- OR或：`egrep 'pattern1|pattern2' filename` 等价于  `grep -E 'pattern1|pattern2' filename`，也可以： `grep -e pattern1 -e pattern2 filename`
- NOT非：`grep -v 'parttern'`，即不包含。
- 范围输出：`grep -A 5 `，`grep -B 5`，`grep -C 5`分别是匹配行的后5行，前5行，前后各5行。
- `egrep` 正则匹配，`zgrep` 查看压缩文件

### find与grep，sed组合，查询文件中的内容
- `find . | xargs grep 'pattern'` 查找当前目录及子目录下所有文件中包含`pattern`的字符行。
- `find . -name 'abc*' | xargs grep 'pattern'` 查找当前目录及子目录中文件名以`abc`开头的，文件中包含`pattern`的字符行。
**Tip:压缩文件`tar.gz`请用`zgrep`**
- `find . -name '*.txt' | xargs sed -i "s/233/666/g;s/235/626/g;s/333/616/g;s/233/664/g"` 查找当前及子目录所有txt文件，并全文替换，多种类型字符串替换！


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

# Top,pmap,iostat,dstat,sar,pidstat,vmstat

### 1. top系统监控
**列含义：**
> **VIRT:** 虚拟内存中含有共享库、共享内存、栈、堆，所有已申请的总内存空间。(进程地址空间没有映射之前的内存)
**RES: ** 是进程正在使用的内存空间(栈、堆)，申请内存后该内存段已被重新赋值。
**SHR: ** 是共享内存正在使用的空间。
**SWAP:** 交换的是已经申请，但没有使用的空间，包括(栈、堆、共享内存)。
**DATA:** 是进程栈、堆申请的总空间。
**CODE:**	可执行代码占用的物理内存大小，单位kb
**PR:	**优先级
**NI:**	nice值。负值表示高优先级，正值表示低优先级

**交互：**
- `c` 或`top -c`：显示启动参数
- `z`：高亮显示，再按`x`高亮当前排序的列，继续`shift + <`或`shift + >` 左右选择排序的列。
- `1` ： 展开显示CPU每个核的使用情况。
- `shift + p`即`P`：按CPU排序
- `shift + m`即`M`：按内存排序
- **f** 键可以选择显示的内容
- **h** 查看帮助
- **k** 再输入pid，kill一个进程
- **`top -p pid`** 只监控一个进程 ` top -bH -d 3 -p  <pid>` 监控进程的线程运行, 或者按H，也可`top -Hp <pid>` 查看PID下最耗CPU的线程pid.

**字段含义：**
`uptime`：系统负载；先看15分钟的。1.7表示有70%的进程在等待。
`进程统计`：Tasks: 215 total,   1 running, 213 sleeping,   0 stopped,   1 zombie：zombie表示僵尸进程。
`cpu使用率`：%Cpu(s): 27.9 us使用率,  4.4 sy内核使用率,  0.0 ni, 67.8 id空闲,  0.0 wa等待IOcpu占比，比较高，则表示cpu多半在等待IO,  0.0 hi,  0.0 si,  0.0 st

### 2. pmap 查看进程空间
`sudo pmap -x pid`  `sudo pmap -q pid` 简略方式查看进程空间
**列含义：**
> **Address**:  start address of map  映像起始地址
**Kbytes: ** size of map in kilobytes  映像大小
**RSS**:  resident set size in kilobytes  驻留集大小**(真是存在与RAM中的部分),其他分布与swap或文件系统,或可执行部分没有加载**
**Dirty:**  dirty pages (both shared and private) in kilobytes  脏页大小
**Mode:**  permissions on map 映像权限: r=read, w=write, x=execute, s=shared, p=private (copy on write)  
**Mapping:**  file backing the map , or '[ anon ]' for allocated memory, or '[ stack ]' for the program stack.  映像支持文件,[anon]为已分配内存 [stack]为程序堆栈
**Offset:**  offset into the file  文件偏移
**Device:**  device name (major:minor)  设备名

### 3. iostat 磁盘IO和CPU监控统计
- **`iostat -c 2 5`（等价于`sar -u 2 5`），统计cpu使用情况**
`%user`：CPU处在用户模式下的时间百分比。
`%nice`：CPU处在带NICE值的用户模式下的时间百分比。
`%system`：CPU处在系统模式下的时间百分比。
`%iowait`：CPU等待输入输出完成时间的百分比。
`%steal`：管理程序维护另一个虚拟处理器时，虚拟CPU的无意识等待时间百分比。
`%idle`：CPU空闲时间百分比。
备注：如果`%iowait`的值过高，表示硬盘存在I/O瓶颈，`%idle`值高，表示CPU较空闲，如果`%idle`值高但系统响应慢时，有可能是CPU等待分配内存，此时应加大内存容量。
`%idle`值如果持续低于10，那么系统的CPU处理能力相对较低，表明系统中最需要解决的资源是CPU。

- **`iostat -x 2 5` （等价于`sar -d 2 5`）或`iostat -x -k 2 5` 统计磁盘IO情况**
`rrqm/s`: 每秒对该设备的读请求被合并次数，文件系统会对读取同块(block)的请求进行合并
`wrqm/s`: 每秒对该设备的写请求被合并次数
`r/s`:  每秒完成的读 I/O 次数。即 rio/s
`w/s`:  每秒完成的写 I/O 次数。即 wio/s
`rsec/s`:  每秒读扇区数。即 rsect/s
`wsec/s`:  每秒写扇区数。即 wsect/s
`rkB/s`:  每秒读K字节数。是 rsect/s 的一半，因为每扇区大小为512字节。
`wkB/s`:  每秒写K字节数。是 wsect/s 的一半。
`avgrq-sz`:  平均每次IO操作的数据大小(扇区数为单位)
`avgqu-sz`:  平均等待处理的IO请求队列长度
`await`:  平均每次设备I/O操作的等待时间 (包括等待时间和处理时间，毫秒为单位)。
`svctm`: 平均每次IO请求的处理时间(毫秒)，已不具备参考价值
`%util`:  一秒中有百分之多少的时间用于 I/O 操作，即被io消耗的cpu百分比
备注：如果 `%util` 接近 100%，说明产生的I/O请求太多，I/O系统已经满负荷，该磁盘可能存在瓶颈。


上述指令：`iostat -x -k 2 5`，`-k`表示将按扇区统计，改为可读的方式：按KB统计。
1. 每秒44次读IO，1次写IO。
2. 每秒读1MB左右数据，写入6kb数据。
3. 等到IO队列`avgqu-sz`，中每秒有0.26请求驻留，说明IO较快。
4. 每次IO平均处理时间是5.48ms。

- `iostat -d -k 2 5`，`-k`可换成`-m`，分别表示IO的KB，MB；统计磁盘IO


#### vmstat系统监控
- `vmstat 2` 每隔2秒采集一次数据。
- `vmstat 2 5` 每隔2秒采集一次数据，总共采样5次。

**列含义：**
**free**   空闲的物理内存的大小
**cache** 用来记忆我们打开的文件,给文件做缓冲
**in** 每秒CPU的中断次数，包括时间中断
**cs** 每秒上下文切换次数
**us** 用户CPU时间
**sy** 系统CPU时间

### 4. dstat系统监控
该命令和`vmstat`相同，提供更直观的彩色输出，但是需要我们先安装：`sudo yum install dstat`；`sudo apt-get install dstat`
- `dstat -dnyc -N eth0 -C total -f 5`
- `dstat -taf --debug`
- `dstat -tcndylp --top-cpu`
- `dstat --time --cpu --net --disk --sys --load --proc --top-cpu`
- `dstat -tcyif`
- **`dstat -tvn 5`，使用最多！！！**
显示信息说明:
cpu：hiq、siq分别为硬中断和软中断次数。 
system：int、csw分别为系统的中断次数（interrupt）和上下文切换（context switch）。


### 5. pidstat （用于监控全部或指定进程占用系统资源的情况）
- `pidstat -w 1 2` 每1秒采样一次，共采样2次，获取进程上下文切换情况：
cswch 自愿的上下文切换，当某一任务处于阻塞等待时，将主动让出自己的CPU资源。
nvcswch 非自愿的上下文切换，时间片到，更高优先级抢占！
`pidstat -w -p <pid> 1` 每秒采样一次，统计指定进程上下文切换情况！
- `pidstat -d 1`  每1秒采样一次，统计IO使用情况
- `pidstat -l` 列出所有进程名和它的启动参数！
- `pidstat -u -p <pid> 2` 2秒采样一次，查看指定进程CPU使用情况。

### 6. sar 系统历史情况统计
1. `sar -q` 查看运行队列中的进程数、系统上的进程大小、平均负载
`runq-sz`：运行队列的长度（等待运行的进程数）
`plist-sz`：进程列表中进程（processes）和线程（threads）的数量
`ldavg-1`：最后1分钟的系统平均负载 ldavg-5：过去5分钟的系统平均负载
`ldavg-15`：过去15分钟的系统平均负载

2. `sar -r` 查看内存使用状况
`kbmemfree`：这个值和free命令中的free值基本一致,所以它不包括buffer和cache的空间.
`kbmemused`：这个值和free命令中的used值基本一致,所以它包括buffer和cache的空间.
**`%memused`：物理内存使用率，这个值是kbmemused和内存总量(不包括swap)的一个百分比.**
`kbbuffers和kbcached`：这两个值就是free命令中的buffer和cache.
`kbcommit`：保证当前系统所需要的内存,即为了确保不溢出而需要的内存(RAM+swap).
`%commit`：这个值是kbcommit与内存总量(包括swap)的一个百分比.

3. `sar -W`查看页面交换发生状况
页面发生交换时，服务器的吞吐量会大幅下降；服务器状况不良时，如果怀疑因为内存不足而导致了页面交换的发生，可以使用这个命令来确认是否发生了大量的交换；
`pswpin/s`：每秒系统换入的交换页面（swap page）数量
`pswpout/s`：每秒系统换出的交换页面（swap page）数量
4. `sar -u` 查看CPU使用率
`%user` 用户模式下消耗的CPU时间的比例；
`%nice` 通过nice改变了进程调度优先级的进程，在用户模式下消耗的CPU时间的比例
`%system` 系统模式下消耗的CPU时间的比例；
**`%iowait` CPU等待磁盘I/O导致空闲状态消耗的时间比例；iowait比例太高说明性能瓶颈在IO上**
`%steal` 利用Xen等操作系统虚拟化技术，等待其它虚拟CPU计算占用的时间比例；
`%idle` CPU空闲时间比例；


**Tip：要判断系统瓶颈问题，有时需几个 sar 命令选项结合起来：**
怀疑CPU存在瓶颈，可用 `sar -u` 和 `sar -q` 等来查看
怀疑内存存在瓶颈，可用`sar -B`、`sar -r` 和 `sar -W` 等来查看
怀疑I/O存在瓶颈，可用 `sar -b`、`sar -u` 和 `sar -d` 等来查看



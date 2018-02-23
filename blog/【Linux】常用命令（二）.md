---
title: 【Linux】常用命令（二）
date: 2016/08/13 22:12:11
toc: false
list_number: false
categories:
- Linux
tags:
- Linux
---

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
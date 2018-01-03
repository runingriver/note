---
title: cron，at，batch(任务调度)
date: 2016/11/12 11:12:22
toc: true
list_number: false
categories:
- Linux
tags:
- Linux
---

## 一，基础知识
1. **corntab**周期执行任务,**anacron**调度每日、每周或每月执行的作业,**at**调度只运行一次的作业.
2. batch 命令被用来在系统平均负载达到 0.8% 以下时执行一次性的任务，通俗的所就是系统空闲时执行任务。
3. crontab调度的任务是不会加载任何环境变量的,这就是为什么shell中可以执行,cron中不行的原因.
4. crond守护进程每分钟唤醒一次，并检查 crontab 判断需要执行什么任务. 
    crond守护进程是在系统启动时由 init 进程启动的(重启不影响任务的运行)。

## 二，命令
### 1. 常用crontab命令
-u: 只有root 才能执行这个任务，也即帮其他用户新建/删除 crontab 工作调度: `sudo crontab -e -u zongzhe.hu`
-e: 编辑crontab 的工作内容
-l : 查阅crontab的工作内容
-r: 删除所有的crontab的工作内容，若仅要删除一项，请用 -e 去编辑
### 2. ubuntu下启动、停止与重启cron:
`$sudo /etc/init.d/cron start`
`$sudo /etc/init.d/cron stop`
`$sudo /etc/init.d/cron restart` 当任务失效时可以通过restart解决
### 3. 表达式
`* * * * * command` 分 小时 几号 月份 星期几
在以上各个字段中，还可以使用以下特殊字符：
- 星号`*`：代表所有可能的值
- 逗号`,`：可以用逗号隔开的值指定一个列表范围，例如，“1,2,5,7,8,9”
- 中杠`-`：可以用整数之间的中杠表示一个整数范围，例如“2-6”表示“2,3,4,5,6”
- 正斜线`/`：可以用正斜线指定时间的间隔频率，例如“0-23/2”表示每两小时执行一次。
     同时正斜线可以和星号一起使用，例如*/10，如果用在minute字段，表示每十分钟执行一次。

### 4. 知识点
-  crond检测的时间周期是“分钟”， 每分钟会读取一次` /etc/crontab`， 以及 /var/spool/cron 里面的用户crontab记录并执行。
- 用户执行计划保存在: `/var/spool/cron/username`中,username是cron的用户名
- 系统执行计划保存在: `/etc/crontab`
- `/var/log/cron`中保存这cron的执行日志
### 5. 示例
 `3,15 8-11 * * * command`  在上午8点到11点的第3和第15分钟执行
 `3,15 8-11 */2 * * command`  每隔两天的上午8点到11点的第3和第15分钟执行
 `10 1 * * 6,0 /etc/init.d/smb restart`  每周六、周日的1 : 10重启smb
 `45 4 1,10,22 * * /etc/init.d/smb restart` 每月1、10、22日的4 : 45重启smb
 `0 10 10 ? * 2`    每个星期一，10点10分 执行

#### 指定用户执行
- 命令: 
    查看test用户下的cron任务: `sudo crontab -l -u test`
    编辑test用户下的crontab任务: `sudo crontab -e -u test`
- 执行hive导数为示例:
 1. 将shell设置为test用户文件: `sudo chown test run.sh` , `sudo chgrp test run.sh ` 
 2. 添加任务: `sudo crontab -e -u test` , `10 6 * * * /bin/bash /home/q/xxx/run.sh > /home/q/xxx/log2 2>&1 &`
#### 注意
1. 脚本中涉及文件路径时写全局路径(因为cron执行shell的当前目录不是shell文件所在目录)；
2. 每条cron之前可以加一个注释,功能,时间,用户!
3. **执行脚本前一定要检查脚本的woner和执行权限** ,  确保crontab创建者有**访问目录**和**执行shell**的权限
4. 脚本执行要用到java或其他环境变量时，通过source命令引入环境变量，如:
```
!/bin/sh
source /etc/profile #只要是cron任务,都加上吧
export RUN_CONF=/home/d139/conf/platform/cbp/cbp_jboss.conf
/usr/local/jboss-4.0.5/bin/run.sh -c mev &
```
或 `0 * * * * . /etc/profile;/bin/sh /var/bin/restart_audit.sh`

5. 多个脚本互斥执行,可以用进程锁`flock`解决
```
*/1 * * * * /usr/bin/flock -xn /var/run/test.lock -c /home/q/test/test.sh >>/home/q/test/test.log 2>&1
```
注: `test.lock`文件需要自己手动创建并给予权限.还要注意锁的抢占，可能某个shell总是拿到锁的控制权.

**问题:**
如果在crontab里有个定时任务设置为一分钟执行一次，但是它执行的时间可能会超过一分钟，此时crontab一分钟后会再次运行该脚本，这样就会出现冲突，
如果程序不做容错处理，可能会导致出现一些问题。如果想解决这个问题，可以用Linux中的进程锁控制crontab执行的并发问题。
解决：在crontab设置lock
`*/5 * * * * /usr/bin/flock -xn /var/run/test.lock -c '/scripts/test.sh >/dev/null 2>&1'`
Tip：`flock -h` 查看使用,`test.lock`文件需要手动创建并给予权限,多个cron任务,注意锁的抢占!

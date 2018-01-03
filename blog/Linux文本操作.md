---
title: Linux文本操作命令
date: 2016/11/12 11:12:22
toc: true
list_number: false
categories:
- Linux
tags:
- Linux
---


## 1. tr 按列替换
- `echo "{123}" | tr -d '{}'` 删除输入中的 "{" 和 "}"
- `cat testfile |tr a-z A-Z`  将文件testfile中的小写字母全部转换成大写字母

## 2. cut : cut [-b] [file] 列切分
cut 命令从文件的每一行剪切字节、字符和字段并将这些字节、字符和字段写至标准输出。
如果不指定 File 参数，cut 命令将读取标准输入。必须指定 -b、-c 或 -f 标志之一。
- `echo "123,456" | cut -d "," -f 1` 以","分隔截取输入中的每行的第一部分内容输出.
- `who | cut -b 3` 以字节为分隔, 输出每行的第三个字节. `-c` 是以字符为分隔
- `count=$(echo -e "${line}" | cut -f 2)` 其中,`line`中字符是以tab作为分隔符!
- `echo $RANDOM | cut -c1-4`，取随机数的前4位，一个生产随机数的方法。
### join
将两个文件中相同行的内容连接起来
`file1`文件内容：
```
qwe 123
asd 456
```
`file2`文件内容：
```
qwe 111
asd 222
```
`join file1 file2` 输出：
```
qwe 123 111
asd 456 222
```
1. `join -e HHH file1 file2` ：如果file1或file2中找不到指定栏位，则填入`HHH`
2. `join -i -t | HHH file1 file2`：匹配相同栏位时忽略大小写，并以`|`作为分隔符。
3. `-a`,`-v`,`-1`,`-2`：都是用来决定没有匹配行时，是否输出不匹配的行！

## 3. chown, chgrp, chmod 权限
`sudo chgrp root *` 修改当前目录下所有文件为root组
`sudo chown root *` 修改当前目录下所有文件的owner为root
`sudo chmod 777 -R dir` 递归修改dir的权限为777
`sudo chmod 755  file.sh` 修改file.sh为任何人可执行权限

## 4. sudo
- `sudo -l` 查看当前用户运行命令权限
- `sudo -u <user> command` 使用user用户执行命令command

## 5. uniq 去重
`uniq -c file` 在每行的旁边增加重复的数量。

## 6. nl 输出前加行号
- `nl file`在输出的内容前加行号
- `nl -b a file` 遇到空行，也加行号。
- `nl -n rz -w 3 file` 行号3位对其，前面补0.
## 7. shuf 打乱文件顺序
- `shuf sort_file -o rsort_file` 
## 8. split 将文件切分
- `split -5000 file`或`split -l 5000 file` 将file按行切分成多个文件, 文件最大行为5000
- `split -5000 -d file` 以数字作为后缀,默认:`xaa,xab,xac`, 现在`x00,x01,x02`
- `split file.txt -b 10M` 将文件file.txt平均切分成10M
### 合并文件
- `cat x* > file.txt` 将以`x`开头的所有文件合并到file.txt`中
- `cat file1 file2 > file3` 将file1和file2合并保存到file3中. 
 **文件内容顺序, 按照file1+file2的顺序保存到file3**
### 模式切分`csplit`
将文本文件file以第 2 行为分界点切割成两份，命令： `csplit testfile 2`
### 文件求交，差，补
`cat a b | sort | uniq > c`   # c 是 a 并 b
`cat a b | sort | uniq -d > c`   # c 是 a 交 b
`cat a b b | sort | uniq -u > c`   # c 是 a - b
## 9. `sed`按行操作文本（大文本操作）
**大文本数据修改，编辑，保存，不能用编辑器打开，可以借助`sed`对大文件进行修改**
sed编辑行以1为起始index！
## 详解
- `-e` 多次编辑
eg：`nl file | sed -e '3,$d' -e 's/bash/blueshell/'` 删除第三行到最后一行，然后将1-2行中匹配`bash`的字符串替换成`blueshell`字符串
eg：`sed -e 4a\newLineContent file` 在第四行后天添加一行内容`newLineContent`
- `-n` 仅显示script处理后的结果。
eg：`nl /etc/passwd | sed -n '5,7p'` 仅列出文件的5-7行。
eg：`nl /etc/passwd | sed -n '/root/p'` 仅列出匹配root的行
- `-i`  直接编辑源文件**危险动作**
 * 替换：`sed -i 's/\.$/\!/g' file`  将file的最后一行中的`.`替换成`!`
 * 最后一行插入：`sed -i '$ a\This is a test' file` 在最后一行，再添加一行内容：`This is a test' file` `$`表示最后一行，`a`表示在后面添加。
 * 第一行插入：`sed -i '1 i\This is a test' file`   其中`1`表示第一行，`i`表示在第一行的前面添加。
 * 删除多行：`sed -i '1,4564d' file` 删除从1～4564行，包括第4564行，index从1开始。

`-e` 接的动作：
- `a` ：新增， 在当前行的下一行添加，是新的一行
- `i` ：插入， 在当前行的上一行插入，是新的一行
- `c` ：取代 
- `d` ：删除， 后面没有内容；
eg：`nl file | sed '2,5d'` 删除第二到五行！
eg：`sed '2d' file` 删除第二行
eg：`sed '/My/,/You/d' datafile` 删除包含"My"的行到包含"You"的行之间的行
eg：`sed '/My/,10d' datafile` 删除包含"My"的行到第十行的内容
eg：`nl file | sed -n '/root/p'` 删除所有行中包含root的行！
- `p` ：打印列。通常 p 与参数 sed -n 一起用
eg：只查看文件的第100行到第200行 `sed -n '100,200p' a.log`
- `s` ：替换，/要被取代的字串/新的字串/g

### 使用
1. 查找行号  `grep -n --color '您的司机账户已被冻结' outbox.csv`
2. 删除对应行号保存： `sed -e 5d file1 > ./file2` 删除第五行 并保存到当前目录下的file2文件中。
3. 删除匹配项： `cat file1 | sed '/hello/d' > ./file2` 删除所有行中包含`hello`字符串的行保存。
 `nl file1 | sed '/hello/d' > ./file2` 在每行内容前加一个行号，保存到文件中！
4. `nl /etc/passwd | sed -n '/bash/{s/bash/blueshell/;p;q}' ` 首先匹配所有`bash`行，然后执行`{}`里面的一组动作，替换bash为blueshell，p打印，q退出！

## 10. 只输出一行中匹配的字符串.
语法: `grep -o 'regex'`
- `less file* | grep type | grep -o 'user\[.*\]user_id' | grep -o '\[.*\]' | sort | uniq` 以`file`开头的所有文件中,每行包含`type`的字符串,提取字符串中以`user[.*]user_id`形式存在`[]`中的内容!

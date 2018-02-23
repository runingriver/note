---
title: 【Linux】常用命令（一）
date: 2016/08/13 19:22:11
toc: false
list_number: false
categories:
- Linux
tags:
- Linux
---

前言：下面总结使用Linux类系统的一些总结。

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







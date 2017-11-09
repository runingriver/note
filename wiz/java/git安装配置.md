## Ubuntu git安装配置
用命令`git --version`查看是否已安装，且版本为1.9.5或更高。若没安装或版本太低：
`sudo apt-get install git-core git-gui git-doc gitk`
再用`git --version`查一下，如果安装的不是1.9.5版本，那是不是你的ubuntu太老了？试试下面的方法：
```
udo add-apt-repository ppa:git-core/ppa
sudo apt-get update
sudo apt-get install git
```
`add-apt-repository` 是由`python-software-properties` 这个工具包提供的，如果使用` add-apt-repository`显示`“command not found”`需要安装`python-software-properties`
安装方法：
1.首先需要安装software-properties-common
`sudo apt-get install software-properties-common`
2.然后安装python-software-properties
` sudo apt-get install python-software-properties`
检查：`git --version` 

## 公司git配置
使用全局配置：
```
git config --global user.name "zongzhe.hu"            
git config --global user.email "zongzhe.hu@qunar.com" 
git config --global push.default simple               
git config --global core.autocrlf false               
git config --global gui.encoding utf-8               
git config --global core.quotepath off                
```
ok，完成！

Tip: 如果要链接github，则需要在github仓库中单独执行
`git config user.name "runingriver"`
`git config user.email "huzongzhe@126.com"`
将用户名，邮箱设置为github的用户名邮箱。

### 生成ssh key pair
`ssh-keygen -t rsa -C "zongzhe.hu@qunar.com"`
然后一路回车，不要输入任何密码之类，生成ssh key pair。
如果在Linux上，需要把其中的私钥告诉本地系统：`ssh-add ~/.ssh/id_rsa`
再把其中公钥的内容复制到GitLab上。具体方法是：`cat ~/.ssh/id_rsa.pub`
Tip：`.ssh`目录可以备份，升级Ubuntu系统后可以直接拷贝到用户目录！
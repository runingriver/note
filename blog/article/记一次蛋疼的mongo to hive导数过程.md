---
title: 记一次蛋疼的mongo to hive导数过程
date: 2017/04/13 11:12:22
toc: false
list_number: false
categories:
- shell
tags:
- shell
- mongo
---

## 1. 起因
一次hive查数过程中，发现hive中缺省了10天的近3000w的数据，自问自答：怎么办，当然是要补数啊！从哪里补，mongo啊（还好mongo中有一份）！
mongo中数据是bson保存，而且数据列与hive不一样！

## 2. 解决方案
#### 方案1：`mongoexport`
思路：由于`mongoexport`只能以逗号分割字段，所以要导到hive里面最快的方式就是，利用mysql可以导逗号的cvs文件，还可以指定列，并且约束严格可以方便的检查数据正确性。
所以，第一反应是`mongo to cvs to mysql to hive`，但是很快就失败了，过程还是要记录下来的！
**第一步**：`mongo to cvs`
语句：`sudo ./mongoexport -hxxx --port xxx -u xxx -pxxx -d sms -c outbox1 --type=csv -f id,type,mobile, -q '{optime:{$gte: "2017-02-19 05:40:00", $lte: "2017-02-20 05:40:00"}}' -o /home/q/temp_mongo/mongo_data.cvs`
**第二步**：`cvs to mysql`
语句：
```
LOAD DATA LOCAL INFILE '/home/xxx/xx00' INTO TABLE xxxtable FIELDS TERMINATED BY ',' LINES TERMINATED BY '\n';
```
插曲：`mongo_data.cvs`数据太大60多G，采用`split`切分`split -50000 mongo_data.cvs`,每个文件5w行切分，第一次测试就先切1000行吧：`csplit /mongo_data.cvs 1000`先将文件切成了2份！

#### 问题
1. `mongoexport`对字符串不加`"`导致字段中包含逗号`,`，导致导入失败！（知道为啥不直接到hive了吧，导数过程肯定有问题啊，mysql解决问题多方便快捷）

2. `mongoexport`对于`\`不会转义，所以字符串中出现`\[汉字]`，`eg : \请...`形式的字符，mysql无法识别。
报错： `[HY000][1300] Invalid utf8 character string: 'xxx'` 

**最简单的方法要解决这些问题太玛法，迅速放弃，期待`mongoexport`更智能点吧，找其他快速解决的办法！**

#### 方案2：`mongo shell`
 `mongoexport`不能解决问题，借助shell也许是最快的办法了。
思路：`mongo shell to  cvs to hive`
**第一步**： 新建脚本 `export.js`

```
db.auth("xxx","xxx");
conn = new Mongo();
db = conn.getDB("xxxdb");
var cur = db.xxxdb.find({optime:{$gte: "2017-02-19 05:40:00", $lte: "2017-02-20 05:40:00"}});
var obj;
while(cur.hasNext()){
    obj = cur.next();
    print(obj.id+"\t"+ ... +"\t"+obj.subaccount+"\n");
}
```
tip： 和在命令行语法差不多，可以随意指定输出格式！这样就可以直接一步到hive了

**第二步**： 使用`mongo`执行`cd .../mongodb/bin`目录下的mongo脚本，`./mongo --help`查看帮助
`sudo ./mongo xxxip:30000/xxdb -u xxx -p xxx export.js > /home/q/temp_mongo/outbox`
注：`export.js`放在当前目录，所以没有路径！并且要删除outbox前两行输出: `sed -i '1,2d' outbox`
**第三步**：导hive
```
#!/usr/bin/env bash
source /etc/profile
eval cd $(dirname $0)
currentDir=$(pwd)

line="xxx"
_HIVE_TABLE=xxxdb
PATH_FILE="${currentDir}/xxx"

gzip ${PATH_FILE}
PATH_GZ="${PATH_FILE}.gz"
echo "PATH_GZ:${PATH_GZ}"
hive -e "set mapreduce.job.name = ${0}_xxx;USE wirelessdata; \
alter table ${_HIVE_TABLE} add IF NOT EXISTS PARTITION(num='${line}'); \
LOAD DATA LOCAL INPATH '${PATH_GZ}' OVERWRITE INTO TABLE ${_HIVE_TABLE} partition(num=${line});" || exit 1
rm -f ${PATH_FILE}
rm -f ${PATH_GZ}
echo "end success."
```
搞定，也算比较快的方式吧- -！
## 3. 总结
我想只有坑踩多了，才会成长吧！你将从本文获取如下知识点：
1. 使用`mongoexport`导出mongo数据。
2. 使用shell脚本**个性化**导出mongo数据。
3. cvs导mysql，字符串中特殊字符的问题。
4. cvs导hive的脚本基本知识。
5. `mysql，hive，mongo`之间数据导入导出方法。
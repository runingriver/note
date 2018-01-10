# 2015fresh1说明
## 文本还原
现有2个文件，地址在：
sdxl_prop.txt
sdxl_template.txt
要求：
根据sdxl_prop.txt中内容替换掉sdxl_template.txt 里$function(index)形式文字，将其还原成一本完整小说，写到文件sdxl.txt中，输出在classpath下。
其中function有4种函数，替换规则如下：
1) natureOrder 自然排序，即文本中排列顺序
2) indexOrder 索引排序，文本中每行第一个数字为索引
3) charOrder 文本排序，java的字符排序
4) charOrderDESC 文本倒序，java的字符倒序
注意：txt文件需要程序中现读现解析，而不是下载成本地文件。
结果文件可以参考附件中的sdxl.txt
## 统计Java源文件代码行数
统计附件中的StringUtils.java文件的有效代码行数。
1) 有效不包括空行、注释
2) 考虑代码里有多行注释的情况
3) 不用考虑代码和注释混合在一行的情况
请把StringUtils.java放到classpath下；统计结果输出到validLineCount.txt中，只输出行数。

## 聊天记录排序还原
题目：
有一段被打乱的聊天记录，保存在文件chat.txt 中。请将这段聊天记录按时间正序排序，发言时间相同的按号码正序排序，保存到文件chat_sorted.txt中。
同时统计每个人这段时间的发言的次数，以“号码:次数”（冒号为英文冒号，中间无空格）的格式保存到count.txt中，保存顺序按号码正序排列。
输出的文件都放在项目根目录下。
补充说明：
1.一次发言的内容可以有多行；
2.昵称可以随意命名，没有任何限制，且可以随时修改；
3.聊天记录中的昵称为发言时的昵称，并非导出聊天记录时的昵称（即为snapshot）；
4.不考虑发言内容包含抬头（如“2016-04-27 10:15:13 雪娇-培训(491235110)”）的情况；
5.号码长度最长为12位。

chat.txt文件内容示例如下（请放到项目resource目录下）：
```
2016-04-27 10:15:13 雪娇-培训(491235110)
问下大家
Java作业都是用的IDEA吧
```

内容说明：
时间：2016-04-27 10:15:13
昵称：雪娇-培训
号码：491235110
发言内容：
问下大家
Java作业都是用的IDEA吧

扩展（时间还有富裕的同学选做，新建模块，命名为exam1.1）：
1.输出结果时使用发言人最后的昵称，如发言人先后用了昵称A、B和C，排序后保存的文本中都使用昵称C；
2.将每个人使用过的昵称按时间序给出，格式为：“号码:昵称A,昵称B,昵称C”（冒号为英文冒号，逗号为英文逗号，中间无空格），保存到nickname.txt中，保存顺序按号码正序排列。

## Linux命令实现
linux下有很多对文本进行操作的命令，比如cat filename 可以将文件的所有内容输出到控制台上。grep keyword filename可以将文件内容中包含keyword的行内容输出到控制台上。
wc -l filename可以统计filename文件中的行数。
|是代表管道的意思，管道左边命令的结果作为管道右边命令的输入，比如cat filename | grep exception | wc -l，可以用来统计一个文件中exception出现的行数。
请实现一个功能，可以解析一个linux命令（只包含上面三个命令和上面提到的参数以及和管道一起完成的组合，其它情况不考虑），并将结果输出到控制台上，比如下面这几个例子：
```
cat xx.txt
cat xx.txt | grep xml
wc -l xx.txt
cat xx.txt | grep xml | wc -l
```

要求
1. 作业提交到自己 git 分支的下, 包名为 exam2, 测试类为 Test.java
2. 正确使用日志, 日志不要打文件日志, 只需要输出到控制台
3. 不得使用 `e.printStackTrace()` 以及 `System.out.println()`

## 文本diff功能
题目功能
• 上传两个文本文件（文件 < 1k）
• 对比差异
• 存储差异结果到数据库
• 展示历史对比记录
• 登陆后可删除历史对比记录

对比规则
文件内容均为普通键值对，如下两个文件a和b
```
a1=x
a2=y
a3=z
```
和:
```
a1=x
a3=x
a4=n
```
分如下3种情况输出结果:
• 源存在、目标不存在 => `-a2=y`
• 源不存在、目标存在 => `+a4=n`
• 源和目标存在、但不同 => `-a3=z;+a3=x`

## 字符统计
统计：
1) 查找一个目录下(路径为运行时传入参数)，所有文件中数字、字母(大小写不区分)、汉字、空格的个数、行数。
2) 将结果数据写入到文件characterType.txt中。
文件格式如下：
数字：198213个
字母：18231个
汉字：1238123个
空格：823145个
行数：99812行
数字0：123个
数字1：1214个
数字2：23423个
......
字母A：754456个
字母B：7567个
字母C：456456个
......
文件格式中分两种，总概要信息和详细统计信息，详细信息只需要统计数字和字母，区分大小写。
同样，输出的文件放在classpath下

## 日志数据统计
从 access.log 中统计数据
0. 使用 Java&Guava
1. 输出请求总量
2. 输出请求最频繁的10个 HTTP 接口，及其相应的请求数量
3. 统计 POST 和 GET 请求量分别为多少
4. URI 格式均为 /AAA/BBB 或者 /AAA/BBB/CCC 格式，按 AAA 分类，输出各个类别下 URI 都有哪些

## 对象解析
1 有一个(任意)对象，里面有N个properties以及getter和setter方法
2 有一个properties文件，有N个key,value来描述对象中property的值
3 有一个scheme固定的xml，用来描述这个对象 
要求写一个解析器：
1 将xml中的占位符，替换为properties文件中的value
2 将xml解析成对象，调用getter方法的时候可以获得值
3 用面向对象的思想，使该解析器有扩展性 
例子见附件，注意：
1 对象是任意对象，不是例子中的Student，对象中的property都是java中的原生类型
2 xml和properties在使用的时候都是根据对象配置好的
3 xml的scheme是固定的，就是附件中的scheme
```
<object class="com.qunar.fresh.Student">
	<property name="name">
		<value>${name}</value>
	</property>
	<property name="age">
		<value>${age}</value>
	</property>
	<property name="birth">
		<value>${birth}</value>
	</property>
</object>
```
```
name=a
age=10
birth=2003-11-04
```
```
public class Student {

    private String name;

    private int age;

    private Date birth;
    ...
```



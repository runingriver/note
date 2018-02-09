---
title: 【HBase】一些有必要知道的知识点（二）
date: 2017/06/07 11:12:22
toc: false
list_number: false
categories:
- HBase
tags:
- HBase
---

### 9. HTable，HConnection，HConnectionManager三者之间的关系
1. `HConnection`是client和HBase集群交互的封装,它知道怎么找到HBase的master,region在集群中的哪个RS;
内部通过缓存保存region的相关信息(保存ROOT表,META表),并且通过与ZK通信知道master,region移动后的重新校准!
`HConnection`不是一个和server的网络连接,而是管理所有与HBase集群(包括master,region server)的网络连接!
它由`HConnectionManager`创建,被`HTable`调用!并且其实现类`HConnectionImplementation`是在`HConnectionManager`中!

2. `HConnectionManager`是HBase client的核心,它负责引入配置和环境(如:zk的地址,maxKeyValueSize,caching size,WriteBufferSize等),并创建`HConnection`和`HTable`对象,
HConnectionManager中提供一个线程池(pool name类似`hconnection-0x114e7639-shared--pool1-t190`)负责处理业务提交过来的操作!
默认配置:`coreThreads=maxThreads=256`个线程;`workQueue=LinkedBlockingQueue(maxThreads*100)`

3. `HTable`用于与一张指定的HBase table交互,一个非线程安全的类,官方建议使用时创建用完然后丢掉,因为他认为创建一个对象很便宜,且底层是共用连接和线程池;

### 10. SingleColumnValueExcludeFilter和SingleColumnValueFilter的区别
`SingleColumnValueExcludeFilter` : 列值过滤,当返回的数据中不包含过滤的列,用此方法,效率更高!
`SingleColumnValueFilter` : 列值过滤,当返回的数据中包含过滤的列,用此方法
**Tip: 二者都是列值过滤,不同点仅在于返回的数据中是否包含过滤的列!**

### 11. bloomFilter在HBase中的应用

get的过程，首先通过ROOT和META表找到Region，一个Region下有1个Memstroe和N个StoreFile（HFile等价于StoreFile）；
Memstroe和每个StoreFile在都对应一个BloomFilter，每个StoreFile还在内存中有一个LRUBlockCache，缓存数据块（默认64KB）；
通过StoreFile的BloomFilter可以确定该StoreFile是否存在对应的RowKey，这样就可以过滤掉一些StoreFile，减少部分IO；
然后,确定StoreFile后，先去内存的LRUBlockCache中找，如果没有再去找HDFS要。

> BloomFilter是一个列族（cf）级别的配置属性，如果在表中设置了BloomFilter，那么HBase会在生成StoreFile时，包含一份BloomFilter结构的数据，称其为MetaBlock；
> MetaBlock与DataBlock（真实的KeyValue数据）一起由LRUBlockCache维护，所以开启BloomFilter会有一定的存储及内存cache开销。

bloomFilter在HBase中的应用有两种：
1. 基于ROW（仅rowkey）的过滤；建表时指定：`{NAME=>'cf',BLOOMFILTER=>'ROW'}`
    > 列限定符级布隆过滤器检查行和列限定符组合是**否不存在**！

2. 基于ROW+CF（rowkey加column Family）；建表时指定：`{NAME=>'cf',BLOOMFILTER=>'ROWCOL'}`
    > 行级布隆过滤器在数据块里检查特定行键是**否不存在**！

Tip：建表时`BLOOMFILTER`参数默认NONE。

两种布隆过滤器举例：
1. 根据rowkey来过滤storefile的布隆过滤器。 
举例：假设有2个storefile文件sf1和sf2 
sf1包含`kv1（r1 cf:q1 v）`、`kv2（r2 cf:q1 v）`
sf2包含`kv3（r3 cf:q1 v）`、`kv4（r4 cf:q1 v）`
如果设置了CF属性中的bloomfilter为`ROW`，那么`get(r1)`时就会过滤掉sf2（不扫描sf2），同理get(r3)就会过滤掉sf1

**Tip：只在get方式生效，因为scan没有确定的rowkey！**

2. 根据rowkey+qualifier来过滤storefile。
举例：假设有2个`storefile`文件sf1和sf2
sf1包含`kv1（r1 cf:q1 v）`、`kv2（r2 cf:q1 v）` 
sf2包含`kv3（r1 cf:q2 v）`、`kv4（r2 cf:q2 v）`

如果设置了CF属性中的bloomfilter为`ROWCOL`，那么`get(r1,q1)`就会过滤掉`sf2`，`get(r1,q2)`就会过滤掉`sf1`
而，这里如果设置了CF属性中的bloomfilter为`ROW`， 无论`get(r1,q1)`还是`get(r1,q2)`，都会读取`sf1+sf2`； 

**Tip：get和scan都生效，只要指定了cf！**

### 12. HFile中数据块的大小，对读性能的影响
答：一个HFile文件由多个数据块和一个Trailer组成，数据块由一个header和多个k-v的数据组成（这也从存储形式解释了HBase中所有存储区间都是**前闭后开**）；
Trailer则是HFile中所有数据的索引。默认数据块大小是64KB，可在建表时通过：`{NAME=>'cf',BLOCKSIZE=>'65536'}`设置块大小。
数据块越小，相同数据量下数据块就会越多，相应数据块中的header就越多，占用存储也越多，这样加载到内存的数据块就会小些，所以随机查询性能更好。
数据块越大，则加载到内存的数据越多，随之顺序读的性能就会更好。

### 13. 如果建表时使用snappy压缩，压缩解压是在哪个维度进行的，如果改变编码老数据怎么处理，需要删除表吗？
答：snappy只在数据存盘时是压缩的，也即数据落盘是压缩，查询是解压；在内存里（MemStore或BlockCache）或通过网络传输时都是没有压缩的！
编码设置安装列族设置，无需删表重建，此后合并生产HFile时自动使用新压缩方法，所有老HFile仍是旧压缩方法，直到重新被拿出来合并为新的HFile后才会是新的压缩算法。

### 14. rowkey不定长查询：如果scan范围：100200～100300；那么1002001，会在查询结果中吗？
答：会在查询结果中，因为其安装字典顺序进行比较的！已测试验证！



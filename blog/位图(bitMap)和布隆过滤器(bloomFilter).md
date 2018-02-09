---
title: 位图(bitMap)和布隆过滤器(bloomFilter)
date: 2017/08/13 19:22:11
toc: false
list_number: false
categories:
- 数据结构
tags:
- bitMap
- bloomFilter
---


# 一. 位图（BitMap）
位图，一种数据结构，数组中的每一位（bit）存放某种状态，每一位对应一个整数下标表示位置。
> 是用一个数组中的每个数据的每个二进制位表示一个数是否存在。1表示存在，0表示不存在

通常用一个`long[]`数组作为bitMap的存储结构；如果要保存2个以上的状态，可以使用2bit对应数组的一个下标，也叫`2-bitMap`
位图容量：1long=8byte，1byte=8bit，1long=64bit；512M=2^32=42亿个数据。

优点：时间空间效率非常优秀
缺点：可读性差
应用：1.很大的数据排序; 2.很大的数据中判断重复; 3.很大的数据中判断是否存在

# 二. 布隆过滤器（BloomFilter）
布隆过滤器，它可以说是位图的一种拓展，也是用位数组作为存储结构，一个元素通过K个Hash函数将这个元素映射到位数组中的K个点上，并置1；
检索时只要看这些点是不是都是1就知道该元素是否存在了（可能存在，因为这K个点可能被其他的元素hash置1了）；如果任何一个位上是0，则元素一定不存在！

参考：http://colobu.com/2014/08/13/How-to-use-bloom-filter-to-build-a-large-in-memory-cache-in-Java/

优点：查找时间复杂度为`O(n)`，n为元素hash算法的个数；时间空间非常优秀；能将40亿的数据内存由16GB变成500MB（16*1024/4*8=512M）
缺点：不支持删除；hash冲突可能存在误判；
应用：
1. 爬网站，判断一个网站是否爬过；
2. 缓存地名，判断输入是否是一个地名（或是否存在该地名）；
3. HBase的get读取和scan+cf的读取，都有用到bloomFilter

### bloomFilter在HBase中的应用
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

### Guava中BloomFilter的使用
```
        BloomFilter<String> stringBloom = BloomFilter.create(new Funnel<String>() {
            @Override
            public void funnel(String from, PrimitiveSink into) {
                into.putString(from, Charsets.UTF_8);
            }
        }, 1024 * 1024 * 1024 * 4, 0.0001);
```
参数：
1. `expectedInsertions`：预估要插入的数据量
2. `fpp`：错误率，默认3%，即5个hash函数。

### 拓展
BloomFilter需要确定的几个参数及其计算方式：
n: 期望插入的数量
p: 期望错误率
m: 数组总共的位数: `m = − n*lnp / (ln⁡2)^2`
k: 最优hash函数个数：  `k= (m/n) * ln2`

hash函数计算方式：`hi(x)=h1(x)+ih2(x)`
```
long hash64 = …; 
int hash1 = (int) hash64; 
int hash2 = (int) (hash64 >>> 32);
for (int i = 1; i <= numHashFunctions; i++) {
    int nextHash = hash1 + i * hash2;
}
```

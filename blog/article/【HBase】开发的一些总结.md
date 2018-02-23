---
title: 【HBase】开发的一些总结
date: 2017/06/04 11:12:22
toc: false
list_number: false
categories:
- HBase
tags:
- HBase
---

## 一. shell查询

1. 基于列的精确查找
列中`value=861304mQc8851`记录: `scan 'hbase_db:hbase_table', FILTER=>"ValueFilter(=,'binary:861304mQc8851')"`
2. 基于列的模糊查找
列中`value rlike 135`的记录:  `scan 'hbase_db:hbase_table', FILTER=>"ValueFilter(=,'substring:135')"`
3. 指定列的模糊查找
```
mobile rlike 135`的记录 : `scan 'hbase_db:hbase_table', FILTER=>"ColumnPrefixFilter('mobile') AND ValueFilter(=,'substring:135')"
```
Tip:多个列用:`MultipleColumnPrefixFilter (‘Col1’, ‘Col2’)`
4. 多条件查询
```
scan 'hbase_db:hbase_table', FILTER=>"ColumnPrefixFilter('mobile') AND ( ValueFilter(=,'substring:135') OR ValueFilter(=,'substring:159') )"
```
5. QualifierFilter基于列的查询:Qualifier表示列
```
scan 'hbase_db:hbase_table', FILTER => "PrefixFilter ('20170713') AND (QualifierFilter (=, 'binary:mobile'))"
```

- - - 

6. **rowkey前缀范围查询**
`scan 'hbase_db:hbase_table', FILTER => "PrefixFilter ('20170714')"`
`scan 'hbase_db:hbase_table', {COLUMNS => ['cf'], LIMIT => 10, STARTROW => '2017072986159tIkc0922'}`
注: 范围查询表示从该点开始的row,往后面查,往往会有很多结果,所以一定要加limit!
7. `start Rowkey~StopRowkey`查询,值的设置很有讲究,否则查询不到记录:
`scan 'hbase_db:hbase_table', {STARTROW=>'20170713', STOPROW=>'2017071386189k7TT269'}`
8. 以rowkey前缀开始,前缀中包含xx的记录:
```
scan 'hbase_db:hbase_table', {STARTROW=>'20170713', FILTER => "PrefixFilter ('2017071386188Cyac3878')"}
```

9. rowkey模糊查询
```
scan 'hbase_db:hbase_table', {FILTER => org.apache.hadoop.hbase.filter.RowFilter.new(org.apache.hadoop.hbase.filter.CompareFilter::CompareOp.valueOf('EQUAL'), org.apache.hadoop.hbase.filter.SubstringComparator.new('861879Lff1782'))}
```
10. rowkey正则查询:
```
scan 'hbase_db:hbase_table', {FILTER => org.apache.hadoop.hbase.filter.RowFilter.new(org.apache.hadoop.hbase.filter.CompareFilter::CompareOp.valueOf('EQUAL'),org.apache.hadoop.hbase.filter.RegexStringComparator.new('^20170713'))}
```

- - - 

11. 基于**行**的过滤器
两行记录: `PageFilter (2)`
```
scan 'hbase_db:hbase_table', FILTER => "PrefixFilter ('20170713') AND PageFilter (2)"`
等价于:  `scan 'hbase_db:hbase_table', FILTER => "PrefixFilter ('20170713')",LIMIT => 2
```
12. 基于**列**的过滤器
显示4列数据: `limit 4:ColumnCountGetFilter (4)`
`scan 'hbase_db:hbase_table', FILTER => "PrefixFilter ('20170713') AND ColumnCountGetFilter (4)"`
显示每行中从第1**列**开始的2**列**数据: `limit 1,2:ColumnPaginationFilter (2, 1)`

```
scan 'hbase_db:hbase_table', FILTER => "PrefixFilter ('2017071386189k7TT2690') AND ColumnPaginationFilter (2,1)"
```

注：如果结果集是N行M列，那么就是N*M个数据，`ColumnPaginationFilter(2,1)`表示从NxM个数据中的第一个数据开始取2个数据！`PageFilter(2)`，表示取2行数据，即2*M个数据，`ColumnCountGetFilter(2)`，表示从N*M个数据数据中取2个数据。
12. `get <namespace:table> <rowkey>` 或 `t.get <rowkey>`

## 二. Hbase开发一些点

1. HTable是一个非线程安全的类，官方建议每个操作都去new一个HTable对象，因为其创建成本较低。个人测试写了一个HTable对象池，将HTable对象封装并缓存起来，直观影响就是在大量读写时gc相对要较少，执行时间有略微提升。
2. 基于row的`SubstringComparator`和R`egexStringComparator`在HBase表中具有大量数据时，以这两个两件来做过滤，完全不可用；不过我们可以在给定`starKey和endKey`的前提下使用，让其不去全表rowkey执行扫描。
Tip：同理基于column Family的Key-value扫描同样，需要先通过`starKey和endKey`等的row约束下缩小范围，然后进行过滤。
3. 对于较复杂的过滤，我们可以将符合条件数据全部查询出来，然后在client端进行过滤，但是得保证数据不能太大！





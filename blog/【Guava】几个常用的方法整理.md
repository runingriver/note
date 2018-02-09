---
title: 【Guava】几个常用的方法整理
date: 2016/08/14 19:22:11
toc: true
list_number: false
categories:
- Guava
tags:
- Guava
---


# Predicate和Predicates——集合过滤与转换
## Predicate

断言的最基本应用就是过滤集合。所有Guava过滤方法都返回”视图”,即并非用一个新的集合表示过滤，而只是基于原集合的视图。
Predicates提供多个断言：`and，or，not`
List的过滤视图被省略了，因为不能有效地支持类似`get(int)`的操作。改用`Lists.newArrayList(Collections2.filter(list, predicate))`做拷贝过滤。
对Set的转换操作被省略了，因为不能有效支持`contains(Object)`操作懒视图实际上不会全部计算转换后的Set元素，因此不能高效地支持`contains(Object)`。改用`Sets.newHashSet(Collections2.transform(set, function))`进行拷贝转换。
过滤方法：
```
Iterables.filter(Iterable, Predicate)FluentIterable.filter(Predicate)
Iterators.filter(Iterator, Predicate)
Collections2.filter(Collection, Predicate)
Sets.filter(Set, Predicate)
Sets.filter(SortedSet, Predicate)
Maps.filterKeys(Map, Predicate)Maps.filterValues(Map, Predicate)Maps.filterEntries(Map, Predicate)
Maps.filterKeys(SortedMap, Predicate)Maps.filterValues(SortedMap, Predicate)Maps.filterEntries(SortedMap, Predicate)
Multimaps.filterEntries(Multimap, Predicate)
```
用Predicate过滤Iterable的工具：
```
//是否所有元素满足断言？懒实现：如果发现有元素不满足，不会继续迭代
Iterators.all(Iterator, Predicate)FluentIterable.allMatch(Predicate) 
//是否有任意元素满足元素满足断言？懒实现：只会迭代到发现满足的元素	
Iterators.any(Iterator, Predicate)FluentIterable.anyMatch(Predicate) 
Iterators.find(Iterator, Predicate)
Iterables.find(Iterable, Predicate, T default)
//循环并返回一个满足元素满足断言的元素，如果没有则抛出NoSuchElementException
Iterators.find(Iterator, Predicate, T default) 
//返回第一个满足元素满足断言的元素索引值，若没有返回-1
Iterators.indexOf(Iterator, Predicate) 
//移除所有满足元素满足断言的元素，实际调用Iterator.remove()方法
Iterators.removeIf(Iterator, Predicate) 
```

tip：另一种：`Optional<T> tryFind(Iterable, Predicate)` 的返回一个满足元素满足断言的元素，若没有则返回`Optional.absent()`

**集合转换：**
到目前为止，函数编程最常见的用途为转换集合。同样，所有的Guava转换方法也返回原集合的视图。
```
Iterables.transform(Iterable, Function)FluentIterable.transform(Function)
Iterators.transform(Iterator, Function)
Collections2.transform(Collection, Function)
Lists.transform(List, Function)
Maps.transformValues(Map, Function)Maps.transformEntries(Map, EntryTransformer)
Maps.transformValues(SortedMap, Function)Maps.transformEntries(SortedMap, EntryTransformer)
Multimaps.transformValues(Multimap, Function)Multimaps.transformEntries(Multimap, EntryTransformer)
Multimaps.transformValues(ListMultimap, Function)Multimaps.transformEntries(ListMultimap, EntryTransformer)
Tables.transformValues(Table, Function)
```
**组合Function使用：**
```
Iterables.transform(Iterable, Function)FluentIterable.transform(Function)
Iterators.transform(Iterator, Function)
Collections2.transform(Collection, Function)
Lists.transform(List, Function)
Maps.transformValues(Map, Function)Maps.transformEntries(Map, EntryTransformer)
Maps.transformValues(SortedMap, Function)Maps.transformEntries(SortedMap, EntryTransformer)
Multimaps.transformValues(Multimap, Function)Multimaps.transformEntries(Multimap, EntryTransformer)
Multimaps.transformValues(ListMultimap, Function)Multimaps.transformEntries(ListMultimap, EntryTransformer)
Tables.transformValues(Table, Function)
```
tip：`Predicates.compose` 将Predicate和Function进行组合。将Function的结果作为Predicate的输入，然后进行判断过滤操作。

# Function和Functions——对象转换

- `Functions.compose(FunctionA,FunctionB)` 用途：可以对不同的函数进行组合,将FunctionB的输出作为FunctionA的输入进行再处理。可以实现嵌套的数据处理操作。
- `Predicates.compose()` 将Predicate和Function进行组合。将Function的结果作为Predicate的输入，然后进行判断过滤操作。
- **forMap：**
使用map作为`Function<A,B>`也就是用map的value，返回一个以map的key为参数，value为输出的function; 用于对map进行转化，有两个实现方法，在值不存在的情况下，`get(key)`返回null的时候，可以中断或返回null。
- `forPredicate`：使用Predicate作为boolean
- `toStringFunction`将对象集合转成String类型的。
测试实例：
```
//List对象转换
public static void listTransfor() {
    List<String> lowerCast = Splitter.onPattern(" ").splitToList("a age c d");
    List<String> transformUpcast = Lists.transform(lowerCast, new Function<String, String>() {
        @Nullable
        public String apply(@Nullable String input) {
            return input.toUpperCase();
        }
    });
    System.out.println(transformUpcast.toString());
}
```
## 设计精髓：
使用单例模式，解决`toStringFunction`
```
 public static Function<Object, String> toStringFunction() {
  return ToStringFunction.INSTANCE;
}
// enum singleton pattern
private enum ToStringFunction implements Function<Object, String> {
  INSTANCE;
  @Override
  public String apply(Object o) {
    checkNotNull(o);  // eager for GWT.
    return o.toString();
  }
  @Override public String toString() {
    return "toString";
  }
}
```

# Convertor&Supplier

## Convertor
它是一个转换器：非常简单，只需要知道它的使用方法：
```
Double val = Doubles.stringConverter().convert("1.0");
assertEquals(new Double(1), val);
String valAsString = Doubles.stringConverter().reverse().convert(new Double(1)); 
assertEquals("1.0", valAsString);

```

相应的基本类型的转换器有：`Floats，Longs，shorts，Ints`和`Maps`
Maps使用案例：
```
BiMap<String, String> stateCapitals = HashBiMap.create();
   
stateCapitals.put("Wisconsin", "Madison");
stateCapitals.put("Iowa", "Des Moines");
stateCapitals.put("Minnesota", "Saint Paul");
stateCapitals.put("Illinois", "Springfield");
stateCapitals.put("Michigan", "Lansing");
Converter<String, String> converter = Maps.asConverter(stateCapitals);
String state = converter.reverse().convert("Madison");
assertEquals("Wisconsin", state);
```

## Supplier&Suppliers

Supplier接口：这个接口可以提供一个对象通过给定的类型。我们也可以看到通过各种各样的方式来创建对象。
Suppliers类：这个类是Suppliers接口的默认实现类。

接口方法：
```
public interface Supplier<T> {
　　T get();
}
```

Supplier接口可以帮助我们实现几个典型的创建模式。当get方法被调用，我们可以返回相同的实例或者每次调用都返回新的实例。Supplier也可以让你灵活选择是否当get方法调用的时候才创建实例。
Supplier接口的强大之处在于它抽象的复杂性和对象如何需要创建的细节，让开发人员自由地在他觉得任何方式创建一个对象时最好的方法。
接口使用案例：
```
 public class ComposedPredicateSupplier implements Supplier<Predicate<String>> {
　　@Override
　　public Predicate<String> get() {
　　　　City city = new City("Austin,TX","12345",250000, Climate.SUB_ TROPICAL, 45.3);
　　　　State state = new State("Texas","TX", Sets.newHashSet(city), Region.SOUTHWEST);
　　　　City city1 = new City("New York,NY","12345",2000000,Climate.TEMPERATE, 48.7);
　　　　State state1 = new State("New York","NY",Sets.newHashSet(city1), Region.NORTHEAST);
　　　　Map<String,State> stateMap = Maps.newHashMap();
　　　　stateMap.put(state.getCode(),state);
　　　　stateMap.put(state1.getCode(),state1);
      //个Function实例可以通过State的缩写来查找State
　　　　Function<String,State> mf = Functions.forMap(stateMap);
       //使用Predicate实例来评估在那些地方是否有这个State
　　　　return Predicates.compose(new RegionPredicate(), mf);
　　}
}
```

### Suppliers的使用
`Suppliers.memoize`方法返回一个包装了委托实现的Supplier实例。当第一调用get方法，会被调用真实的Supplier实例的get方法。memoize方法返回被包装后的Supplier实例。包装后的Supplier实例会缓存调用返回的结果。后面的调用get方法会返回缓存的实例。
`Suppliers.memoizeWithExpiration`方法与`memoize`方法工作相同，只不过缓存的对象超过了时间就会返回真实Supplier实例get方法返回的值，在给定的时间当中缓存并且返回Supplier包装对象。注意这个实例的缓存不是物理缓存，包装后的Supplier对象当中有真实Supplier对象的值。
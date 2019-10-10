---
layout: post
title: "Guava 用户指南之集合"
date: "2019-09-10"
description: "Guava 用户指南之集合"
tag: [java, guava]
---

#### 不可变集合
使用示例
```Java
public static final ImmutableSet<String> COLOR_NAMES = ImmutableSet.of(
      "red",
      "orange",
      "yellow",
      "green",
      "blue",
      "purple");

class Foo {
   Set<Bar> bars;
   Foo(Set<Bar> bars) {
       this.bars = ImmutableSet.copyOf(bars); // defensive copy!
   }
}
```
##### 为何要使用不可变集合
- 当对象被不可信的库调用时, 不可变形式是安全的
- 不可变对象被多个线程调用时, 不存在竞态条件问题
- 不可变集合不需要考虑变变化, 可以节省时间和空间 (所有的不可变集合都比可变形式有更好的内存利用率)
- 不可变集合因为固定不变, 可以作为常量来使用

创建对象的不可变拷贝是一项很好的防御性编程技巧, Guava 为所有 JDK 标准集合类型和 Guava 新集合类型都提供了简单易用的不可变版本  
JDK 虽然也提供了 Collections.unmodifiableXXX 方法将集合包装为不可变形式, 但有一些缺点
- 笨重而累赘: 不能舒适的用在所有想做防御性拷贝的场景
- 不安全: 要保证没人通过原集合的引用进行修改, 返回的集合才是事实上不可变的
- 低效: 包装过的集合仍然保有可变集合的开销, 如并发修改的检查, 散列表的额外空间等

如果没有修改某个集合的需求, 或者希望集合保持不变时, 防御性的拷贝是个很好的实践; Guava 所有的不可变集合都不接受 NULL 值, 需要包含 NULL 值的集合请使用 Collections.unmodifiableXXX 方法

##### 如何使用不可变集合
不可变集合可以使用如下多种方式创建
- copyOf: ImmutableSet.copy(set)
- of: ImmutableSet.of("a", "b", "c")
- Builder 工具类
```Java
public static final ImmutableSet<Color> GOOGLE_COLORS =
        ImmutableSet.<Color>builder()
            .addAll(WEBSAFE_COLORS)
            .add(new Color(0, 191, 255))
            .build();
```
###### 比想象中智能的 copyOf
ImmutableXXX.copyOf 方法会尝试在安全的是偶避免做拷贝, 例如
```Java
ImmutableSet<String> foobar = ImmutableSet.of("foo", "bar", "baz");
thingamajig(foobar);

void thingamajig(Collection<String> collection) {
    ImmutableList<String> defensiveCopy = ImmutableList.copyOf(collection);
    ...
}
```
ImmutableList.copyOf(cllection) 会智能的直接返回 cllection.asList(), 它是一个 ImmutableSet 的常量时间复杂度的 List 视图; 作为一种探索, ImmutableXXX.copyOf(ImmutableCollection) 会试图对如下情况避免线性拷贝
- 在常量时间内伤会用底层数据结构是可能的 (ImmutableSet.copyOf(ImmutableList) 就不能在常量时间内完成)
- 不会造成内存泄露 (有个很大的不可变集合 ImmutableList<String> hugeList, ImmutableList.copyOf(hugeList.subList(0, 10)) 就会显式地拷贝, 以免不必要地持有 hugeList 的引用)
- 不改变语义 (ImmutableSet.copyOf(myImmutableSortedSet) 会显式地拷贝, 因为和基于比较器的 ImmutableSortedSet 相比, ImmutableSet 对 hashCode() 和 equals 有不同语义)

在可能的情况下避免线性拷贝, 可以最大限度的减少防御性编程带来的开销

###### asList 视图
所有不可变集合都有一个 asList() 方法提供 ImmutableList 视图, 可以用列表形式方便地读取集合元素; 例如可以使用 sortedSet.asList().get(k) 从 ImmutableSortedSet 中读取第 k 个最小元素  
asList() 返回的 ImmutableList 通常是 (并不总是) 开销稳定的视图实现, 而不是简单地把元素拷贝进 List; 也就是说 asList 返回的列表视图通常比一般的列表平均性能更好, 比如在底层集合支持的情况下, 它总是使用高效的contains方法

##### 关联可变集合和不可变集合

| 可变接口集合 | 属于 JDK 还是 Guava | 不可变版本 |
| :--- | :--- |
| Collection | JDK | ImmutableCollection |
| List | JDK | ImmutableList |
| Set | JDK | ImmutableSet |
| SortedSet/NavigableSet | JDK | ImmutableSortedSet |
| Map | JDK | ImmutableMap |
| SortedMap | JDK | ImmutableSortedMap |
| Multiset | Guava | ImmutableMultiset |
| SortedMultiset | Guava | ImmutableSortedMultiset |
| Multimap | Guava | ImmutableMultimap |
| ListMultimap | Guava | ImmutableListMultimap |
| SetMultimap | Guava | ImmutableSetMultimap |
| BiMap | Guava | ImmutableBiMap |
| ClassToInstanceMap | Guava | ImmutableClassToInstanceMap |
| Table | Guava | ImmutableTable |

#### 新集合类型
Guava 引入了很多 JDK 没有的新集合类型, 这些新类型是为了和 JDK 集合框架共存, 而没有网 JDK 集合中硬塞其他概念; 一般的, Guava 集合非常精准的遵循了 JDL 接口契约
##### Multiset
统计一个词在文档总出现的次数, 使用 JDK 集合实现如下
```Java
Map<String, Integer> counts = new HashMap<String, Integer>();
for (String word : words) {
    Integer count = counts.get(word);
    if (count == null) {
        counts.put(word, 1);
    } else {
        counts.put(word, count + 1);
    }
}
```
以上写法相对冗长, 也容易出错, 并且不支持同时收集多种统计信息, 如总词数等  
Guava 提供了一个新的集合类 Multiset (类似数学上集合的概念, Multiset 继承自 Collection 接口而不是 Set 接口), 可以多次添加相等的元素; 维基百科从数学角度这样定义 Multiset："集合 [set] 概念的延伸, 它的元素可以重复出现, 与集合 [set] 相同而与元组 [tuple] 相反的是, Multiset元素的顺序是无关紧要的: Multiset {a, a, b}和{a, b, a}是相等的"  
可以从两种方式来看 Multiset
- 没有元素顺序限制的 ArrayList<E>
  - add(E) 添加单个给定元素
  - iterator() 返回一个迭代器, 包含 Multiset 的所有元素 (包括重复元素)
  - size() 返回所有元素个数 (包括重复元素)
- Map<E,Integer, 键为元素, 值为计数
  - count(Object) 返回给定元素的计数; HashMultiset.count 的复杂度为 O(1), TreeultiSet.count 的复杂度为 O(logn)
  - entrySet() 返回 Set<Multiset.Entry<E>>
  - elementSet() 返回所有不重复元素的 Set<E>
  - 所有 Multiset 实现的内存消耗随着不重复元素的个数线性增长

值得注意的是, 除了极少数情况, Multiset 和 JDK 中原有的 Collection 接口契约完全一致, 除了 TreeMultiset 在判断元素是否相等时, 与 TreeSet 一样用 compare 而不是 Object.equals
###### Multiset 不是 Map
Multiset<E> 不是 Map<E,Integer>, 虽然 Map 可能是某些 Multiset 实现的一部分; 准确来说 Multiset 是一种 Collection 类型, ; 关于 Multiset 和 Map 的显著区别还包括
- Multiset 中的元素计数只能是正数; 任何元素的计数都不能为负, 也不能是 0
- Multiset.size()返回集合的大小, 等同于所有元素计数的总和
- Multiset.iterator()会迭代重复元素
- Multiset 支持直接增加, 减少或设置元素的计数; setCount(elem, 0) 等同于移除所有 elem
- 对 Multiset 中没有的元素, Multiset.count(elem) 始终返回 0
###### Multiset 的各种实现

| Map | 对应的 Multiset | 是否支持 NULL 元素 |
| :--- | :--- | :--- |
| HashMap | HashMultiset | 是 |
| TreeMap | TreeMultiset | 是 (如果 comparator 支持) |
| LinkedHashMap | LinkedHashMultiset | 是 |
| ConcurrentHashMap | ConcurrentHashMultiset | 否 |
| ImmutableMap | 	ImmutableMultiset | 否 |

##### SortedMultiset
SortedMultiset 是 Multiset 接口的变种, 它支持高效地获取指定范围的子集; 例如可以用 `latencies.subMultiset(0,BoundType.CLOSED, 100, BoundType.OPEN).size()` 来统计站点中延迟在 100 毫秒以内的访问, 然后把这个值和 `latencies.size()` 相比, 以获取这个延迟水平在总体访问中的比例

##### Multimap
Guava 提供 Multimap 可以很容易把一个键映射到多个值, 可以替代 Map<K,List<V>> 或 Map<K,Set<V>> 的实现; 换句话说, Multimap 就是把键映射到任意多个值的一般方式, 可以用两种方式来看 Multimap
- 键-单个值映射: a -> 1, a -> 2, a -> 4, b -> 4, c -> 5
- 键-值集合映射: a -> [1, 2, 4], b -> 4, c -> 5

一般来说, Multimap 接口应该用第一种方式看待, 但 asMap() 视图返回 Map<K,Collection<V>>, 可以按另一种方式看待 Multimap; 很少会直接使用 Multimap 接口, 更多时候会用 ListMultimap 或 SetMultimap 接口, 它们分别把键映射到 List 或 Set
###### 修改 Multimap
Multimap.get(key) 以集合形式返回键所对应的值视图, 即使没有任何对应的值, 也会返回空集合; ListMultimap.get(key) 返回 List, SetMultimap.get(key) 返回 Set; 对于值视图集合进行修改最终都会反映到底层的 Multimap
```Java
Set<Person> aliceChildren = childrenMultimap.get(alice);
aliceChildren.clear();
aliceChildren.add(bob);
aliceChildren.add(carol);
```
###### Multimap 的视图
- asMap 为 Multimap<K,V> 提供 Map<K,Collection<V>> 形式的视图; 返回的 Map 支持 remove 操作, 并且会反映到底层的 Multimap, 但它不支持 put 或 putAll 操作; 更重要的是, 如果想为 Multimap 中没有的键返回 null, 而不是一个新的可写的空集合, 可以使用asMap().get(key)
- entries 用 Collection<Map.Entry<K, V>> 返回 Multimap 中所有 "键-单个值映射" (包括重复键)
- keySet 用 Set 表示 Multimap 所有不同的键
- keys 用 Multiset 表示 Multimap 中的所有键, 每个键重复出现的次数等于它映射的值的个数; 可以从这个 Multiset 中移除元素, 但不能做添加操作; 移除操作会反映到底层的 Multimap
- values() 用一个 "扁平" 的 Collection<V> 包含 Multimap 中的所有值
###### Multimap 不是 Map
Multimap<K,V> 不是 Map<K,Collection<V>>, 虽然某些 Multimap 实现中可能使用了 map; 它们之间的显著区别包括
- Multimap.get(key) 总是返回非 NULL, 但是可能空的集合; 这并不意味着 Multimap 为相应的键花费内存创建了集合
- 如果更喜欢像 Map 那样, 为 Multimap 中没有的键返回 NULL, 请使用 asMap() 视图获取一个 Map<K,Collection<V>>
- 当且仅当有值映射到键时, Multimap.containsKey(key) 才会返回 true
- Multimap.entries() 返回 Multimap 中所有 "键-单个值映射" (包括重复键); 如果想要得到所有 "键-值集合映射" 请使用 asMap().entrySet()
- Multimap.size() 返回所有 "键-单个值映射" 的个数, 而非不同键的个数; 要得到不同键的个数, 请改用 Multimap.keySet().size()
###### Multimap 的各种实现

| 实现 | 键行为类似 | 值行为类似 |
| :--- | :--- | :--- |
| ArrayMultimap | HashMap | ArrayList |
| HashMultimap | HashMap | HashSet |
| LinkedListMultimap* | LinkedListHashMap | LinkedList* |
| LinkedHashMultimap** | LinkedHashMap | LinkedHashMap |
| TreeMultimap | TreeMap | TreeSet |
| ImmutableListMultimap | ImmutableMap | ImmutableList |
| ImmutableListMultimap | ImmutableMap | ImmutableSet |

除了两个不可变形式的实现, 其他所有实现都支持 NULL 键和 NULL 值

##### BiMap
通常实现键值对的双向映射需要维护两个单独的 map, 并保持它们间的同步; 但这种方式很容易出错, 而且对于值已经在 map 中的情况, 会变得非常混乱
```Java
Map<String, Integer> nameToId = Maps.newHashMap();
Map<Integer, String> idToName = Maps.newHashMap();
nameToId.put("Bob", 42);
idToName.put(42, "Bob");
//如果"Bob"和42已经在map中了，会发生什么?
//如果我们忘了同步两个map，会有诡异的bug发生...
```
BiMap 是特殊的 Map
- 可以用 inverse() 反转 BiMap<K,V> 的键值映射
- 保证值是唯一的, 因此 values() 返回 Set 而不是普通的 Collection

在 BiMap 中, 如果想把键映射到已经存在的值会抛出 IllegalArgumentException 异常; 对特定值想要强制替换它的键, 请使用  BiMap.forcePut(key, value)

###### BiMap 的各种实现

| 实现 | 键值实现 | 值键实现 |
| :--- | :--- | :--- |
| HashBiMap | HashMap | HashMap |
| ImmutableBiMap | ImmutableMap | ImmutableMap |
| EnumBiMap | EnumMap | EnumMap |
| EnumHashBiMap | EnumMap | HashMap |

##### Table
TODO
##### ClassToInstanceMap
TODO
##### RangeMap
TODO
##### RangeSet
TODO

#### 集合工具类
工具类和特定集合接口对应关系如下

| 集合接口 | 属于 JDK 还是 Guava | 对应的 Guava 工具类 |
| :--- | :--- | :--- |
| Collection | JDK | Collections2 |
| List | JDK | Lists |
| Set | JDK | Sets |
| SortedSet | JDK | Sets |
| Map | JDK | Maps |
| SortedMap | JDK | Maps |
| Queue | JDK | Queues |
| Multiset | Guava | Multisets |
| Multimap | Guava | Multimaps |
| BiMap | Guava | Maps |
| Table | Guava | Tables |

##### 静态工厂方法
TODO
##### Iterables
在可能的情况下, Guava 提供的工具方法更偏向于接受 Iterable 而不是 Collection 类型; 对于不存放在内存的集合, 比如从数据库或其他数据中心收集的结果集, 因为实际上还没有攫取全部数据, 这类结果集都不能支持类似 size() 的操作  
因此, 很多支持所有集合的操作都在 Iterables 类中, 大多数 Iterables 方法有一个在 Iterators 类中对应的版本, 用来处理 Iterator; Iterables 使用 FluentIterable 类进行了补充, 它包装了一个 Iterable 实例, 并对许多操作提供了流式调用的语法  
###### 常用方法

| 方法 | 描述 |
| :--- | :--- |
| concat(Iterable) | 串联多个 iterable 的懒视图 |
| frequency(Iterable, Object) | 返回对象在 iterable 中出现的次数 |
| partition(Iterable, int)| 把 iterable 按指定大小分割, 得到的子集都不能进行修改操作 |
| getFirst(Iterable, T default) | 返回 iterable 的第一个元素, 若 iterable 为空则返回默认值 |
| getLast(Iterable, T default) | 返回 iterable 的最后一个元素, 若 iterable 为空则返回默认值 |
| elementsEqual(Iterable, Iterable) | 如果两个 iterable 中的所有元素相等且顺序一致, 返回 true |
| unmodifiableIterable(Iterable) | 返回 iterable 的不可变视图 |
| limit(Iterable, int) | 限制 iterable 的元素个数限制给定值 |
| getOnlyElement(Iterable) | 获取 iterable 中唯一的元素, 如果 iterable 为空或有多个元素, 则快速失败 |

###### 与 Collection 方法相似的工具方法
Collection 的实现天然支持操作其他 Collection, 但却不能操作 Iterable; 下面的方法中, 如果传入的 Iterable 是一个 Collection 实例, 则实际操作将会委托给相应的 Collection 接口方法

TODO
###### FluentIterable
TODO

##### Lists
TODO
##### Sets
TODO
##### Maps
TODO
##### Multisets
TODO
##### Multimaps
TODO
##### Tables
TODO

#### 集合扩展工具类

##### Forwarding 装饰器
针对所有类型的集合接口, Guava 都提供了 Forwarding 抽象类以简化装饰者模式的使用; Forwarding 抽象定义了一个抽象方法: delegate(), 可以覆盖这个方法来返回被装饰对象, 其他方法都会直接委托给 delegate()  
此外, 很多集合方法都对应一个 "标准方法[standardxxx]" 实现, 可以用来恢复被装饰对象的默认行为, 以提供相同的优点; 以下示例装饰一个 List, 让其记录所有添加进来的元素; 无论元素是用什么方法 (add(int, E), add(E), addAll(Collection)) 添加进来的, 都希望进行记录，因此需要覆盖所有这些方法
```java
public class AddLoggingList<E> extends ForwardingList<E> {
    final List<E> delegate; // backing list

    AddLoggingList(List<E> delegate) {
        this.delegate = delegate;
    }

    @Override
    protected List<E> delegate() {
        return delegate;
    }

    @Override
    public void add(int index, E elem) {
        System.out.println(String.format("add: index = %s, elem = %s", index, elem));
        super.add(index, elem);
    }

    @Override
    public boolean add(E elem) {
        return standardAdd(elem); // 用add(int, E)实现
    }

    @Override
    public boolean addAll(Collection<? extends E> c) {
        return standardAddAll(c); // 用add实现
    }
}
```
默认情况下, 所有方法都直接转发到被代理对象, 因此覆盖 `ForwardingMap.put` 并不会改变 `ForwardingMap.putAll` 的行为; 小心覆盖所有需要改变行为的方法, 并且确保装饰后的集合满足接口契约
##### PeekingIterator
TODO
##### AbstractIterator
TODO
##### AbstractSequentialIterator
有一些迭代器用其他方式表示会更简单, AbstractSequentialIterator 就提供了表示迭代的另一种方式
```Java
Iterator<Integer> powersOfTwo = new AbstractSequentialIterator<Integer>(1) { // 注意初始值1!
    protected Integer computeNext(Integer previous) {
        return (previous == 1 << 30) ? null : previous * 2;
    }
};
```
这里实现了 computeNext(T) 方法, 它能接受前一个值作为参数; 这里必须额外传入一个初始值, 或者传入 NULL 让迭代立即结束; 因为 computeNext(T) 假定 NULL 值意味着迭代的末尾 (AbstractSequentialIterator 不能用来实现可能返回 NULL 的迭代器)

---
title: Java集合接口介绍
abbrlink: 3518917720
date: 2020-06-09 22:42:57
categories:
  - Java
  - Core
tags:
  - 源码
author: 长歌
---
{% cq %}
Java集合所使用的接口定义了Java集合框架的主体脉络，通过组合不同的接口完成不同类型的集合定义与相关的操作。  
Java集合相关的主要接口有：`Iterator`、`Collection`、`List`、`Set`、`SortedSet`、`Queue`、`Deque`、`Map`、`SortedMap`
{% endcq %}
<!-- More -->

# Iterator

> 迭代器接口，迭代器模式的实现。可以用来遍历集合。

```java
public interface Iterator<E> {
    // 如果迭代具有更多元素，则返回 true 。
    boolean hasNext();
        
    // 返回迭代器中的下一个元素
    E next();

    // 从底层集合中删除此迭代器返回的最后一个元素，默认不支持移除（可选)
    default void remove() {
        throw new UnsupportedOperationException("remove");
    }

    // 对每个剩余元素执行给定的操作，直到所有元素都被处理或动作引发异常
    default void forEachRemaining(Consumer<? super E> var1) {
        Objects.requireNonNull(var1);

        while(this.hasNext()) {
            var1.accept(this.next());
        }

    }
}
```

# Collection
> 集合、容器。Java集合类中的顶级接口
> 继承 Iterator 以支持迭代


| 返回类型                 | 方法及描述                                                   |
| ------------------------ | ------------------------------------------------------------ |
| boolean                  | add(E e) <br/>确保此集合包含指定的元素（可选操作）。         |
| boolean                  | addAll(Collection<? extends E> c)<br>将指定集合中的所有元素添加到此集合（可选操作）。 |
| void                     | clear()<br>从此集合中删除所有元素                            |
| boolean                  | contains(Object o)<br>如果此集合包含指定的元素，则返回 `true` 。 |
| boolean                  | containsAll(Collection<?> c)<br>如果此集合包含指定 `集合`中的所有元素，则返回true。 |
| boolean                  | equals(Object o)<br>将指定的对象与此集合进行比较判断是否相等。 |
| int                      | hashCode()<br>返回此集合的哈希值                             |
| boolean                  | isEmpty()<br>如果此集合不包含元素，则返回 `true` 。          |
| Iterator\<E\>            | iterator()<br>返回此集合中的元素的迭代器                     |
| default Stream\<E\>      | parallelStream()<br>返回可能并行的 `Stream`                  |
| boolean                  | remove(Object o)<br>从该集合中删除指定元素的单个实例（如果存在）。 |
| boolean                  | removeAll(Collection<?> c)<br>删除指定集合中包含的所有此集合的元素 |
| default boolean          | removeIf(Predicate<? super E> filter)<br>删除满足给定谓词的此集合的所有元素。 |
| boolean                  | retainAll(Collection<?> c)<br>仅保留此集合中包含在指定集合中的元素 |
| int                      | size()<br>返回此集合中的元素数。                             |
| default Spliterator\<E\> | spliterator()<br>创建一个`Spliterator`以这个集合中的元素为源 |
| default Stream\<E\>      | stream()<br>返回以此集合作为源的顺序 `Stream`                |
| Object[]                 | toArray()<br>返回一个包含此集合中所有元素的数组              |
| \<T\> T[]                | toArray(T[] a)<br>返回包含此集合中所有元素的数组; 返回的数组的运行时类型是指定数组的运行时类型。 |



```java
public interface Collection<E> extends Iterable<E> {
    // Query Operations

    /**
     * 返回集合元素数目，如果集合包含超过 Integer.MAX_VALUE 个元素，返回Integer.MAX_VALUE
     *
     * @return 返回集合元素数目
     */
    int size();

    /**
     * 如果此集合不包含元素，则返回 true 。
     *
     * @return <tt>true</tt> 如果此集合不包含元素
     */
    boolean isEmpty();

    /**
     * 如果此集合包含指定的元素，则返回true 。返回true如果且仅当该集合至少包含一个元素e使得(o==null ? e==null : o.equals(e)) 。
     * <tt>(o==null&nbsp;?&nbsp;e==null&nbsp;:&nbsp;o.equals(e))</tt>.
     *
     * @param o 指定的元素
     * @return <tt>true</tt> 如果集合包含指定的元素
     * @throws ClassCastException 如果指定的元素类型与集合不匹配
     *         (<a href="#optional-restrictions">optional</a>)
     * @throws NullPointerException 如果给定元素是null并且集合不包含null元素
     *         (<a href="#optional-restrictions">optional</a>)
     */
    boolean contains(Object o);

    /**
     * 返回此集合中的元素的迭代器。 没有关于元素返回顺序的保证
     *（除非这个集合是提供保证的某个类的实例）。
     *
     * @return an <tt>Iterator</tt> over the elements in this collection
     */
    Iterator<E> iterator();

    /**
     * 返回一个包含此集合中所有元素的数组。 如果此集合对其迭代器返回的元素的顺序做出任何保证，则此方法必须以       * 相同的顺序返回元素。
         * 返回的数组将是“安全的”，因为该集合不保留对它的引用。 （换句话说，这个方法必须分配一个新的数组，即使这
         * 个集合是由数组支持的）。 因此，调用者可以自由地修改返回的数组。
         *
         * 此方法充当基于阵列和基于集合的API之间的桥梁。
     *
     * @return 一个包含此集合中所有元素的数组
     */
    Object[] toArray();

    /**
     * 将集合元素放入指定数组中(仅当给定数组元素足够放入且类型匹配时)
     * 当给定数组过大时，非元素位置会被设置为null
     *
     * @param <T> 包含集合的数组的运行时类型
     * @param 要存储此集合的元素的数组，如果它足够大; 否则，为此目的分配相同运行时类型的新数组。
     * @return 一个包含此集合中所有元素的数组
     * @throws ArrayStoreException 如果指定数组的运行时类型不是此集合中每个元素的运行时类型的超类型
     * @throws NullPointerException 如果指定的数组为空
     */
    <T> T[] toArray(T[] a);

    // Modification Operations

    /**
     * 添加元素到集合，如果集合不允许重复且集合中包含待添加元素，返回false。非正常添加情况通过异常进行说明。
     *
     * @param e 要被添加的元素
     * @return <tt>true</tt> true 如果此集合由于调用而更改
     * @throws UnsupportedOperationException 如果此集合不支持add操作
     * @throws ClassCastException 如果指定元素的类阻止将其添加到此集合
     * @throws NullPointerException 如果指定的元素为空，并且该集合不允许空元素
     * @throws IllegalArgumentException 如果元素的某些属性阻止其添加到此集合
     * @throws IllegalStateException 如果由于插入限制，此时无法添加该元素
     */
    boolean add(E e);

    /**
     * 移除集合中的指定元素
     *
     * @param o 要在集合中删除的元素(存在)
     * @return 删除成功返回 true
     * @throws ClassCastException 如果指定元素的类型与此集合不兼容
     *         (<a href="#optional-restrictions">optional</a>)
     * @throws NullPointerException 如果指定的元素为空，并且此集合不允许空元
     *         (<a href="#optional-restrictions">optional</a>)
     * @throws UnsupportedOperationException 如果此 集合不支持remove操作
     */
    boolean remove(Object o);


    // Bulk Operations

    /**
     * 判断此集合是否包含指定集合的所有元素
     *
     * @param  c 待判断的指定元素
     * @return 如果当前集合包含指定集合的所有元素，返回 true
     * @throws ClassCastException 如果指定元素的类型与此集合不兼容
     *         (<a href="#optional-restrictions">optional</a>)
     * @throws NullPointerException 如果指定的集合包含一个或多个空元素，并且此集合不允许空元素（ optional ），或者指定的集合为空
     *         (<a href="#optional-restrictions">optional</a>),
     *         or if the specified collection is null.
     * @see    #contains(Object)
     */
    boolean containsAll(Collection<?> c);

  //    后续方法不再详细介绍，具体说明可以查看API文档
  
    /**
     * 添加此集合中指定集合的元素。如果在添加过程中修改了指定集合，
     */
    boolean addAll(Collection<? extends E> c);

    /**
     * 移除此集合中出现在指定集合中的元素。调用后，当前集合不再含有指定集合中的元素。
     */
    boolean removeAll(Collection<?> c);

    /**
     * 1.8 函数式编程新增方法，删除集合中满足给定条件的所有元素，删除过程中发生移仓直接抛出
     */
    default boolean removeIf(Predicate<? super E> filter) {
        Objects.requireNonNull(filter);
        boolean removed = false;
        final Iterator<E> each = iterator();
        while (each.hasNext()) {
            if (filter.test(each.next())) {
                each.remove();
                removed = true;
            }
        }
        return removed;
    }

    /**
     * 删除当前集合不在指定集合出现过的所有元素（求交集）
     */
    boolean retainAll(Collection<?> c);

    /**
     * 删除集合所有元素
     */
    void clear();


    // Comparison and hashing

    /**
     * 判断传入对象与当前集合是否相等
     */
    boolean equals(Object o);

    /**
     * 返回此集合的哈希值
     */
    int hashCode();

    /**
     * 可分隔迭代器 Spliterator 1.8以后 提供
     */
    @Override
    default Spliterator<E> spliterator() {
        return Spliterators.spliterator(this, 0);
    }

    /**
     * 创建一个流，使用集合的元素作为其资源
     */
    default Stream<E> stream() {
        return StreamSupport.stream(spliterator(), false);
    }

    /**
     * 创建一个并行的流（以此集合作为资源）。
     */
    default Stream<E> parallelStream() {
        return StreamSupport.stream(spliterator(), true);
    }
}
```



# List

> 列表接口，继承自 Collection,下面只介绍其相对于Collection新增的方法

| 返回类型          | 方法及描述                                                   |
| ----------------- | ------------------------------------------------------------ |
| boolean           | addAll(int index, Collection<? extends E> c)<br>将指定集合中的所有元素插入到此列表中的指定位置（可选操作） |
| default void      | replaceAll(UnaryOperator<E> operator)<br>将该列表的每个元素替换为将该运算符应用于该元素的结果。 |
| default void      | sort(Comparator<? super E> c)<br>使用传入的 `Comparator`排序此列表来比较元素 |
| E                 | get(int index)<br>返回此列表中指定位置的元素。               |
| E                 | set(int index, E element)<br>用指定的元素（可选操作）替换此列表中指定位置的元素 |
| void              | add(int index, E element)<br>将指定的元素插入此列表中的指定位置（可选操作）。 |
| E                 | remove(int index)<br>删除该列表中指定位置的元素（可选操作）。 |
| int               | indexOf(Object o)<br>返回此列表中指定元素的第一次出现的索引，如果此列表不包含元素，则返回-1。 |
| int               | lastIndexOf(Object o)<br>返回此列表中指定元素的最后一次出现的索引，如果此列表不包含元素，则返回-1。 |
| ListIterator\<E\> | listIterator()<br>返回列表中的列表迭代器（按适当的顺序）。   |
| ListIterator\<E\> | listIterator(int index)<br>从列表中的指定位置开始，返回列表中的元素（按正确顺序）的列表迭代器。 |
| List\<E\>         | subList(int fromIndex, int toIndex)<br>返回此列表中指定的 `fromIndex` （含）和 `toIndex`之间的列表。返回列表的更改将反应在当前列表上 |

# Set

> Set集合接口，继承自`Collection`，自身没有增加新的方法，接口方法同`Collection`一致.



## SortedSet\<E\>

> 有序Set集合接口，继承自`Set`->`Collection`
>
> 集合中的元素需要实现`Compareable`接口(或者可以被指定的Comparator作为参数比较)，并且所有元素是可比较的（即，Mutual Comparable只是意味着两个对象互相接受作为compareTo方法的参数）

| 返回类型                 | 方法名称及描述                                               |
| ------------------------ | ------------------------------------------------------------ |
| Comparator<? super E>    | comparator()<br>返回用于对该集合中的元素进行排序的比较器，如果此集合使用的是元素的自然排序返回null |
| SortedSet\<E\>           | subSet(E fromElement, E toElement)<br>返回该集合的一部分，范围为from到to，独占。 |
| SortedSet\<E\>           | headSet(E toElement)<br>返回该集合的一部分，其元素严格小于toElement |
| SortedSet\<E\>           | tailSet(E fromElement)<br/>返回该集合的一部分，其元素严格大于toElement |
| E                        | first()<br/>返回该集合的第一个(比较值最小的)元素             |
| E                        | last()<br/>返回该集合的最后一个(比较值最大的)元素            |
| default Spliterator\<E\> | spliterator()<br>在此集合中的元素上创建一个Spliterator       |

# Queue

> 队列接口，队列，先进先出。它只允许在表的前端进行删除操作，而在表的后端进行插入操作
>
> `LinkedList`实现了`Queue`接口，所以我们可以把`LinkedList`当成队列来使用

| 返回类型 | 方法名称及描述                                               |
| -------- | ------------------------------------------------------------ |
| E        | element()<br>查询但不删除这个队列的头(**队列为空抛出异常**)  |
| boolean  | offer(E e)<br>不违反容量限制的情况下立即执行，将制定元素插入队列 |
| E        | peek()<br>查询但不删除队列的头，如果队列为空返回null         |
| E        | pool()<br>查询并删除队列的头，如果队列为空返回null           |
| E        | remove()<br>查询并删除队列的头(**队列为空抛出异常**)         |

## Deque

> 双端队列，该接口提供了插入，移除，查询元素的方法。这些方法每种存在两种形式：返回特殊值(null或false)和抛出异常。
>
> `Deque`没有严格要求禁止插入空元素，但是我们要避免这么做，因为很多时候我们通过判断null来确定`Deque`是否是空的。

| 返回类型       | 方法名称                        | 方法描述                                              |
| -------------- | ------------------------------- | ----------------------------------------------------- |
| void           | addFirst(E e)                   | 插入队列前面，超出容量限制抛出`IllegalStateException` |
| void           | addLast(E e)                    | 插入队列末尾，超出容量限制抛出`IllegalStateException` |
| boolean        | offerFirst(E e)                 | 插入队列头部                                          |
| boolean        | offerLast(E e)                  | 插入队列尾部                                          |
| E              | removeFirst()                   | 查询并删除队列第一个元素                              |
| E              | removeLast()                    | 查询并删除队列最后一个元素                            |
| E              | pollFirst()                     | 查询并删除第一个元素，队列为空返回null                |
| E              | pollLast()                      | 查询并删除最后一个元素，队列为空返回null              |
| E              | getFirst()                      | 获取队列第一个元素(不删除)                            |
| E              | getLast()                       | 获取队列最后一个元素(不删除)                          |
| E              | peekFirst()                     | 查询且不删除队列第一个元素，队列为空返回null          |
| E              | peekLast()                      | 查询且不删除队列最后一个元素，队列为空返回null        |
| boolean        | removeFirstOccurrence(Object o) | 从队列中删除第一次出现的指定元素                      |
| boolean        | removeLastOccurrence(Object o)  | 从队列中删除最后一次出现的指定元素                    |
| void           | push(E e)                       | 入栈，超出限制抛出`IllegalStateException`             |
| E              | pop()                           | 出栈                                                  |
| Interator\<E\> | descendingIterator()            | 返回倒序的迭代器                                      |



# Map

> Map集合是一个**双列集合**，一个元素包含两个值(key,value)
>
> Map集合中的元素，key不允许重复，value可以重复

| 返回类型          | 方法名称                                                     | 方法描述                                                     |
| ----------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| void              | clear()                                                      | 清空元素                                                     |
| default V         | compute(K key,BiFunction<? super K , ? extends V> remappingFunction) | 尝试计算指定键的映射及其当前映射的值（如果没有当前映射， `null` ） |
| default V         | computeIfAbsent(K key , Function<? super K, ? exnteds V> remappingFunction) | 如果指定的key没有对应的value（或映射到 `null` ），则尝试使用给定的映射函数计算其值，并将其输入到此映射中，除非 `null` |
| default V         | computeIfPresent(K key,BiFunction<? super K , ? super V , ? extends V> remappingFunction) | 如果指定的key存在，尝试计算给定key和value的映射              |
| boolean           | containsKey(Object key)                                      | 如果集合包含这个key，返回true                                |
| boolean           | containsValue(Object value)                                  | 如果集合包含这个value，返回ture                              |
| Set<Entry<K , V>> | entrySet()                                                   | 返回包含此集合映射的Set集合                                  |
| default void      | forEach(BiConsuer<? super K , ? super V> action)             | 对集合的每个映射执行给定的操作                               |
| V                 | get(Object key)                                              | 返回给定key的value，key不存在返回null                        |
| default V         | getOfDefalt(Object key , V defaultValue)                     | 返回给定key的value，key不存在返回默认值                      |
| boolean           | isEmpty()                                                    | 如果集合为空，返回true                                       |
| Set\<K\>          | keySet()                                                     | 返回包含这个集合所有key的Set集合                             |
| default V         | merge(K key , V value , BiFunction<? super V , ? super V , ? extends V> remappingFunction) | 如果key不存在或值为null,则将其value设置为给定值              |
| V                 | put(K key , V value)                                         | 添加映射到集合，key重复会覆盖value的值                       |
| void              | putAll(Map<? extends K , ? extends V> m)                     | 添加给定集合所有映射到此集合                                 |
| default V         | putIfAbsent(K key, V value)                                  | 如果key不存在，返回null并添加映射。否则返回value             |
| V                 | remove(Object key)                                           | key存在，移除映射返回value;否则，返回null                    |
| default boolean   | remove(Object key, Object value)                             | 当key和value都符合集合元素时删除映射，返回true               |
| default V         | replace(K key, V value)                                      | key存在时替换value并返回原值，key不存在时返回null            |
| default boolean   | replace(K key, V oldValue, V newValue)                       | 映射存在时，替换value的值为newValue                          |
| default void      | replaceAll(BiFunction<? super K, ? super V, ? extends V> function) | 对映射执行方法，将value替换为方法返回值。                    |
| int               | size()                                                       | 返回集合大小                                                 |
| Collection<V>     | values()                                                     | 返回包含此集合所有元素的Colleciton                           |



## SortedMap

> 根据Key，和对应比较规则进行排序的Map集合接口

| 返回类型              | 方法名称                   | 方法描述                                        |
| --------------------- | -------------------------- | ----------------------------------------------- |
| Comparator<? super K> | comparator()               | 返回比较器，如果使用key的自然顺序排序则返回null |
| K                     | firstKey()                 | 返回第一个key(排序值最小)                       |
| SortedMap<K,V>        | headMap(K toKey)           | 返回小于toKey的子集合                           |
| K                     | lastKey()                  | 返回最后一个key(排序值最大)                     |
| SortedMap<K,V>        | subMap(K fromKey, K toKey) | 返回fromKey到toKey的子集合                      |
| SortedMap<K,V>        | tailMap(K fromKey)         | 返回大于fromKey的子集合                         |

# 注意
- **重点关注接口代表的数据类型**
- **重点关注接口方法的用途及异常场景**

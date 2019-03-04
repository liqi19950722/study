# Java集合

[总览](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/doc-files/coll-index.html)

术语

- 不可修改*unmodifiable* 可修改*modifiable*

  > 不支持`add` `remove` `clear`的集合

- 不可变*immutable* 可变*mutable*

  > 任何更改都是不可见的集合

- 固定大小*fixed-size* 非固定大小 *variable-size*

  >元素可修改，大小不变的列表

- 随机访问*random access*  顺序访问*sequential access* 

  > 索引访问

**接口(Collection interfaces)**

- `java.lang.Iterable`
  - `java.util.Collection`
    - `java.util.Set`
      - `java.util.SortedSet`
        - `java.util.NavigableSet`
    - `java.util.List`
    - `java.util.Queue`
      - `java.util.Deque`
      - `java.util.concurrent.BlockingQueue`
        - `java.util.concurrent.BlockingDeque`
        - `java.util.concurrent.TransferQueue`
- `java.util.Map`
  - `java.util.SortedMap`
    - `java.util.NavigableMap`
  - `java.util.concurrent.ConcurrentMap`
    - `java.util.concurrent.ConcurrentNavigableMap`



**遗留实现(Legacy implementations)**

- `java.util.Vector`
  - `java.util.Stack`
- `java.util.Dictionary`
  - `java.util.Hashtable`
- `java.util.Enumeration`
- ~~`java.util.BitSet`~~



**抽象实现(Abstract implementations)**

- `java.util.AbstractCollection`
  - `java.util.AbstractList`
    - `java.util.AbstractSequentialList`
  - `java.util.AbstractSet`
  - `java.util.AbstractQueue`
- `java.util.AbstractMap`



**便利实现(Convenience implementations)**

`java.util.Collections`

- 单例实现`Collections.singleton*`

- 空实现`Collections.empty*`

- 转换实现

  - `java.util.Collections#enumeration`
  - `java.util.Collections#list`
  - `java.util.Collections#newSetFromMap`
  - `java.util.Collections#asLifoQueue`
  - `java.util.Arrays#hashCode`
  - `java.util.Arrays#toString`

- 列举实现

  - `java.util.stream.Stream#of`
  - `java.util.List#of`
  - `java.util.Set#of`
  - `java.util.Map#of`

  

**包装实现(Wrapper implementations)**

- 同步包装接口`java.util.Collections#synchronized*`
- 只读包装接口`java.util.Collections#unmodifiable*`
- 类型安全包装接口`java.util.Collections#checked*`



**特殊实现(Special-purpose implementations)**

- `java.util.WeakHashMap`

- `java.util.IdentityHashMap`



**算法(Algorithms)**

- 计算复杂度
- 空间复杂度
- 递归算法
- 稳定性
- 比较排序

排序算法

- 选择排序
- 冒泡排序
- 插入排序
- 快速排序
- 并归排序

二分查找算法

## 附

[Java collections cheat sheet](https://cn.bing.com/search?q=java%20collections%20cheat%20sheet&qs=ds&form=QBRE)

[Consistent Hash](https://en.wikipedia.org/wiki/Consistent_hashing)

[Rendezvous Hash](https://en.wikipedia.org/wiki/Rendezvous_hashing)

[sortalgorithm](https://en.wikipedia.org/wiki/Sorting_algorithm)

[QuickSort](https://en.wikipedia.org/wiki/Quicksort)

[BubbleSort](https://en.wikipedia.org/wiki/Bubble_sort)

[InsertionSort](https://en.wikipedia.org/wiki/Insertion_sort)

[SelectionSort](https://en.wikipedia.org/wiki/Selection_sort)

[MergeSort](https://en.wikipedia.org/wiki/Merge_sort)


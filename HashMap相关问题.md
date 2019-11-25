### 1. 为什么长度都是2的n次方

>  其实有两个原因，一个是为了均匀分布，一个是为了效率。

> 在HashMap中为了获取 key 对应的 index 值，是按照下面的公式

​     index = HashCode(key) & (Length - 1)

####  1.1 均匀分布

举个例子，Length为10时，Length - 1二进制为 1001 ，那么 前面的hash值，后四位 为 1001，或者1011，或者1111，得到的index都是一样的。

但是如果Length为16，Length - 1二进制位 1111，就能避免这些问题。hash的最终结果会达到最平均，减少了key的碰撞，加快了查询效率。

#### 1.2 效率问题

其实一般情况下，我们是使用HashCode(key) % (Length - 1) 来计算index的，也就是hash值对length取余。但是，Sun的人发现，在 length为2的n次方时，

hash &（length - 1）== hash % (length - 1)

并且，位运算的效率比取余高很多，因此长度都定为2的n次方。



详情参考这两个博客

[HashMap的长度为什么设置为2的n次方](https://blog.csdn.net/sybnfkn040601/article/details/73194613)

[HashMap中的为什么hash的长度为2的幂而&位必须为奇数](https://blog.csdn.net/zjcjava/article/details/78495416)



### 2. hash碰撞

> 这个问题，要分版本说了

#### 2.1 jdk 1.7

> 1.7版本，hash碰撞后，会通过 hash & (length - 1), 取到index，遍历该index对应的Entry链表，有 key 值一致的，就把新的 value 填入，并返回老的 value。
>
> 如果没有一致的，就使用 key 和 value ，创建新的 entry，作为整个链表的头，重新填入 数组的index位置。

#### 2.2 jdk 1.8

> 数组 + 链表的结构，在遍历长链表的时候速度太慢。
>
> 为了优化，在index对应的entry，其后面的链表长度大于等于8时，链表会转化为红黑树。



### 3. resize方法扩容

> 当HashMap插入的元素数量，大于 threshold = 扩容因子 * 容量 时，需要创建一个容量为原先二倍的，新数组，并把老数组里的数据复制过去



> 遍历老数组时，对每个数据分情况讨论

- 如果只是单一元素，那么 计算新的位置并插入就好
- 如果是一条链表，需要遍历链表，并且一个个的判断(e.hash & oldCap 注意 oldCap是老HashMap的长度，默认为16，不用 -1)，其在新链表中是否应该修改位置(其实一条链表，最多也只能在新hashmap中，分成两条链表)。最后插入对应位置

详情参考这个:

[Java 1.8中HashMap的resize()方法扩容部分的理解](https://blog.csdn.net/u013494765/article/details/77837338)



### 4. 与HashTable, SparseArray，ArrayMap的区别

#### 4.1 SparseArray的原理

> SparseArray比HashMap更节省内存，某些情况下性能更好。
>
> 主要是能避免对key的自动装箱，比如 int 到 Interger。



> SparseArray内部，使用 int[] mKeys， Object[] mValues，
>
> 两个数组来进行数据存储，一个存储key，一个存储value。
>
> key数组是从小到大排列的，但是只支持int类型的key。



> SparseArray在存取的时候，都是使用二分查找确定key的位置。

#### 4.2 SparseArray推荐使用场景

SparseArray在添加，查找，删除数据时，都要进行二分查找，所以数据量大时性能会大打折扣，降低至少50%。

以下两个场景下推荐使用

- 数据量不大，最好千级以内
- key必须为int类型

#### 4.3 ArrayMap

> 与SparseArray类似，但是可以存任意类型。
>
> 看下他的定义 ArrayMap<K, V> implements Map<K, V>。



> 内部两个数组，
>
> 分别是按照key的hash值从小到大排序的，int[] mHashes数组。
>
> 以及依次保存key和value的，Object[] mArray。
>
> mHashes数组的第i个元素，对应key和value，在mArray中的下标，是 2 * i 和 2 * i + 1

#### 4.4 HashTable



#### 4.5参考

[Android内存优化（使用SparseArray和ArrayMap代替HashMap）](https://blog.csdn.net/u010687392/article/details/47809295#commentsedit)

[HashMap与ArrayMap(和SparseArray)的比较与选择](https://blog.csdn.net/shangsxb/article/details/78898323)
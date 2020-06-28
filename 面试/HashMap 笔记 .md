### 1. 为什么长度都是2的n次方

> 其实有两个原因，一个是为了均匀分布，一个是为了效率。

> 在HashMap中为了获取 key 对应的 index 值，是按照下面的公式

 index = HashCode(key) & (Length - 1)

#### 1.1 均匀分布

举个例子，Length为10时，Length - 1二进制为 1001 ，那么 前面的hash值，后四位 为 1001，或者1011，或者1111，得到的index都是一样的。

但是如果Length为16，Length - 1二进制位 1111，就能避免这些问题。hash的最终结果会达到最平均，减少了key的碰撞，加快了查询效率。

#### 1.2 效率问题

其实一般情况下，我们是使用HashCode(key) % (Length - 1) 来计算index的，也就是hash值对length取余。但是，Sun的人发现，在 length为2的n次方时，

hash &（length - 1）== hash % (length - 1)

并且，位运算的效率比取余高很多，因此长度都定为2的n次方。

详情参考这两个博客

[HashMap的长度为什么设置为2的n次方](https://blog.csdn.net/sybnfkn040601/article/details/73194613)

[HashMap中的为什么hash的长度为2的幂而&位必须为奇数](https://blog.csdn.net/zjcjava/article/details/78495416)

#### 1.3 HashMap的hash方法

jdk 1.8 的hash方法，相比1.7进行了简化，源码如下

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

hashmap中不是直接使用key的hashcode值，而是将key.hashCode无符号右移16位后，与自身异或的结果，作为hash值。原因如下

- 混合原始hashcode的高低位，增大了hash值的随机性，从而降低了hash碰撞的概率，hash值可以分布的更分散
- `hash()` 是高频操作，位运算的效率比较高

### 2. jdk 1.7 与 jdk 1.8 的变化

1. （数据结构）数组tab + 链表 改成了 数组tab + 链表 或 红黑树
2. （插入）链表的插入方式从头插法改成了尾插法，简单说就是插入时，如果数组位置上已经有元素，1.7将新元素放到数组中，原始节点作为新节点的后继节点。1.8遍历链表，将元素放置到链表的最后
3. （resize）扩容的时候1.7需要对原数组中的元素进行重新hash定位在新数组的位置，1.8采用更简单的判断逻辑，位置不变或索引+旧容量大小
4. （resize时机）在插入时，1.7先判断是否需要扩容，再插入，1.8先进行插入，插入完成再判断是否需要扩容
5. （hash）hashmap类中的hash方法逻辑变化

### 3. resize方法扩容

> 遍历老数组时，对每个数据分情况讨论

- 如果只是单一元素，那么 计算新的位置`hash & (newCap -1)` 并插入就好

- 如果是一条链表，需要遍历链表，并且一个个的判断`e.hash & oldCap ` (注意 oldCap是老HashMap的长度，默认为16，不用 -1)，来确定其在新链表中是否应该修改位置(其实一条链表在新hashmap中，最多分成两条链表。结果是false对应老位置 `i`, 结果为 true 对应新位置 `i+oldCap`)

详情参考这个:

[Java 1.8中HashMap的resize()方法扩容部分的理解](https://blog.csdn.net/u013494765/article/details/77837338)



### 4. HashMap中的红黑树

HashMap 中有三个关于红黑树的关键参数:

- TREEIFY_THRESHOLD-------8

  当拉链中的结点数超过8，会把拉链转为红黑树

- UNTREEIFY_THRESHOLD-------6

  扩容后，因为一条拉链可能被分为两条链。新链中的结点数小于6时，将红黑树还原为链表结构

- MIN_TREEIFY_CAPACITY-------64

  最小的树形化容量，大致是只有hashmap中的`tab.length`超过64，才能树形化

更多关于HashMap中红黑树的内容，看看  [HashMap 在 JDK 1.8 后新增的红黑树结构](https://blog.csdn.net/wushiwude/article/details/75331926#comments)

### 5. HashMap put 原理

```java
// onlyIfAbsent 为 true时，表示当遇到key完全一致的case时，是否用新值 value 取代老 value
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        // p 是key对应hash值，算出来的对应的数组里的 node
        // 如果数组里的这一项为空
        if ((p = tab[i = (n - 1) & hash]) == null)
        		// 创建一个对应的新 Node 对象
            tab[i] = newNode(hash, key, value, null);
        else {
        		// hash值对应数组里的node不为空
            Node<K,V> e; K k;
            // Node e保存当前node链的头
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // 如果当前链已经转化为红黑树，那么走红黑树的插入逻辑
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
            		// 遍历整条Node链
                for (int binCount = 0; ; ++binCount) {
                		// 走到链表尾，则直接插入
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        // 整条链的长度达到8，把整个链转为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    // 找到key完全一致Node，跳出循环
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            // e != null 表示 在上面的循环中，我们找到了一个Node结点，key与我们的新key完全一致
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                // hashmap允许插入null值，如果当前结点是null，可以直接插入
                // 如果onlyIfAbsent为false（默认值）,也可以直接插入
              
              	// onlyIfAbsent 为 true时，代表如果新插入的key-value
              	// key与链表里已存在的node的key完全一致 & oldValue 不为null时
              	// 能不能插入new value
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
  			// 整个hashmap里的已插入key-value 的数量超过一定比例
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```





### 6. 与HashTable, SparseArray，ArrayMap的区别

#### 6.1 SparseArray的原理

> SparseArray比HashMap更节省内存，某些情况下性能更好。

>

> 主要是能避免对key的自动装箱，比如 int 到 Interger。

> SparseArray内部，使用 int[] mKeys， Object[] mValues，

>

> 两个数组来进行数据存储，一个存储key，一个存储value。

>

> key数组是从小到大排列的，但是只支持int类型的key。

> SparseArray在存取的时候，都是使用二分查找确定key的位置。

#### 6.2 SparseArray推荐使用场景

SparseArray在添加，查找，删除数据时，都要进行二分查找，所以数据量大时性能会大打折扣，降低至少50%。

以下两个场景下推荐使用

- 数据量不大，最好千级以内

- key必须为int类型

#### 6.3 ArrayMap

> 与SparseArray类似，但是可以存任意类型。

> 看下他的定义 ArrayMap<K, V> implements Map<K, V>。

> 内部两个数组，

> 分别是按照key的hash值从小到大排序的，int[] mHashes数组。

> 以及依次保存key和value的，Object[] mArray。

> mHashes数组的第i个元素，对应key和value，在mArray中的下标，是 2 * i 和 2 * i + 1

#### 6.4 HashTable

> 多线程中不建议使用HashTable，建议使用ConcurrentHashMap

- HashTable的put和get方法均加了 `synchronized` 锁，线程安全
- HashTable使用的还是数组+链表的数据结构，hash碰撞时还是头插法
- HashTable 初始大小为11，每次扩容为原来的 `2N+1`。
- 正是这个原因，在计算key在hash表中的位置时，需要用除法，而没法像hashmap那样用 `key & capacity`，速度比较慢。
- 好处是HashTable使用奇数的作为表大小，取模hash的结果会更加均匀。
- HashMap中，key和value都可以为null。HashTable中都不能为null，会报空指针



#### 6.5 ConcurrentHashMap

- put 操作

  对于put操作，如果Key对应的数组元素为null，则通过[CAS操作](http://www.jasongj.com/java/thread_safe/#CAS（compare-and-swap）)将其设置为当前值。

  如果Key对应的数组元素（也即链表表头或者树的根元素），不为null。则对该元素使用synchronized关键字申请锁，然后进行操作。

  如果该put操作使得当前链表长度超过一定阈值，则将该链表转换为树，从而提高寻址效率。

- get操作

  ```java
      // ConcurrentHashMap中的Node数组
  		transient volatile Node<K,V>[] table
  		······
  		static class Node<K,V> implements Map.Entry<K,V> {
          final int hash;
          final K key;
          volatile V val;
          volatile Node<K,V> next;
          ......
      }
  ```

  

  对于读操作，由于 Node数组 被volatile关键字修饰，因此不用担心数组的可见性问题。同时每个元素是一个Node实例（Java 7中每个元素是一个HashEntry），它的Key值和hash值都由final修饰，不可变更，无须关心它们会被修改。而Value及next引用，也由volatile修饰，可见性也有保障。

#### 6.6 参考

[Android内存优化（使用SparseArray和ArrayMap代替HashMap）](https://blog.csdn.net/u010687392/article/details/47809295#commentsedit)

[HashMap与ArrayMap(和SparseArray)的比较与选择](https://blog.csdn.net/shangsxb/article/details/78898323)

[Java进阶（六）从ConcurrentHashMap的演进看Java多线程核心技术](http://www.jasongj.com/java/concurrenthashmap/)
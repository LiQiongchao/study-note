# HashMap 源码分析（详细）

本文分析基于 `JDK1.8` 

HashMap是使用数组 + 链表 + 红黑树 的结构来存储的。当hash冲突时使用链表来存储数据，当链表上的数量大于等于8时，转换成红黑树存储元素。

## `hashCode` 方法

HashMap中处处用到了hashCode，先说一下hashCode方法。

HashCode#hash(Object key)

```java
    /**
     * Computes key.hashCode() and spreads (XORs) higher bits of hash
     * to lower.  Because the table uses power-of-two masking, sets of
     * hashes that vary only in bits above the current mask will
     * always collide. (Among known examples are sets of Float keys
     * holding consecutive whole numbers in small tables.)  So we
     * apply a transform that spreads the impact of higher bits
     * downward. There is a tradeoff between speed, utility, and
     * quality of bit-spreading. Because many common sets of hashes
     * are already reasonably distributed (so don't benefit from
     * spreading), and because we use trees to handle large sets of
     * collisions in bins, we just XOR some shifted bits in the
     * cheapest possible way to reduce systematic lossage, as well as
     * to incorporate impact of the highest bits that would otherwise
     * never be used in index calculations because of table bounds.
     */
    static final int hash(Object key) {
        int h;
        // 用hashCode的高位与低位进行异或运算，让高位参与运算，防止hashCode过小时冲突问题
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

知识点补充：

* `^` 运算是相应位数的什不一样时取1，否则取0
* 正数在计算机底层是以原码形式存储，而负数是以补码形式存储
* `Integer#hashCode()` 值是本身；`Long#hashCode()` 与 `HashMap` 的相似 `(int)(value ^ (value >>> 32))` ；`String#hashCode()` 是 `h = 31 * h + val[i];` 把原本的 hash 乘以31后再加上每个字符串起来。



## `HashMap` 的构造器

### 无参构造器

```java
    /**
     * The load factor used when none specified in constructor.
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    
   /**
     * Constructs an empty <tt>HashMap</tt> with the default initial capacity
     * (16) and the default load factor (0.75).
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```

指定了默认的加载因子，`Node` 的初始化在第一次放元素的时候。

### 有参构造器

有参构造器最后调用的都是一个。

```java
    /**
     * Constructs an empty <tt>HashMap</tt> with the specified initial
     * capacity and load factor.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
```

注意的是 `tableSizeFor(initialCapacity)` 的方法，取比自己最大且最近的2的幂次方的数。

```java
    /**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        // 先减1，防止该数刚好是2的幂次方
        int n = cap - 1;
        // 利用最高位的1把低全变成1，这样变变成全部的1了。比如：9
        // 1001 | 100 = 1101
        n |= n >>> 1;
        // 1101 | 10 = 1111，上次或后最差的情况会把前两位变成1，这一步操作后就能把前4位变成1
        n |= n >>> 2;
        // 1111 | 1 = 1111
        n |= n >>> 4;
        // 1111 | 0 = 1111
        n |= n >>> 8;
        // 1111 | 0 = 1111
        n |= n >>> 16;
        // 因为是以前面或后的结果再移动，所以移动16次刚好把int的数给处理完。
        // 再加1返回，刚好是2的幂次方
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

举例说明：

```java
n = 0100 0000 0000 0000

0100 0000 0000 0000
 010 0000 0000 0000		n |= n >>> 1
0110 0000 0000 0000
  01 1000 0000 0000		n |= n >>> 2
0111 1000 0000 0000
     0111 1000 0000		n |= n >>> 4
```



## put方法

### `putVal()` 方法

```java
    /**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
	// onlyIfAbsent key 存在时是否覆盖 value，true为不覆盖
	// evict HashMap中什么也没有做，在子类LinkHashMap中用来删除头节点。
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        // 第一次put时，会初始化数组
        if ((tab = table) == null || (n = tab.length) == 0)
            // 初始化 & 扩容 后面分析
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            // 当该位置为null时直接放入该位置
            tab[i] = newNode(hash, key, value, null);
        else {
            // 当该数组的位置不为null时，然后判断是链表还是红黑树
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                // 如果插入元素的key与position上key相等，就把赋值给临时变量e
                e = p;
            else if (p instanceof TreeNode)
                // 如果是红黑树节点，就放入树中
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                // key不等于链表的头节点，也不是红黑树时，遍历链表，放到链表的最后面
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        // 放置链表最后面
                        p.next = newNode(hash, key, value, null);
                        // >= 7时，即链表上有8个元素的时候进行转换成红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        // 等于链表上存在的某一个节点时，退出，走下面的是否覆盖的操作
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                // 修改值并返回旧值
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            // 扩容
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```



### `resize()` 方法

```java
    /**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *
     * @return the table
     */
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        // 旧数组的阈值
        int oldThr = threshold;
        int newCap, newThr = 0;
        // 扩容而非初始化
        if (oldCap > 0) {
            // 容量达到最大，无法扩容
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 扩容
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        // 初始化数组
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        // 计算扩容的阈值
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    // 该位置上只有一个元素，直接放到新数组
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    //　该位置是红黑树结构
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    // 链表
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // 在新数组的下标还是当前的下标
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            // 在新数组的下标是 oldCap + index
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        // 
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

#### 链表迁移

假如旧链表在旧数组的下标是3，capacity 是16，扩容后，capacity 是32。链表迁移是把链表上的数据分成两组，一组数据在新数组的下标是3，另一 组的下标应该是19（oldCapacity + index）。分组的依据是使用 `e.hash & oldCap` 因为在旧的数组上计算下脚标使用的是 `e.hash & oldCap - 1` ，所以扩容后，只比较 oldCap 的最高位就能确定在扩容后数组的位置。

## get方法

get的代码比较简单。

### get(Object)

```java
    /**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code (key==null ? k==null :
     * key.equals(k))}, then this method returns {@code v}; otherwise
     * it returns {@code null}.  (There can be at most one such mapping.)
     *
     * <p>A return value of {@code null} does not <i>necessarily</i>
     * indicate that the map contains no mapping for the key; it's also
     * possible that the map explicitly maps the key to {@code null}.
     * The {@link #containsKey containsKey} operation may be used to
     * distinguish these two cases.
     *
     * @see #put(Object, Object)
     */
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
```

### getNode(int, Object)

```java
    /**
     * Implements Map.get and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            // 如果是数组元素（链表/红黑树的第一个值），直接返回
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    // 查找红黑树
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    // 遍历查看链表
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```






[toc]
## 简介 ##
- HashMap可以根据键值存取数据。
- HashMap位于 **java.util** 包下，继承AbsractMap，实现Map，Cloneable，Serializable接口。
```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
```

## 内部结构 ##
- **链表**的数组，jdk1.8中当链表长度大于8时，链表会转化为**红黑树**。<br>![](https://tech.meituan.com/img/java-hashmap/hashMap%E5%86%85%E5%AD%98%E7%BB%93%E6%9E%84%E5%9B%BE.png)
- 结点
    - 链表：包含键、值、hash值、下一个结点引用。
    - 树结点：继承LinkedHashMap.Entry和HashMap.Node，包含红黑树相关结点。
```java
    transient Node<K,V>[] table;

    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
	}
    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;
	}
```

## 常用方法 ##
### put ###
- 流程图。<br>![](https://tech.meituan.com/img/java-hashmap/hashMap%20put%E6%96%B9%E6%B3%95%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B%E5%9B%BE.png)
- 源代码。
```java
    public V put(K key, V value) {
		//计算key对应的hash
        return putVal(hash(key), key, value, false, true);
    }

    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
		//判断table是否为空或者长度是否为0，是则扩容
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
		//结点索引的桶的头结点为p，如果为null，则直接插入Node
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
			//key和头结点的key相等，用e指向头结点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
			//如果桶头结点为红黑树结点，则插入树结点
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
					//在链表的尾结点后插入新结点
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
						//如果结点数过大，将链表转化为红黑树之后在插入
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
					//如果存在相同的key，e指向结点
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
			//在链表或树中找到相同key的结点，按需要覆盖
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
		//如果表中元素的数量大于阈值，扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

### get ###
- 根据key，在桶中的链表或者红黑树中寻找对应的结点，返回结点的value。
```java
    public V get(Object key) {
        Node<K,V> e;
		//计算key对应的hash
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
		//hash索引的桶有结点，则开始查找需要的结点
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
			//检查第一个结点是否符合要求
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
				//第一个结点为红黑树结点
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
				//第一个结点为链表结点
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

### remove ###
- 根据key，在桶中的链表或者红黑树中寻找结点node，并删除该结点。
```java
    public V remove(Object key) {
        Node<K,V> e;
		//计算hash
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
		//如果桶中有结点，开始查找结点node
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
			//结点为第一个结点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
				//结点在红黑树中
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
				//结点在链表中
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
			//结点存在且可删除，则删除
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
				//结点为红黑树结点
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
				//结点为头结点
                else if (node == p)
                    tab[index] = node.next;
				//结点为一般结点
                else
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```

## 内部方法 ##
### hash ###
- 对key的hashcode在进行hash，将高位和低位混合，保留高位特征，**增加低位的随机性**，从而使得hashmap的容积较小时，**分布较为均匀**。
```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

### resize ###
- 使用新的大容量数组替代已有的小数组。结点在新的数组中的索引由hash值新增的bit是0还是1决定。
```java
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
			//旧数组的长度达到最大容积，2^30，则不再扩容
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
			//新容积和新阈值加倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
		//使用阈值作为新的容积
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
		//使用默认的容积和阈值
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
		//如果没有新的阈值，则根据容积和参数计算
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
		//创建新的数组
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
		//将旧的数组中的数据转移到新的数组中去
        if (oldTab != null) {
			//遍历原数组
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
					//桶中只有一个结点，计算hash对应的新桶的索引，将该结点放入新桶
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
					//红黑树结点的转移
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
					//链表结点的转移
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
							//查看新增的hash值是0还是1，如果是0，在新桶使用原索引，如果是1，在新桶中使用原索引+oldCap
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
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

## 相似数据结构 ##
### 继承关系 ###
- ![](https://tech.meituan.com/img/java-hashmap/java.util.map%E7%B1%BB%E5%9B%BE.png)

### 介绍 ###
1. HashMap：它根据键的hashCode值存储数据，大多数情况下可以直接定位到它的值，因而具有很快的访问速度，但遍历顺序却是不确定的。 HashMap最多**只允许一条记录的键为null**，允许多条记录的值为null。HashMap**非线程安全**，即任一时刻可以有多个线程同时写HashMap，可能会导致数据的不一致。如果需要满足线程安全，可以用 Collections的synchronizedMap方法使HashMap具有线程安全的能力，或者使用ConcurrentHashMap。
2. Hashtable：Hashtable是遗留类，很多映射的常用功能与HashMap类似，不同的是它承自Dictionary类，并且是**线程安全**的，任一时间只有一个线程能写Hashtable，并发性不如ConcurrentHashMap，因为ConcurrentHashMap引入了分段锁。Hashtable不**建议在新代码中使用**，不需要线程安全的场合可以用HashMap替换，需要线程安全的场合可以用ConcurrentHashMap替换。
3. LinkedHashMap：LinkedHashMap是HashMap的一个子类，**保存了记录的插入顺序**，在用Iterator遍历LinkedHashMap时，先得到的记录肯定是先插入的，也可以在构造时带参数，按照访问次序排序。
4. TreeMap：TreeMap实现SortedMap接口，能够把它**保存的记录根据键排序**，默认是按键值的升序排序，也可以指定排序的比较器，当用Iterator遍历TreeMap时，得到的记录是排过序的。如果使用排序的映射，建议使用TreeMap。在使用TreeMap时，key必须实现Comparable接口或者在构造TreeMap传入自定义的Comparator，否则会在运行时抛出java.lang.ClassCastException类型的异常。
5. 对于上述四种Map类型的类，要求映射中的**key是不可变对象**。不可变对象是该对象在**创建后它的哈希值不会被改变**。如果对象的哈希值发生变化，Map对象很可能就定位不到映射的位置了。

## 参考 ##
- [Java 8系列之重新认识HashMap](https://tech.meituan.com/java-hashmap.html)
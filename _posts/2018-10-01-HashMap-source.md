---
layout: post
title:  HashMap源码分析
date:   2018-10-01 08:09:34 +0800
categories: java
tag: java
---

* content
{:toc}


## 前言

首先声明一点，本文源码分析使用的jdk版本为jdk8。相对于jdk7，HashMap在jdk8中的实现有所不同，采用的整体结构为数组+链表+红黑树。
在jdk8之前的版本，HashMap的版本为数组+链表，在hash相同的情况下，就会把hash值相同的节点放在一个“桶”中，桶的实现为链表，如果桶中的元素很多，查找元素的时间复杂度就会很高，达到O(N)，很影响性能。
jdk8及之后的版本对此进行了改进，在桶中元素达到了某个阈值时会将链表转换为红黑树的结构，从而提升其查找效率，时间复杂度相对低很多，达到O(logN）。

## HashMap中的静态常量

1. `static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;` -- 默认初始容量，16。容量必须是2的N次方
2. `static final int MAXIMUM_CAPACITY = 1 << 30;` -- 最大容量，2的30次方
3. `static final float DEFAULT_LOAD_FACTOR = 0.75f;` -- 默认负载因子。负载因子为扩容所使用，默认为0.75。当数组元素个数/负载因子>容量时，就会对容量进行扩容，扩容策略为当前容量乘以2。
4. `static final int TREEIFY_THRESHOLD = 8;` -- 链表转为红黑树的阈值，链表的长度达到8时，需要将链表转换为红黑树，以提升查询效率
5. `static final int UNTREEIFY_THRESHOLD = 6;` -- 红黑树转为链表的阈值，红黑树的节点<=6个时进行转化
6. `static final int MIN_TREEIFY_CAPACITY = 64;` -- 链表转红黑树的条件中，除了TREEIFY_THRESHOLD外，还需要数组的长度至少达到64。

## 构造方法

```java
public HashMap(int initialCapacity, float loadFactor) {
	if (initialCapacity < 0) // 初始容量不能小于0
		throw new IllegalArgumentException("Illegal initial capacity: " +
										   initialCapacity);
	if (initialCapacity > MAXIMUM_CAPACITY) // 初始容量不能比最大容量大，如果比最大容量大，则初始容量就是最大容量
		initialCapacity = MAXIMUM_CAPACITY;
	if (loadFactor <= 0 || Float.isNaN(loadFactor)) // 负载因子必须为整数，且必须是数字。比如0/0也算是Float，但是不是一个数字
		throw new IllegalArgumentException("Illegal load factor: " +
										   loadFactor);
	this.loadFactor = loadFactor; // 设置负载因子
	this.threshold = tableSizeFor(initialCapacity); // 将初始容量向上取整为2的N次次方
}
```

## tableSizeFor

代码如下：

```java
static final int tableSizeFor(int cap) {
	int n = cap - 1;
	n |= n >>> 1;
	n |= n >>> 2;
	n |= n >>> 4;
	n |= n >>> 8;
	n |= n >>> 16;
	return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

### cap为27时，对tableSizeFor的推算（非2的N次方）

该方法写得比较牛逼，返回的结果为“比给定参数大或等于的最小的2的N次方”。至于为什么要无符号右移并按位或，笔者也不是很清楚。但是我们可以尝试着对此方法进行举例验证。

假如cap为27，二进制为11011。
n = cap - 1; // n为11010
n |= n >>> 1; // n >>> 1为01101，和11010按位或，n为11111
n |= n >>> 2; // n >>> 2为00111，和11111按位或，n为11111
n |= n >>> 4; // n >>> 4为00001，和11111按位或，n为11111
n |= n >>> 8; // n >>> 8为00000，和11111按位或，n为11111
n |= n >>> 16; // n >>> 16为00000，和11111按位或，n为11111
(n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1; // n比0大，且比最大容量小，所以n+1，即：11111+1 = 100000 = 32

另外，“比给定参数大或等于的最小的2的N次方”可以使用取模的方式来实现，为什么该方法不使用这种方式来做呢？我想的是取模对于现代cpu来说可能效率比按位操作效率低好几个数量级，所以一切还是为了性能啊。

### cap为16时，对tableSizeFor的推算（2的N次方），且初始减一，算出的结果为16

cap = 10000

n = cap – 1; // n为01111
n |= n >>> 1; // n >>> 1为00111，和01111按位或，n为01111
n |= n >>> 2; // n >>> 2为00011，和01111按位或，n为01111
n |= n >>> 4; // n >>> 4为00000，和01111按位或，n为01111
n |= n >>> 8; // n >>> 8为00000，和01111按位或，n为01111
n |= n >>> 16; // n >>> 16为00000，和01111按位或，n为01111
(n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1; // n比0大，且比最大容量小，所以n+1，即：01111+1 = 10000 = 16

### cap为16时，对tableSizeFor的推算(2的N次方)，且初始不减一。

结论：如果最初没有进行减一操作，算出的结果为32，和我们期望不同。

cap = 10000

// n = cap – 1; // n为01111
n |= n >>> 1; // n >>> 1为01000，和10000按位或，n为11000
n |= n >>> 2; // n >>> 2为00110，和11000按位或，n为11110
n |= n >>> 4; // n >>> 4为00001，和11110按位或，n为11111
n |= n >>> 8; // n >>> 8为00000，和11111按位或，n为11111
n |= n >>> 16; // n >>> 16为00000，和11111按位或，n为11111
(n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1; // n比0大，且比最大容量小，所以n+1，即：11111+1 = 100000 = 32

## 拷贝构造方法

```java
public HashMap(Map<? extends K, ? extends V> m) {
	this.loadFactor = DEFAULT_LOAD_FACTOR; // 负载因子为默认负载因子，0.75
	putMapEntries(m, false); // 将原map里面的元素一个个地取出来，放到当前map
}
```

### putMapEntries

```java
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
	int s = m.size();
	if (s > 0) { // 被拷贝map必须要有元素，才进行拷贝。没有元素的话，不需要拷贝。
		if (table == null) { // pre-size，当前map没有被初始化，需要先初始化。
			float ft = ((float)s / loadFactor) + 1.0F; // 根据原map的容量和当前map的负载因子，计算出当前map需要的容量大小，为什么要+1呢？原因在于这儿有除法，可能有精度损失，我们+1是为了将可能损失的精度给收上来
			int t = ((ft < (float)MAXIMUM_CAPACITY) ?
					 (int)ft : MAXIMUM_CAPACITY); // 当前map需要的容量可能超过最大容量，这时取最大容量
			if (t > threshold) // 当前map需要的容量比数组阈值更大，则需要重新设置当前map的数组阈值。
				threshold = tableSizeFor(t);
		}
		else if (s > threshold) // 如果容量比阈值更大，直接扩容
			resize();
		for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) { // 将目标map的一个个元素放置到当前map，putVal我们后面会分析
			K key = e.getKey();
			V value = e.getValue();
			putVal(hash(key), key, value, false, evict);
		}
	}
}
```

接下来，我们分析一下HashMap的几个重要方法，例如put、remove、get，这几个方法掌握了，那么HashMap最核心，也最重要的部分就算是掌握了。

## put

```java
/**
 * 如果插入的key存在于map中，则会替代map中key对应的value，并将被替换掉的value返回回来。
 * 主要需要关注hash(key)和putVal。
 */
public V put(K key, V value) {
	return putVal(hash(key), key, value, false, true);
}
```

```java
static final int hash(Object key) {
	int h;
	// key为null时返回0；否则将key的hashCode中的高位和低位进行按位异或操作，那么为什么需要按位异或呢？
	// 官方的解释是为了减少hash冲突的概率。如果不这样做，那么出现如hashCode为2、18、34的等差数据，冲突就会很明显。
	// 但是如果将高位和低位进行按位异或，并将异或的结果填充到低位，那么冲突就会少很多。
	// 官方也给出了一组对比测试数据进行解释。
	return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
			   boolean evict) {
	Node<K,V>[] tab; Node<K,V> p; int n, i;
	// 如果数组为空，或者数组长度为0，则调用resize()进行扩容。扩容后的初始容量为16
	// n为数组的长度
	if ((tab = table) == null || (n = tab.length) == 0)
		n = (tab = resize()).length;
	// 如果当前key的hash不在数组中，则创建新节点，然后放置到数组中
	// 关键代码(n - 1) & hash，n始终为2^N，则n-1始终为1111...的二进制数，对hash进行按位与之后，就相当于生成数组下标
	// 例如：数组容量为16时，n-1的二进制数为1111。1111 & 1011010 = 1010，下标为10
	if ((p = tab[i = (n - 1) & hash]) == null)
		tab[i] = newNode(hash, key, value, null);
	// hash在数组中，则需要做下面3种判断处理
	else {
		Node<K,V> e; K k;
		// 桶的第一个元素的hash值相桶，key也相同
		if (p.hash == hash &&
			((k = p.key) == key || (key != null && key.equals(k))))
			e = p;
		// hash值不同，且桶的第一个元素为树节点，则放入红黑树中
		else if (p instanceof TreeNode)
			e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
		// 否则的话，hash值不同，且桶为链表
		else {
			// 遍历链表
			for (int binCount = 0; ; ++binCount) {
				// 已经遍历完了，没有找到重复的key，则新创建一个节点在链表的末尾
				if ((e = p.next) == null) {
					p.next = newNode(hash, key, value, null);
					// 链表长度超过 8 - 1，则将链表转换为红黑树，转换的方法为：treeifyBin。
					// treeifyBin方法中又加了额外的数组容量判断，当容量大于64时，才转换红黑树，否则进行扩容。
					if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
						treeifyBin(tab, hash);
					// 跳出循环
					break;
				}
				// 找到相同的key，跳出循环
				if (e.hash == hash &&
					((k = e.key) == key || (key != null && key.equals(k))))
					break;
				p = e;
			}
		}
		// HashMap中存在相同的key，会做两件事情
		// 1. 将旧值返回回去
		// 2. 如果是允许值覆盖或者旧值为null，则重复设置新值
		if (e != null) { // existing mapping for key
			V oldValue = e.value;
			if (!onlyIfAbsent || oldValue == null)
				e.value = value;
			afterNodeAccess(e);
			return oldValue;
		}
	}
	// 修改记录数+1，modCount用于并发的判断。
	++modCount;
	// 如果容量超过阈值，则需要对数组进行扩容，并重新hash。
	if (++size > threshold)
		resize();
	// HashMap中什么都没做
	afterNodeInsertion(evict);
	return null;
}
```

我们看下putVal里面比较关键的几个方法，首先是resize()，这个方法可谓是比较长的。

```java
final Node<K,V>[] resize() {
	// 备份旧数组
	Node<K,V>[] oldTab = table;
	// 旧数组的容量，如果旧数组为null，容量为0；否则是旧数组的长度
	int oldCap = (oldTab == null) ? 0 : oldTab.length;
	// 旧数组的阈值
	int oldThr = threshold;
	// 新容量和新阈值默认为0
	int newCap, newThr = 0;
	// 旧容量为0，
	// 1. 如果旧容量是否大于或等于最大允许容量，则阈值直接为整数的最大值
	// 2. 否则，旧容量乘以2则为新容量，且旧容量大于或等于默认初始容量（16）大，则设置新的阈值为旧阈值乘以2
	if (oldCap > 0) {
		if (oldCap >= MAXIMUM_CAPACITY) {
			threshold = Integer.MAX_VALUE;
			return oldTab;
		}
		else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
				 oldCap >= DEFAULT_INITIAL_CAPACITY)
			newThr = oldThr << 1; // double threshold
	}
	// 旧的阈值大于0，初始的时候容量为0且阈值为16，这时直接设置新容量为旧阈值
	else if (oldThr > 0) // initial capacity was placed in threshold
		newCap = oldThr;
	// 新容量使用默认容量（16）
	// 新阈值使用默认容量*默认负载因子（16 * 0.75 = 12）
	else {               // zero initial threshold signifies using defaults
		newCap = DEFAULT_INITIAL_CAPACITY;
		newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
	}
	// 如果新阈值为0，则重新设置新阈值为新容量*负载因子，且不能超过最大整数
	if (newThr == 0) {
		float ft = (float)newCap * loadFactor;
		newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
				  (int)ft : Integer.MAX_VALUE);
	}
	// 将临时变量新阈值设置newThr，赋值为当前类成员变量threshold
	threshold = newThr;
	// 重新创建新数组，长度为新容量
	@SuppressWarnings({"rawtypes","unchecked"})
	Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
	table = newTab;
	// 旧数组不为null，才进行扩容操作
	if (oldTab != null) {
		// 遍历旧数组
		for (int j = 0; j < oldCap; ++j) {
			Node<K,V> e;
			// 旧数组元素不为空，才进行处理，且e为当前节点
			if ((e = oldTab[j]) != null) {
				oldTab[j] = null;
				// e没有下一个节点，说明e在数组中是一个独立的节点，则直接放到新数组指定下标的位置
				// 扩容之后其位置可能不变，也可能是原有位置+旧容量
				// 例如，e的hash为11010，原有容量为32（二进制为10000），原有的数组下标为1010 & 1111 = 1010(二进制) = 10(十进制)
				// 扩容后的数组下标为11010 & 11111 = 11010 = 26
				if (e.next == null)
					newTab[e.hash & (newCap - 1)] = e;
				// 如果e为树节点，则将红黑树进行拆解重新hash，后面我们会详细分析此方法
				else if (e instanceof TreeNode)
					((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
				// 其他情况，则e为链表。
				// 这儿需要注意的是，链表里面的元素需要重新遍历，然后将每个元素进行重新hash。
				// 遍历的结果会将数组元素拆分为两个部分
				// 1. 0 =< idx < oldCap
				// 2. 0 =< oldCap < newCap，即：(oldCap*2)
				// 1的成因. 为什么这么做呢？我们分析一下这行代码：(e.hash & oldCap) == 0，假如oldCap为01000，如果要满足此条件，则e.hash倒数第5位肯定为0，则将其元素放在loHead和loTail形成的链表中
				// 2的成因. 如果不满足(e.hash & oldCap) == 0，则说明e.hash倒数第5位为1，那么将其元素放在hiHead和hiTail形成的链表中
				else { // preserve order
					Node<K,V> loHead = null, loTail = null;
					Node<K,V> hiHead = null, hiTail = null;
					Node<K,V> next;
					do {
						next = e.next;
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
					// 清空loTail.next使其成为单链表，原有位置放入loHead
					if (loTail != null) {
						loTail.next = null;
						newTab[j] = loHead;
					}
					// 清空hiTail.next使其成为单链表，原有位置+旧容量放入hiHead
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

其次，我们分析一下newNode()，比较简单，直接调用Node的有参构造方法进行实例化

```java
Node<K,V> newNode(int hash, K key, V value, Node<K,V> next) {
	return new Node<>(hash, key, value, next);
}
```

其次，我们分析一下treeifyBin()，该方法用于将链表转换为红黑树

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
	int n, index; Node<K,V> e;
	// 数组为空，或者数组容量比64小，则扩容并重新分配hash
	if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
		resize();
	else if ((e = tab[index = (n - 1) & hash]) != null) {
		TreeNode<K,V> hd = null, tl = null;
		do {
			// 将链表中的首元素转换为树节点，使用prev和next引用的方式，将所有元素串起来
			TreeNode<K,V> p = replacementTreeNode(e, null);
			if (tl == null)
				hd = p;
			else {
				p.prev = tl;
				tl.next = p;
			}
			tl = p;
		} while ((e = e.next) != null);
		// 将节点转换为红黑树
		if ((tab[index] = hd) != null)
			hd.treeify(tab);
	}
}
```

最后，我们分析一下split()方法，该方法用于拆分一颗红黑树。

## get
```java
public V get(Object key) {
	Node<K,V> e;
	// 查到节点就返回节点的value，否则返回null
	return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

## getNode

```java
final Node<K,V> getNode(int hash, Object key) {
	Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
	// 必须满足前提条件，才进行查询操作
	// 1. 数组不为null且不为空
	// 2. 数组中存在hash值相同的元素
	if ((tab = table) != null && (n = tab.length) > 0 &&
		(first = tab[(n - 1) & hash]) != null) {
		// hash值相同的第一个元素如果满足key相等，直接返回
		if (first.hash == hash && // always check first node
			((k = first.key) == key || (key != null && key.equals(k))))
			return first;
		if ((e = first.next) != null) {
			// 如果节点类型为红黑树，在红黑树中搜索
			if (first instanceof TreeNode)
				return ((TreeNode<K,V>)first).getTreeNode(hash, key);
			// 否则在链表中搜索
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

## remove

```java
public V remove(Object key) {
	Node<K,V> e;
	// 被移除的节点存在，移除后返回被移除元素的value，否则返回null
	return (e = removeNode(hash(key), key, null, false, true)) == null ?
		null : e.value;
}
```

## removeNode

```java
final Node<K,V> removeNode(int hash, Object key, Object value,
						   boolean matchValue, boolean movable) {
	Node<K,V>[] tab; Node<K,V> p; int n, index;
	// 数组不为空，且数组中存在hash值相同的元素，才进行移除操作
	if ((tab = table) != null && (n = tab.length) > 0 &&
		(p = tab[index = (n - 1) & hash]) != null) {
		Node<K,V> node = null, e; K k; V v;
		// hash值相同的第一个节点，如果key相同，则当前节点就是待删除的节点
		if (p.hash == hash &&
			((k = p.key) == key || (key != null && key.equals(k))))
			node = p;
		else if ((e = p.next) != null) {
			// 如果hash值相同的第一个节点为红黑树节点，则直接在红黑树中去查找待删除的节点
			if (p instanceof TreeNode)
				node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
			// 否则是一个链表，以遍历链表的方式来查找
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
		// 查找到了待删除节点
		// 1. 如果是红黑树节点，从红黑树中删除
		// 2. 如果是hash值相同的第一个节点，直接通过设置数组元素的指针为第二个节点即可
		// 3. 如果是链表中的非首节点，则将被删除节点的上一个节点的next指针指向被删除节点的下一个元素
		// 4. HashMap的元素个数减一
		if (node != null && (!matchValue || (v = node.value) == value ||
							 (value != null && value.equals(v)))) {
			if (node instanceof TreeNode)
				((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
			else if (node == p)
				tab[index] = node.next;
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

## clear

```java
public void clear() {
	Node<K,V>[] tab;
	modCount++;
	if ((tab = table) != null && size > 0) {
		size = 0;
		// 直接将数组中的每个元素设置为null即可，非常简单
		for (int i = 0; i < tab.length; ++i)
			tab[i] = null;
	}
}
```

## containsKey

```java
// 看得出来是调用的getNode方法，前面已经分析过了
public boolean containsKey(Object key) {
	return getNode(hash(key), key) != null;
}
```

## containsValue

```java
public boolean containsValue(Object value) {
	Node<K,V>[] tab; V v;
	if ((tab = table) != null && size > 0) {
		// 遍历每一个数组元素
		for (int i = 0; i < tab.length; ++i) {
			// 对每个数组元素使用next指针来遍历，可以看出这个方法会对每个节点都遍历一次，数据量一多，则效率会非常低
			for (Node<K,V> e = tab[i]; e != null; e = e.next) {
				if ((v = e.value) == value ||
					(value != null && value.equals(v)))
					return true;
			}
		}
	}
	return false;
}
```
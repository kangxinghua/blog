---
title: 深入理解HashMap
date: 2018-09-19 21:46:03
tags: 笔记
---
首先介绍一下Hash表:
# 哈希表定义
Hash,一般翻译做"散列",也有直译为"哈希"的,就把任意长的输入(又叫做预映射,pre-image),通过散列算法,变换成固定长度输出,该输出就是散列值.

**这种转换是一种压缩映射,**也就是, 散列值的空间通常远小于输入的空间,不同的输入可能会散列成相同的输出,而不可能从散列值来唯一的确定输入值.**简单的说就是一种将任意长度的消息压缩到某一固定长度的消息摘要的函数.**

<!-- more -->

Hash主要用于信息安全领域中加密算法,它把一些不同长度的信息转化成杂乱的128位的编码,这些编码值叫做Hash值.也可以说.Hash就是找到一种数据内容和数据地址之间的映射关系例如字符串hello的哈希算法

``` java
char value = "hello"; int key = (((((((27 (int)'h'+27) (int)'e') + 27) 　 (int)'l') + 27)  (int)'l' +27)  27 ) + (int)'o' ; 
```
## 接着说一下HashMap

我想大家对于HashMap并不陌生,程序员基本上使用HashMap.估计很多公司面试的时候都会聊起HashMap,既然HashMap这么重要,我们就一起谈谈这个HashMap.

- HashMap概述及实现原理
- HashMap的数据结构
- HashMap的几个关键属性
- HashMap的存取实现
- fail-fast策略
- Hash冲突以及如何解决Hash冲突
- HashMap和HashTable区别

# HashMap概述
> 定义:HashMap实现了Map接口,继承自AbstractMap.其中Map接口定义了键映射到值得规则,而Abstract类已经实现了Map,这里标注Map 这样应该是更加清晰
``` java
public class HashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable{

}
```
## 特点

数组特点:
> 寻址容易,插入和删除困难.

链表得特点:
> 寻址困难,插入和删除容易.

**实现原理** 简单得说,HashMap得key做Hash算法,并将Hash值映射到内存地址,直接取得key对应的value.

HashMap的高性能需要保证以下几点:
- 将key Hash的算法必须是高效的
- Hash值映射到内存地址(数组索引)的算法是快速的
- 根据内存地址(数组索引)可以直接取得对应的值

## HashMap的数据结构
HashMap的数据结构:在Java语言中,最基本的数据结构就两种,一种是数组,另一种是模拟指针(引用),所有的数据结构都可以使用这两种数据结构构造,HashMap也是可以这样的.HashMap其实就是链表散列,是数组和链表的结合体.图下:
![](687474703a2f2f696d672e6d792e6373.png)

观察HashMap的结构图,我们了解到HashMap底层是一个数组,数组中每一项是一个链表.HashMap提供了三个构造函数:
- HashMap()是一个默认的构造器,初始容量16,负载因子为0.75.
- HashMap(int initialCapacity)是一个指定初始容量为initialCapacity,负载因子为0.75的空的HashMap.
- HashMap(int initialCapacity,float loadFactor)是一个指定初始容量为initialCapacity,负载因子为loadFactor的空的HashMap.

我们查看一下HashMap的初始化源码:

``` java
public HashMap() {
    this(DEFAULT_INITIAL_CAPACITY,DEFAULT_LOAD_FACTOR);
}
  
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +loadFactor);
        this.loadFactor = loadFactor;
        threshold = initialCapacity;
        init();
}
```

## HashMap的几个关键属性
1. initialCapacity
    - initialCapacity为HashMap的最大容量,也就是底层数组的长度.
2. 加载因子loadFactor
    - loadFactor为加载因子,即散列表的实际元素数目(n)/散列表的容量(m).另外,laodFactor越大,存储长度越小,查询时间越长.loadFactor越小,存储长度越大,查询时间短.HashMap默认的是0.75负载因子衡量的是一个散列表的空间的使用程度,负载因子越大表示散列表的装填程度越高,反之愈小.对于使用链表法的散列表来说,查找一个元素的平均时间是O(1+a).

## HashMap的存取实现
1. 存储
    我们先看看HashMap的put()方法的源码:
    ``` java
    public V put(K key, V value) {
        // 如果table为null,inflate 该table
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        // 当key为null,调用putForNullKey方法,保存null与table第一个位    置中,这是HashMap允许为null的原因
        if (key == null)
            return putForNullKey(value);
        // 根据key的hashcode进行计算hash值.
        int hash = hash(key);
        // 根据指定hash值在找到对应的table中的索引. 
        int i = indexFor(hash, table.length);
        // 若 i 索引处的 Entry 不为 null,通过循环不断遍历 e 元素的下    一个元素.
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            // 判断该条链上是否有hash值相同的(key相同)
            // 若存在相同,则直接覆盖value,返回旧value
            if (e.hash == hash && ((k = e.key) == key ||    key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        // 如果i索引处的Entry为null,表明此处还没有Entry.
        modCount++;
        // 将key、value添加到i索引处.
        addEntry(hash, key, value, i);
        return null;
    }
    ```
    分析了上面put()方法源代码中可以看出:
    - 当我们向HashMap中put元素的时候,先根据key的HashCode的值计算    Hash值,根据Hash值得到这个元素在数组中的位置(即下标).
    - 如果数组该位置上已经存放有其他元素了,那么在这个位置上的元素将 以链表的形式存放,新加入的放在链头,最先加入的放在链尾.
    - 如果数组该位置上没有元素,就直接将该元素放到此数组中的该位置上 .
    addEntry(hash, key, value, i)方法根据计算出的hash值,将key-value 对放在数组table的i索引处.addEntry 是 HashMap 提供的一个包访问权限    的方法,代码如下:

    ``` java
    void addEntry(int hash, K key, V value, int bucketIndex) {
     // 如果 Map 中的 key-value 对的数量超过了极限
        if ((size >= threshold) && (null!=table[bucketIndex])){
            // 把 table 对象的长度扩充到原来的2倍. 
            resize(2 * table.length);
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }
        createEntry(hash, key, value, bucketIndex);
    }

    void createEntry(int hash, K key, V value, int bucketIndex) {
        // 根据bucketIndex 获取对应的 Entry  
        Entry<K,V> e = table[bucketIndex];
        // 将新创建的 Entry 放入 bucketIndex 索引处,并让新的 Entry  指向原来的 Entry 
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        size++;
    }

    static int hash(int h) {
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
    }
    ```
    Hash方法,该方法为一个纯粹的数学计算,就是计算h的Hash值.我们知道对于HashMap的table而言,数据分布需要均匀(最好每项都只有一个元素,只有就可以直接找到了),不能太紧也不能太松,太紧导致查询速度慢,太松则浪费空间.计算Hash值后,怎么才能保证table元素分布均匀呢?我们会想到取模,但是由于取模的消耗大,HashMap是这样处理的:
    调用indexFor方法
    ``` java
    static int indexFor(int h, int length) {
        return h & (length-1);
    }
    ```
    HashMap的底层数组总是2的n次方,在构造函数中存在:capacity <<= 1;这样做总是能够保证HashMap的底层数组的长度为2的n次方.当length为2的n次方时候,h&(lenth-1)就相当于对length取模,而且速度比直接取模快的多,这是HashMap在速度上的一种优化.至于为什么是2的n次方下面解释.我们回到indexFor方法,该方法仅有一条语句:h&(length-1),这句话除了上面的取模运算外还有一个非常重要的责任:均匀分布table数据和充分利用空间.

    接下来我们以Key值为“apple”的例子来演示这个过程:
    > 计算“apple”的hashcode,结果为十进制的3029737,二进制的101110001110101110 1001
    > HashMap默认初始长度是16,计算HashMap.Length-1的结果为十进制的15,二进制的1111
    > 把以上两个结果做 与运算,101110001110101110 1001 & 1111 = 1001,十进制是9,所以 index=9
    
    可以看出来,hash算法得到的index值完全取决与Key的HashCode的最后几位.这样做不但效果上等同于取模运算,而且大大提高了效率.

    那么回到最初的问题,初始长度为什么是16或者2的次幂?如果不是会怎么样?

    我们假设HaspMap的初始长度为10,重复前面的运算步骤:    
    ![](7347393-a9962c5fb92eb958.png)

    单独看这个结果,表面上并没有问题.我们再来尝试一个新的HashCode 101110001110101110 1011:
    ![](7347393-5a4b6c0a2fc4eef0.png)

    然后我们再换一个HashCode 101110001110101110 1111 试试:
    ![](7347393-e37f9d5a943823b1.png)

    这样我们可以看到,虽然HashCode的倒数第二第三位从0变成了1,但是运算的结果都是1001.也就是说,当HashMap长度为10的时候,有些index结果的出现几率会更大,而有些index结果永远不会出现（比如0111）  

    所以这样显然不符合Hash算法均匀分布的原则.

    而长度是16或者其他2的次幂,Length - 1的值的所有二进制位全为1（如15的二进制是1111,31的二进制为11111）,这种情况下,index的结果就等同于HashCode后几位的值.只要输入的HashCode本身分布均匀,Hash算法的结果就是均匀的.这也是HashMap设计的玄妙之处.

2. 获取
    ``` java
    public V get(Object key) {  
        // 如果key = null时,返回null对应的value值.
        if (key == null)  
            return getForNullKey(); 
        // 根据key的hashcode值做hash获取对应的值
        int hash = hash(key.hashCode()); 
        // 根据指定hash值在找到对应的table中的索引,并根据索引获取该处  的Entry,通过循环不断遍历 e 元素的下一个元素.
        for (Entry<K,V> e = table[indexFor(hash, table.length)];  
            e != null;  
            e = e.next) {  
            Object k;  
            // 判断e元素的hash与hash是否相等,如果相等并且e元素与key    相等则返回e的原则的value
            if (e.hash == hash && ((k = e.key) == key ||    key.equals(k)))  
                return e.value;  
        }  
        // 如果指定hash值在找到对应的table中的索引,并根据索引获取该处  的Entry的为null,则返回null.
        return null;  
    }  
    ```
    从get()的源代码中可以看出:从HashMap中get元素时,首先计算key的hashCode,通过IndexFor(hash,table.length)找到数组中对应位置的某一元素,然后通过key的equals方法在对应位置的链表中找到需要的元素.
    >总结：HashMap 在底层将 key-value 当成一个整体进行处理,这个整体就是 Entry 对象.HashMap 底层采用一个 Entry[] 数组来保存所有的 key和value 键值对,当需要存储一个 Entry 对象时,会根据hash算法来决定其在数组中的存储位置,然后根据equals方法决定其在该数组位置上的链表中的存储位置；同样的当我们需要取出一个Entry时,也会根据hash算法找到其在数组中的存储位置,再根据equals方法从该位置上的链表中取出该Entry.

## fail-fast策略
> fail-fast策略：我们知道HashMap不是线程安全的,如果我们在使用迭代器过程中其他线程更改了该map,就会抛出ConcurrentModificationException,这就是所谓fail-fast策略.

那么这个fail-fast策略是如何实现的呢？ 这个策略主要是通过modCount的这个值实现的,modCount顾名思义就是HashMap的修改次数.每次在HashMap的内容被修改都会增加这个值,那么在HashMap的迭代器被初始化的都会将这个值赋值给expectedModCount.

``` java
HashIterator() {
    expectedModCount = modCount;
    if (size > 0) { // advance to first entry
        Entry[] t = table;
        while (index < t.length && (next = t[index++]) == null);
    }
}
```

在HashMap的迭代器执行的过程中,代码会判断modCount和expectedModCount的值,如果不相等则表示HashMap的其他线程修改.

``` java
public final boolean hasNext() {
    return next != null;
}

final Entry<K,V> nextEntry() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
    Entry<K,V> e = next;
    if (e == null)
        throw new NoSuchElementException();

    if ((next = e.next) == null) {
        Entry[] t = table;
        while (index < t.length && (next = t[index++]) == null);
    }
    current = e;
    return e;
}
```
## Hash冲突以及如何解决hash冲突

**疑问：如果两个key通过hash % Entry[].length得到的index相同,会不会有覆盖的危险？**

这里 HashMap里面用到链式数据结构的一个概念.上面我们提到过 Entry 类里面有一个 next属性,作用是指向下一个 Entry.打个比方, 第一个键值对A进来,通过计算其 key的 hash得到的 index=0, 记做: *Entry[0] = A.*一会后又进来一个键值对B,通过计算其 index也等于0,现在怎么办？ HashMap会这样做 :*B.next = A,Entry[0] = B,*如果又进来C,index也等于0,那么C.next = B,Entry[0] = C；这样我们发现index=0的地方其实存取了A,B,C三个键值对,他们通过next这个属性链接在一起.所以疑问不用担心.

当然HashMap里面也包含一些优化方面的实现,这里也啰嗦一下. 比如：Entry[]的长度一定后,随着map里面数据的越来越长,这样同一个index的链就会很长,会不会影响性能？HashMap里面设置一个因素（也称为因子）,随着map的size越来越大,Entry[]会以一定的规则加长长度.

## hash冲突解决办法

``` java
public V put(K key, V value) {  
    if (key == null)  
           return putForNullKey(value);  
    int hash = hash(key.hashCode());  
    int i = indexFor(hash, table.length);  
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {  
        Object k;  
        //判断当前确定的索引位置是否存在相同hashcode和相同key的元素,如果存在相同的hash值和相同的key的元素,那么新值覆盖原来的旧值,并返回旧值.  
        //如果存在相同的hash值,那么他们确定的索引位置就相同,这时判断他们的key是否相同,如果不相同,这时就是产生了我们常说的hash冲突.  
        //Hash冲突后,那么HashMap的单个bucket里存储的不是一个 Entry,而是一个 Entry 链.  
        //系统只能必须按顺序遍历每个Entry,直到找到想搜索的 Entry 为止——如果恰好要搜索的Entry位于该Entry链的最末端（该 Entry 是最早放入该 bucket 中） 
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {  
            V oldValue = e.value;  
            e.value = value;  
            return oldValue;  
            }  
        }  
        modCount++;  
        addEntry(hash, key, value, i);  
        return null;  
}  
```

HashMap里面的bucket出现了单链表的形式,散列表要解决的一个问题就是散列值的冲突问题,通常是两种方法：链表法和开放地址法.链表法就是将相同hash值的对象组织成一个链表放在hash值对应的槽位；开放地址法是通过一个探测算法,当某个槽位已经被占据的情况下继续查找下一个可以使用的槽位.java.util.HashMap采用的链表法的方式,链表是单向链表.形成单链表的核心代码如下：

``` java
void addEntry(int hash, K key, V value, int bucketIndex) {
 // 如果 Map 中的 key-value 对的数量超过了极限
    if ((size >= threshold) && (null!=table[bucketIndex])){
        // 把 table 对象的长度扩充到原来的2倍. 
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }
    createEntry(hash, key, value, bucketIndex);
}
    
void createEntry(int hash, K key, V value, int bucketIndex) {
    // 根据bucketIndex 获取对应的 Entry  
    Entry<K,V> e = table[bucketIndex];
    // 将新创建的 Entry 放入 bucketIndex 索引处,并让新的 Entry 指向原来的 Entry 
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```

上面方法的代码大家可以看出,系统总是将新添加的 Entry 对象放入 table 数组的 bucketIndex 索引处——如果 bucketIndex 索引处已经有了一个 Entry 对象,那新添加的 Entry 对象指向原有的 Entry 对象（产生一个 Entry 链）,如果 bucketIndex 索引处没有 Entry 对象,也就是上面程序代码的 e 变量是 null,也就是新放入的 Entry 对象指向 null,也就是没有产生 Entry 链. HashMap里面没有出现hash冲突时,没有形成单链表时,HashMap查找元素很快,get()方法能够直接定位到元素,但是出现单链表后,单个bucket 里存储的不是一个 Entry,而是一个 Entry 链,系统只能必须按顺序遍历每个 Entry,直到找到想搜索的 Entry 为止——如果恰好要搜索的 Entry 位于该 Entry 链的最末端（该 Entry 是最早放入该 bucket 中）,那系统必须循环到最后才能找到该元素.

## HashMap和HashTable区别
> HashMap和HashTable采用的是相同的存储机制,因此两者的实现基本一致.

**不同的是:**

> HashMap可以key和value均可以为null,而HashTable则不可以.HashTable不允许null的值,HashTable的key为null的时候,HashTable调用put方法时,直接抛出NullPointerException.其它细微的差别还有,比如初始化Entry数组的大小等等.
> HashTable是线程安全的,内部的方法基本都是synchronized.hashmap则不是线程安全的.
> HashTable中的hash数组默认是11,增加方式old*2+1.hashmap中hash数组的默认大小是16,而且一定是2的指数
> HashTable有一个contains(Object value),功能和containsValue(Object value)功能一样
> HashTable使用Enumeration,HashMap使用Iterator

**以上只是表面的不同,它们的实现也有很大的不同.**
> HashTable中hash数组默认大小是11,增加的方式是 old*2+1.HashMap中hash数组的默认大小是16,而且一定是2的指数.
> 哈希值的使用不同,HashTable直接使用对象的hashCode,代码是这样的:

``` java
int hash = key.hashCode();

int index = (hash & 0x7FFFFFFF) % tab.length;

而HashMap重新计算hash值,而且用与代替求模：

int hash = hash(k);

int i = indexFor(hash, table.length);

static int hash(Object x) {

　　int h = x.hashCode();

　　h += ~(h << 9);

　　h ^= (h >>> 14);

　　h += (h << 4);

　　h ^= (h >>> 10);

　　return h;

}

static int indexFor(int h, int length) {

　　return h & (length-1);

}
```
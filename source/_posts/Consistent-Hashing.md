---
title: 一致性哈希算法的理解与实践(Java)
date: 2016-10-26 22:41:42
tags: 笔记
---
> 转载 [一致性哈希算法的理解与实践](http://yikun.github.io/2016/06/09/%E4%B8%80%E8%87%B4%E6%80%A7%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E7%9A%84%E7%90%86%E8%A7%A3%E4%B8%8E%E5%AE%9E%E8%B7%B5/) 修改了Python->Java
## 概述
在维基百科中，是这么定义的
> 一致哈希是一种特殊的哈希算法。在使用一致哈希算法后，哈希表槽位数（大小）的改变平均只需要对 K/n个关键字重新映射，其中K是关键字的数量， n是槽位数量。然而在传统的哈希表中，添加或删除一个槽位的几乎需要对所有关键字进行重新映射。

## 引出
我们在上文中已经介绍了一致性Hash算法的基本优势，我们看到了该算法主要解决的问题是：当slot数发生变化时，能够尽量少的移动数据。那么，我们思考一下，普通的Hash算法是如何实现？又存在什么问题呢？
那么我们引出一个问题：
> 假设有1000w个数据项，100个存储节点，请设计一种算法合理地将他们存储在这些节点上。  

看一看普通Hash算法的原理：
![](/images/fe155f98-3a5e-11e6-834d-193e6f85afcd.png)

算法的核心计算如下

``` java
import java.util.Arrays;
import java.util.Collections;
import java.util.UUID;

/**
 * Created by KangXinghua on 2016/10/26.
 */
public class NormalHash {
    static Integer ITEMS = 10000000;  
    static Integer NODES = 100;

    static Integer[] node_stat = new Integer[NODES];

    public static void main(String[] args) {
        for (Integer i = 0; i < ITEMS; i++) {
            int n = Math.abs(UUID.randomUUID().hashCode() % NODES);
            if (node_stat[n] == null)
                node_stat[n] = 0;
            node_stat[n] += 1;
        }

        int _ave = ITEMS / NODES;
        int _max = Collections.max(Arrays.asList(node_stat));
        int _min = Collections.min(Arrays.asList(node_stat));
        System.out.printf("Ave: %d%n", _ave);
        System.out.printf("Max: %d\t(%.2f%%)%n", _max, (_max - _ave) * 100.0 / _ave);
        System.out.printf("Min: %d\t(%.2f%%)%n", _min, (_ave - _min) * 100.0 / _ave);
    }
}
```


> Ave: 100000
> Max: 100772	(0.77%)
> Min: 99311	(0.69%)
> 每次运行结果不一样

从上述结果可以发现，普通的Hash算法均匀地将这些数据项打散到了这些节点上，并且分布最少和最多的存储节点数据项数目小于1%。之所以分布均匀，主要是依赖Hash算法能够比较随机的分布。

然而，我们看看存在一个问题，由于该算法使用节点数取余的方法，强依赖node的数目，因此，当是node数发生变化的时候，item所对应的node发生剧烈变化，而发生变化的成本就是我们需要在node数发生变化的时候，数据需要迁移，这对存储产品来说显然是不能忍的，我们观察一下增加node后，数据项移动的情况：
``` java
import java.util.UUID;

/**
 * Created by KangXinghua on 2016/10/26.
 */
public class NormalHashAdd {
    static Integer ITEMS = 10000000;
    static Integer NODES = 100;
    static Integer NEW_NODES = 101;

    static Integer change = 0;

    public static void main(String[] args) {
        for (Integer i = 0; i < ITEMS; i++) {
            int h = Math.abs(UUID.randomUUID().hashCode());

            int n = Math.abs(h % NODES);
            int n_new = Math.abs(h % NEW_NODES);

            if (n_new != n)
                change += 1;
        }

        System.out.printf("Change: %d\t(%.2f%%)%n", change, change * 100.0 / ITEMS);
    }
}
```

> Change: 9900962	(99.01%)
> 每次运行结果不一样

翻译一下就是，**如果有100个item，当增加一个node，之前99%的数据都需要重新移动。**  

这显然是不能忍的，普通哈希算法的问题我们已经发现了，如何对其进行改进呢？没错，我们的一致性哈希算法闪亮登场。  

## 登场

我们上节介绍了普通Hash算法的劣势，即当node数发生变化（增加、移除）后，数据项会被重新“打散”，导致大部分数据项不能落到原来的节点上，从而导致大量数据需要迁移。
那么，一个亟待解决的问题就变成了：当node数发生变化时，如何保证尽量少引起迁移呢？即**当增加或者删除节点时，对于大多数item，保证原来分配到的某个node，现在仍然应该分配到那个node，将数据迁移量的降到最低。**  

一致性Hash算法的原理是这样的：

![](/images/0e8fea32-3a5f-11e6-84b5-ff101495cf49.png)

``` java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.UUID;

/**
 * Created by KangXinghua on 2016/10/27.
 */
public class ConsistHashAdd {
    static Integer ITEMS = 10000000;
    static Integer NODES = 100;
    static Integer NEW_NODES = 101;

    static List<Integer> ring = new ArrayList<>();
    static List<Integer> new_ring = new ArrayList<>();
    static Integer change = 0;

    public static void main(String[] args) {
        initNode(ring, NODES);
        initNode(new_ring, NEW_NODES);
        for (Integer i = 0; i < ITEMS; i++) {
            int h = Math.abs(UUID.randomUUID().hashCode());

            int n = bisectLeft(ring, h) % NODES;
            int n_new = bisectLeft(ring, h) % NEW_NODES;

            if (n_new != n)
                change += 1;
        }

        System.out.printf("Change: %d\t(%.2f%%)%n", change, change * 100.0 / ITEMS);
    }

    public static int bisectLeft(List<Integer> list, Integer key) {
        int idx = Math.min(list.size(), Math.abs(Collections.binarySearch(list, key)));
        while (idx > 0 && list.get(idx - 1) >= key) idx--;
        return idx;
    }

    static void initNode(List<Integer> list, Integer nodes) {
        for (Integer i = 0; i < nodes; i++) {
            int h = Math.abs(UUID.randomUUID().hashCode());
            list.add(h);
        }
        Collections.sort(list);
    }
}
```

我们依然对其进行了实现ConsistHashAdd.java，并且观察了数据迁移的结果：
> Change: 15062	(0.15%) 
> 每次运行结果不一样

虽然一致性Hash算法解决了节点变化导致的数据迁移问题，但是，我们回过头来再看看数据项分布的均匀性，进行了一致性Hash算法的实现ConsistHash.java：
``` java
import java.util.*;

/**
 * Created by KangXinghua on 2016/10/27.
 */
public class ConsistHash {
    static Integer ITEMS = 10000000;
    static Integer NODES = 100;

    static List<Integer> ring = new ArrayList<>();
    static Integer[] node_stat = new Integer[NODES];
    static Map<Integer, Integer> hash2node = new HashMap<>();

    public static void main(String[] args) {
        initNode(ring, NODES);
        for (Integer i = 0; i < ITEMS; i++) {
            int h = UUID.randomUUID().hashCode();

            int n = bisectLeft(ring, h) % NODES;

            if (node_stat[hash2node.get(ring.get(n))] == null)
                node_stat[hash2node.get(ring.get(n))] = 0;
            node_stat[hash2node.get(ring.get(n))] += 1;
        }

        int _ave = ITEMS / NODES;
        int _max = Collections.max(Arrays.asList(node_stat));
        int _min = Collections.min(Arrays.asList(node_stat));
        System.out.printf("Ave: %d%n", _ave);
        System.out.printf("Max: %d\t(%.2f%%)%n", _max, (_max - _ave) * 100.0 / _ave);
        System.out.printf("Min: %d\t(%.2f%%)%n", _min, (_ave - _min) * 100.0 / _ave);
    }

    public static int bisectLeft(List<Integer> list, Integer key) {
        int idx = Math.min(list.size(), Math.abs(Collections.binarySearch(list, key)));
        while (idx > 0 && list.get(idx - 1) >= key) idx--;
        return idx;
    }

    static void initNode(List<Integer> list, Integer nodes) {
        for (Integer i = 0; i < nodes; i++) {
            int h = Math.abs(UUID.randomUUID().hashCode());
            list.add(h);
            hash2node.put(h, i);
        }
        Collections.sort(list);
    }
}
```
> Ave: 100000
> Max: 5004417	(4904.42%)
> Min: 31	(99.97%)
> 每次运行结果不一样
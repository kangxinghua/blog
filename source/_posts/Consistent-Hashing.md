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

import static com.kxh.hash.Utils.hash;

/**
 * Created by KangXinghua on 2016/10/26.
 */
public class NormalHash {

    public static void main(String[] args) {
        Integer ITEMS = 10000000;
        Integer NODES = 100;

        Integer[] node_stat = new Integer[NODES];

        for (Integer i = 0; i < ITEMS; i++) {
            int h = hash(i.toString());
            int n = h % NODES;
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
> Max: 100653   (0.65%)
> Min: 99160    (0.84%)

从上述结果可以发现，普通的Hash算法均匀地将这些数据项打散到了这些节点上，并且分布最少和最多的存储节点数据项数目小于1%。之所以分布均匀，主要是依赖Hash算法能够比较随机的分布。

然而，我们看看存在一个问题，由于该算法使用节点数取余的方法，强依赖node的数目，因此，当是node数发生变化的时候，item所对应的node发生剧烈变化，而发生变化的成本就是我们需要在node数发生变化的时候，数据需要迁移，这对存储产品来说显然是不能忍的，我们观察一下增加node后，数据项移动的情况：
``` java
import static com.kxh.hash.Utils.hash;

/**
 * Created by KangXinghua on 2016/10/26.
 */
public class NormalHashAdd {

    public static void main(String[] args) {
        Integer ITEMS = 10000000;
        Integer NODES = 100;
        Integer NEW_NODES = 101;

        Integer change = 0;

        for (Integer i = 0; i < ITEMS; i++) {
            int h = hash(i.toString());

            int n = h % NODES;
            int n_new = h % NEW_NODES;

            if (n_new != n)
                change += 1;
        }

        System.out.printf("Change: %d\t(%.2f%%)%n", change, change * 100.0 / ITEMS);
    }

}
```

> Change: 9900492   (99.00%)

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

import static org.mycommon.hash.Utils.hash;


/**
 * Created by KangXinghua on 2016/10/27.
 */
public class ConsistHashAdd {

    public static void main(String[] args) {
        Integer ITEMS = 10000000;
        Integer NODES = 100;
        Integer NEW_NODES = 101;

        List<Integer> ring = new ArrayList<>();
        List<Integer> new_ring = new ArrayList<>();
        Integer change = 0;

        for (Integer i = 0; i < NODES; i++) {
            int h = hash(i.toString());
            ring.add(h);
        }
        Collections.sort(ring);

        for (Integer i = 0; i < NEW_NODES; i++) {
            int h = hash(i.toString());
            new_ring.add(h);
        }
        Collections.sort(new_ring);

        for (Integer i = 0; i < ITEMS; i++) {
            int h = hash(i.toString());
            int n = bisectLeft(ring, h) % NODES;
            int n_new = bisectLeft(new_ring, h) % NEW_NODES;

            if (n_new != n)
                change += 1;
        }

        System.out.printf("Change: %d\t(%.2f%%)%n", change, change * 100.0 / ITEMS);
    }

    public static int bisectLeft(List<Integer> list, Integer key) {
        int idx = Math.min(list.size(), Math.abs(Collections.binarySearch(list, key)));
        while (idx > 0 && list.get(idx - 1) >= key) {
            idx--;
        }
        return idx;
    }
}
```

我们依然对其进行了实现ConsistHashAdd.java，并且观察了数据迁移的结果：
> Change: 6379941   (63.80%)

虽然一致性Hash算法解决了节点变化导致的数据迁移问题，但是，我们回过头来再看看数据项分布的均匀性，进行了一致性Hash算法的实现ConsistHash.java：
``` java
import java.util.*;

import static com.kxh.hash.Utils.bisectLeft;
import static com.kxh.hash.Utils.hash;

/**
 * Created by KangXinghua on 2016/10/27.
 */
public class ConsistHash {

    public static void main(String[] args) {
        Integer ITEMS = 10000000;
        Integer NODES = 100;

        List<Integer> ring = new ArrayList<>();
        Integer[] node_stat = new Integer[NODES];
        Map<Integer, Integer> hash2node = new HashMap<>();

        for (Integer i = 0; i < NODES; i++) {
            int h = hash(i.toString());
            ring.add(h);
            hash2node.put(h, i);
        }
        Collections.sort(ring);

        for (Integer i = 0; i < ITEMS; i++) {
            int h = hash(i.toString());
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
}
```
Ave: 100000
Max: 636147 (536.15%)
Min: 2423   (97.58%)

这结果简直是简直了，确实非常结果差，分配的很不均匀。我们思考一下，一致性哈希算法分布不均匀的原因是什么？从最初的1000w个数据项经过一般的哈希算法的模拟来看，这些数据项“打散”后，是可以比较均匀分布的。但是引入一致性哈希算法后，为什么就不均匀呢？数据项本身的哈希值并未发生变化，变化的是判断数据项哈希应该落到哪个节点的算法变了。 

![](/images/8c9e6caa-3a5f-11e6-87ad-fdb462b76aef.png)

因此，主要是因为这100个节点Hash后，在环上分布不均匀，导致了每个节点实际占据环上的区间大小不一造成的。

## 改进-虚节点
当我们将node进行哈希后，这些值并没有均匀地落在环上，因此，最终会导致，这些节点所管辖的范围并不均匀，最终导致了数据分布的不均匀。

![](/images/a0e32fde-3a5f-11e6-969d-085f64220e63.png)

详细实现请见VirtualConsistHash.java

``` java
import java.util.*;

import static com.kxh.hash.Utils.bisectLeft;
import static com.kxh.hash.Utils.hash;

/**
 * Created by KangXinghua on 2016/10/30.
 */
public class VirtualConsistHash {

    public static void main(String[] args) {

        Integer ITEMS = 10000000;
        Integer NODES = 100;
        Integer VNODES = 1000;

        List<Integer> ring = new ArrayList<>();
        Integer[] node_stat = new Integer[NODES];
        Map<Integer, Integer> hash2node = new HashMap<>();

        for (int i = 0; i < NODES; i++) {
            for (int j = 0; j < VNODES; j++) {
                int h = hash(String.valueOf(i) + String.valueOf(j));
                ring.add(h);
                hash2node.put(h, i);
            }
        }
        Collections.sort(ring);

        for (Integer i = 0; i < ITEMS; i++) {
            int h = hash(i.toString());
            int n = bisectLeft(ring, h) % (NODES * VNODES);
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
}

```
输出结果是这样的：
> Ave: 100000
> Max: 117707   (17.71%)
> Min: 9213 (90.79%)

因此，通过增加虚节点的方法，使得每个节点在环上所“管辖”更加均匀。这样就既保证了在节点变化时，尽可能小的影响数据分布的变化，而同时又保证了数据分布的均匀。也就是靠增加“节点数量”加强管辖区间的均匀。
同时，观察增加节点后数据变动情况，详细的代码请见VirtualConsistHashAdd.java：

``` java
import java.util.*;

import static com.kxh.hash.Utils.bisectLeft;
import static com.kxh.hash.Utils.hash;

/**
 * Created by KangXinghua on 2016/10/30.
 */
public class VirtualConsistHashAdd {

    public static void main(String[] args) {

        Integer ITEMS = 10000000;
        Integer NODES = 100;
        Integer NEW_NODES = 101;
        Integer VNODES = 1000;

        List<Integer> ring = new ArrayList<>();
        List<Integer> new_ring = new ArrayList<>();

        Map<Integer, Integer> hash2node = new HashMap<>();
        Map<Integer, Integer> new_hash2node = new HashMap<>();

        for (int i = 0; i < NODES; i++) {
            for (int j = 0; j < VNODES; j++) {
                int h = hash(String.valueOf(i) + String.valueOf(j));
                ring.add(h);
                hash2node.put(h, i);
            }
        }
        Collections.sort(ring);

        for (int i = 0; i < NEW_NODES; i++) {
            for (int j = 0; j < VNODES; j++) {
                int h = hash(String.valueOf(i) + String.valueOf(j));
                new_ring.add(h);
                new_hash2node.put(h, i);
            }
        }
        Collections.sort(new_ring);

        Integer change = 0;
        for (Integer i = 0; i < ITEMS; i++) {
            int h = hash(i.toString());
            int n = bisectLeft(ring, h) % (NODES * VNODES);
            int n_new = bisectLeft(new_ring, h) % (NEW_NODES * VNODES);
            if (hash2node.get(ring.get(n)) != new_hash2node.get(new_ring.get(n_new)))
                change += 1;
        }
        System.out.printf("Change: %d\t(%.2f%%)%n", change, change * 100.0 / ITEMS);
    }
}
```
> Change: 107545    (1.08%)

## 另一种改进
然而，虚节点这种靠数量取胜的策略增加了存储这些虚节点信息所需要的空间。在OpenStack的Swift组件中，使用了一种比较特殊的方法来解决分布不均的问题，改进了这些数据分布的算法，将环上的空间均匀的映射到一个线性空间，这样，就保证分布的均匀性。

![](/images/b01139ec-3a5f-11e6-965a-070f5c4c0afa.png)

代码实现见PartConsistHash.java

``` java
import java.util.*;

import static com.kxh.hash.Utils.bisectLeft;
import static com.kxh.hash.Utils.hash;

/**
 * Created by KangXinghua on 2016/10/27.
 */
public class PartConsistHash {

    public static void main(String[] args) {
        Integer ITEMS = 10000000;
        Integer NODES = 100;
        Integer LOG_NODE = 7;
        Integer MAX_POWER = 31;//这个值与原文不一样。如果是32的话有大部分的节点无法命中
        Integer PARTITION = MAX_POWER - LOG_NODE;
        Integer[] node_stat = new Integer[NODES];

        List<Integer> ring = new ArrayList<>();
        Map<Integer, Integer> part2node = new HashMap<>();

        for (int i = 0; i < Math.pow(2, LOG_NODE); i++) {
            ring.add(i);
            part2node.put(i, i % NODES);
        }

        for (int i = 0; i < NODES; i++) {
            node_stat[i] = 0;
        }

        for (Integer i = 0; i < ITEMS; i++) {
            int h = hash(i.toString()) >> PARTITION;
            int n = bisectLeft(ring, h) % NODES;
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
> Max: 157292   (57.29%)
> Min: 77343    (22.66%)

可以看到，数据分布是比较理想的。如果节点数刚好和分区数相等，理论上是可以均匀分布的。而观察下增加节点后的数据移动比例，代码实现见PartConsistHashAdd.java:
``` java
import java.util.*;

import static com.kxh.hash.Utils.bisectLeft;
import static com.kxh.hash.Utils.hash;

/**
 * Created by KangXinghua on 2016/10/27.
 */
public class PartConsistHashAdd {

    public static void main(String[] args) {
        Integer ITEMS = 10000000;
        Integer NODES = 100;
        Integer NEW_NODES = 101;
        Integer LOG_NODE = 7;
        Integer MAX_POWER = 31;//这个值与原文不一样。如果是32的话有大部分的节点无法命中
        Integer PARTITION = MAX_POWER - LOG_NODE;

        List<Integer> ring = new ArrayList<>();
        Map<Integer, Integer> part2node = new HashMap<>();
        Map<Integer, Integer> new_part2node = new HashMap<>();

        for (int i = 0; i < Math.pow(2, LOG_NODE); i++) {
            ring.add(i);
            part2node.put(i, i % NODES);
            new_part2node.put(i, i % NEW_NODES);
        }

        Integer change = 0;
        for (Integer i = 0; i < ITEMS; i++) {
            int h = hash(i.toString()) >> PARTITION;
            int p = bisectLeft(ring, h);
            int n = part2node.get(p) % NODES;
            int n_new = new_part2node.get(p) % NEW_NODES;
            if (n_new != n)
                change += 1;
        }

        System.out.printf("Change: %d\t(%.2f%%)%n", change, change * 100.0 / ITEMS);
    }
}
```
结果如下所示：
> Change: 2185667   (21.86%)

可以看到，移动也是比较理想的。

## hash和 bisectLeft 代码

``` java
import java.util.Collections;
import java.util.List;

/**
 * Created by KangXinghua on 2016/10/30.
 */
public class Utils {
    public static int hash(String str) {
        final int p = 16777619;
        int hash = (int) 2166136261L;
        for (int i = 0; i < str.length(); i++)
            hash = (hash ^ str.charAt(i)) * p;
        hash += hash << 13;
        hash ^= hash >> 7;
        hash += hash << 3;
        hash ^= hash >> 17;
        hash += hash << 5;

        // 如果算出来的值为负数则取其绝对值
        if (hash < 0)
            hash = Math.abs(hash);
        return hash;
    }

    public static int bisectLeft(List<Integer> list, Integer key) {
        int idx = Math.min(list.size(), Math.abs(Collections.binarySearch(list, key)));
        while (idx > 0 && list.get(idx - 1) >= key) {
            idx--;
        }
        return idx;
    }
}
```

##  写在最后
代码的结果值和原文的不一样的主要原因是hash算法的不一样，原文的hash 是：
``` python
def _hash(value):
    k = md5(str(value)).digest() 
    ha = unpack_from(">I", k)[0]  
    return ha
```

我能力有限，没能翻译过来。
<a href="../相关技术.md"> 上一页 </a>

[TOC]

从接触gvi的概念到使用大概已经一年了，但一直却像隔着层纱看事物一样，让人很不是滋味。
趁着最近抽出时间看底层的代码，惊叹其实现精妙，似有所得，因此记录一下，以便以后复习用。


## 存放
高16位决定要存放在哪个容器里面
底16位用容器存放

### add
>
``` java
public void add(final int x) {
  final short hb = Util.highbits(x);
  final int i = highLowContainer.getIndex(hb);
  if (i >= 0) {
    highLowContainer.setContainerAtIndex(i,
        highLowContainer.getContainerAtIndex(i).add(Util.lowbits(x)));
  } else {
    final ArrayContainer newac = new ArrayContainer();
    highLowContainer.insertNewKeyValueAt(-i - 1, hb, newac.add(Util.lowbits(x)));
  }
}
```
可以看到先用高16位查找要存放在哪个容器里面，即一个int类型被拆分为2个short。
同时上面有一个特殊的下标即“i”，之所有说i很特殊，因为其表示含义很强，即如果
i >= 0则表示该容器已经存在，且表示为容器的下标，索引
如果i <= 0则 (-i-1) 表示新生成的容器要存放的位置。并默认生成一个ArrayContainer
容器来存放低16的数值。
所以关键的算法是找i，以及ArrayContainer的存储方式。
把这查找i算法放一放，先假定容器已经找到，先看看ArrayContainer如何进行存储的。
>

### ArrayContainer
``` java
private static final int DEFAULT_INIT_SIZE = 4;
public ArrayContainer() {
  this(DEFAULT_INIT_SIZE);
}
public ArrayContainer(final int capacity) {
  content = new short[capacity];
}
```
以看出ArrayContainer会默认生成一个4个short类型的数组。
``` java
public Container add(final short x) {
  int loc = Util.unsignedBinarySearch(content, 0, cardinality, x);
  if (loc < 0) {
    // Transform the ArrayContainer to a BitmapContainer
    // when cardinality = DEFAULT_MAX_SIZE
    if (cardinality >= DEFAULT_MAX_SIZE) {
      BitmapContainer a = this.toBitmapContainer();
      a.add(x);
      return a;
    }
    if (cardinality >= this.content.length) {
      increaseCapacity();
    }
    // insertion : shift the elements > x by one position to
    // the right
    // and put x in it's appropriate place
    System.arraycopy(content, -loc - 1, content, -loc, cardinality + loc + 1);
    content[-loc - 1] = x;
    ++cardinality;
  }
  return this;
}
```
先用二分法查找short要放入的位置。这里用loc表示。
如果loc < 0表示x没有被存储过，则用-loc - 1表示要存放的位置。看着就和上面的i相似，但含义远远比上面的i丰富。
如果loc > 0 表示x已经被存储了，所以不用进行额外的操作。
考虑loc < 0 的情况。这里有几点值得注意的
1：loc是怎样获取的。
2：获取loc之后怎么进行操作。
3：short是以什么样的顺序怎样进行存储的。
4：上面说的默认是长度为4的short数组，那么怎样，什么时候进行扩容的。
直接说做法，再分析代码。
如果不考虑性能，存储，最简单粗暴的存放4096个short数最简单傻瓜的方法是
一开始开辟一个长度为4096的数组，然后一开始初始化为-1，添加任意的short
直接往对应的下标存放即可。但是从ArrayContainer一开始只是初始化长度为4的
short数组，明显这种做法是不可能存在代码里面的。
先看下面两个个列子中short数组添加进入容器里面的情况  。
例1：
1，3，5，7
添加4次的情况分别如下

1  |     |   |   |
---| --- |---|----
1 | 3 |   |   |
1 | 3 | 5 |   |
1 | 3 | 5 | 7 |

列子2：
3，6，4，7
容器中的存放情况如下

3  |     |   |   |
---| --- |---|----
3 | 6 |   |   |

第三步

3  |  4   |  6 |   |
---| --- |---|----
3 | 4 | 6  | 7  |

可以看出short数组在容器中是有序的。
如果loc小于0的时候（-loc - 1）表示存放的位置。
同时，像例2中的第三步需要把（-loc - 1）到末尾的数组往后移动一位，这就是
System.arraycopy(content, -loc - 1, content, -loc, cardinality + loc + 1);
这一行代码的含义。因为cardinality表示的是当前最后一个数值的位置，而（-loc-1）表示插入的位置
所以cardinality + loc + 1表示的是要移动多少位。
这也正是loc这个值的巧妙之处。一个返回值有多种用途。
同时从上面的源码中也可以看出在移动，插入具体数值之前，如果容器已经填充完整还需要进行适当的扩容。
并且，如果当前存储的数值个数已经超过DEFAULT_MAX_SIZE（4096）就从ArraryContainer转换成BitmapContainer这个
下面还会分析，先看最重要的一个算法，即找到loc的算法。

#### loc查找算法

``` java
public static final boolean USE_HYBRID_BINSEARCH = true;

public static int unsignedBinarySearch(final short[] array, final int begin, final int end,
    final short k) {
  if (USE_HYBRID_BINSEARCH) {
    return hybridUnsignedBinarySearch(array, begin, end, k);
  } else {
    return branchyUnsignedBinarySearch(array, begin, end, k);
  }
}

protected static int hybridUnsignedBinarySearch(final short[] array, final int begin,
    final int end, final short k) {
  int ikey = toIntUnsigned(k);
  // next line accelerates the possibly common case where the value would
  // be inserted at the end
  if ((end > 0) && (toIntUnsigned(array[end - 1]) < ikey)) {
    return -end - 1;
  }
  int low = begin;
  int high = end - 1;
  // 32 in the next line matches the size of a cache line
  while (low + 32 <= high) {
    final int middleIndex = (low + high) >>> 1;
    final int middleValue = toIntUnsigned(array[middleIndex]);

    if (middleValue < ikey) {
      low = middleIndex + 1;
    } else if (middleValue > ikey) {
      high = middleIndex - 1;
    } else {
      return middleIndex;
    }
  }
  // we finish the job with a sequential search
  int x = low;
  for (; x <= high; ++x) {
    final int val = toIntUnsigned(array[x]);
    if (val >= ikey) {
      if (val == ikey) {
        return x;
      }
      break;
    }
  }
  return -(x + 1);
}
```
可以看到最终还是走二分查找法，亮点我觉得只有两个
一个是(low + high) >>> 1用位运算来获取中位数。
另一个是和最后一个数比较如果大于最后一个数则返回-(end + 1)
这也就是前面说的-loc-1表示插入位置的原因，即-(-(end+1))-1=end即放入最后一个位置。











## 参考文章
1. https://github.com/RoaringBitmap/RoaringBitmap

2. https://blog.csdn.net/xywtalk/article/details/50991738

3. https://www.cnblogs.com/565261641-fzh/p/7686757.html

4. https://www.cnblogs.com/blog-cq/p/5793529.html

5. https://blog.csdn.net/zgrjkflmkyc/article/details/12185143

6. https://blog.csdn.net/zgrjkflmkyc/article/details/12186139
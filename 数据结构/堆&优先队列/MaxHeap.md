# 二叉堆

##  二叉堆的定义

1. 二叉堆是一颗完全二叉树

   完全二叉树：把元素一层一层放，如果每一层都放满，没有空的叶子节点，就是满二叉树，如果有空的叶子节点，那就是完全二叉树。

2. 二叉树中每一个节点的值总是小于父节点的值（最大堆）。如下图所示便是一个最大二叉堆

![](img/二叉堆.png)



## 用数组表示表示堆

如上二叉 堆我们可以用这样一个数组表示，将堆中的元素按顺序依次放入数组中，这样的话，要表示一个节点的父子关系不如定义成树结构，可以用指针方便的表示。但其实我们观察可以发现每一个节点他的父子关系与他在数组中的索引是有一定关系的：

   节点本身的索引：index
   节点的左孩子索引：2 * index
   节点的右孩子索引：2 * index + 1
   节点的父亲节点的索引：index / 2

   ![](img/二叉堆的数组表示.png)



## 向堆中添加元素

![](img/插入元素.png)

向二叉堆中插入元素的过程如上如图所示，首先插入80，发现破坏了最大堆的性质，因此将80和父元素交换位置，交换后，80和父元素71相比，需要继续交换位置，直到满足最大堆的性质。这个过程叫元素的**上浮**

代码实现：

```java
/**
  * 添加元素
  *
  * @param e e
  */
public void insert(E e) {

    if (array.length - 1 == size) {
        enlargeArray(2 * array.length + 1);
    }

    int hole = ++size;
    for (array[0] = e; e.compareTo(array[hole / 2]) > 0; hole /= 2) {
        array[hole] = array[hole / 2];
    }
    array[hole] = e;
}
```



## 从堆中删除元素

![](img/下沉操作.png)

删除元素的过程如图所示：首先删除堆顶元素，然后将最后一个元素挪到堆顶，这个时候破坏了堆的性质，就开始执行**下沉**操作，让堆顶元素和孩子比较，将较大的孩子上移，堆顶元素下沉，直到符合堆的条件为止。

```java
/**
  * 删除堆顶元素
  *
  * @return 堆顶元素
  */
public E deleteMax() {
    E maxItem = findMax();
    array[1] = array[size];
    array[size--] = null;
    siftDown(MAX_INDEX);

    // 当元素个数变成总容量的1/4的时候，释放内存到
    if (size == array.length / 4 && array.length / 2 != 0) {
        enlargeArray(array.length / 2);
    }
    return maxItem;
}

/**
  * 下沉操作
  *
  * @param hole 开始下沉的位置
  */
private void siftDown(int hole) {
    int child = 0;
    E tmp = array[hole];
    for (; hole * 2 <= size; hole = child) {
        child = hole * 2;
        if (child != size && array[child + 1].compareTo(array[child]) > 0) {
            child++;
        }
        if (array[child].compareTo(tmp) > 0) {
            array[hole] = array[child];
        } else {
            break;
        }
    }
    array[hole] = tmp;
}
```



## 完整代码

```java
/**
 * 最大堆
 *
 * @author HXY
 * @param <E>
 */
public class MaxHeap<E extends Comparable<E>> {
    private static final int DEFAULT_CAPACITY = 10;
    private static final int MAX_INDEX = 1;
    private int size;
    private E[] array;

    /**
     * 构造函数，构造指定容量的最大堆
     */
    public MaxHeap(int capacity) {
        array = (E[]) new Comparable[capacity];
        size = 0;
    }

    /**
     * 构造函数，构造容量为10的最大堆
     */
    public MaxHeap() {
        this(DEFAULT_CAPACITY);
    }

    /**
     * 是否为空
     * @return
     */
    public boolean isEmpty() {
        return size == 0;
    }

    /**
     * 查找最大元素
     *
     * @return 堆顶元素
     */
    public E findMax() {
        if (isEmpty()) {
            throw new IllegalArgumentException("Can not find max because the heap is empty.");
        }

        return array[0];
    }

    private void resize(int newCapacity) {
        E[] newData = (E[]) new Comparable[newCapacity];
        for (int i = 0; i < array.length; i++) {
            newData[i] = array[i];
        }
        array = newData;
    }

    /**
     * 添加元素
     *
     * @param e e
     */
    public void insert(E e) {

        if (array.length - 1 == size) {
            resize(2 * array.length + 1);
        }

        // 如果使用交换的方法，需要三次赋值，一个元素上浮d层，就需要3d次赋值，我们这个方法优化可以只用d+1次赋值
        int hole = ++size;
        for (array[0] = e; e.compareTo(array[hole / 2]) > 0; hole /= 2) {
            array[hole] = array[hole / 2];
        }
        array[hole] = e;
    }

    /**
     * 删除堆顶元素
     *
     * @return 堆顶元素
     */
    public E deleteMax() {
        E maxItem = findMax();
        array[1] = array[size];
        array[size--] = null;
        siftDown(MAX_INDEX);

        // 当元素个数变成总容量的1/4的时候，释放内存到
        if (size == array.length / 4 && array.length / 2 != 0) {
            resize(array.length / 2);
        }
        return maxItem;
    }

    /**
     * 下沉操作
     *
     * @param hole 开始下沉的位置
     */
    private void siftDown(int hole) {
        int child = 0;
        E tmp = array[hole];
        for (; hole * 2 <= size; hole = child) {
            child = hole * 2;
            if (child != size && array[child + 1].compareTo(array[child]) > 0) {
                child++;
            }
            if (array[child].compareTo(tmp) > 0) {
                array[hole] = array[child];
            } else {
                break;
            }
        }
        array[hole] = tmp;
    }

}
```



## 时间复杂度分析

通过二叉树的性质可以知道n个节点的完全二叉树深度为h = log<sub>2</sub>(n+1)，插入和删除堆顶元素的时候应用了堆的的调整算法`siftDown()`和`siftUp()`，循环的次数最多为h - 1, 所以插入和删除的时间复杂度为**O(log<sub>2</sub>n)**。

## leetcode 347题

**题目：**给定一个非空的整数数组，返回其中出现频率前 k高的元素。

```
输入: nums = [1,1,1,2,2,3], k = 2
输出: [1,2]
```

**Java实现：**

```java
package com.leetcode;

import java.util.*;

/**
 * leetcode347 https://leetcode-cn.com/problems/top-k-frequent-elements/
 */
public class Solution347 {

    public static List<Integer> topKFrequent(int[] nums, int k) {
        // 第一步，建立一个key：元素值----value：出现频率的哈希表
        HashMap<Integer, Integer> map = new HashMap<>();
        for (int num : nums) {
            if (map.containsKey(num)) {
                map.put(num, map.get(num) + 1);
            } else {
                map.put(num, 1);
            }
        }

        PriorityQueue<Integer> queue = new PriorityQueue<>(
                (a, b) -> map.get(a) - map.get(b)
        );

        // 维护一个大小为k的最小堆，每次新加元素和堆顶元素进行比较，如果新元素的频率大于堆顶元素，则将堆顶元素
        // 替换为新加入的元素，
        for (int key: map.keySet()) {
            if (queue.size() < k) {
                queue.add(key);
            } else if (map.get(key) > map.get(queue.peek())) {
                queue.remove();
                queue.add(key);
            }
        }

        // 输出堆
        LinkedList<Integer> result = new LinkedList<>();
        while (!queue.isEmpty()) {
            result.add(queue.remove());
        }
        return result;
    }

}

```


---
layout: post
title: '深入剖析 PriorityQueue：原理、使用与实战'
date: 2024-12-21
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: Java



---

> 由一道LeetCode引发的对PriorityQueue学习思考，本文涵盖PriorityQueue概念、原理、特性、源码，详述使用、核心方法、注意事项、场景等。

# 0. 写在前面

最近刷LeetCode时，遇到一个题目：

> [692. 前K个高频单词](https://leetcode.cn/problems/top-k-frequent-words/)
>
> 给定一个单词列表 words 和一个整数 k ，返回前 k 个出现次数最多的单词。
>
> 返回的答案应该按单词出现频率由高到低排序。如果不同的单词有相同出现频率， 按字典顺序 排序。

![]({{ '/assets/img/PriorityQueue/1.png' | prepend: '' }})
![](.\img\PriorityQueue\1.png)

发现使用PriorityQueue有奇效，之前对于这个优先队列用到的比较少，遂决定深入剖析一下。

给出此题目的比较高效的解：

```java
class Solution {
    public List<String> topKFrequent(String[] words, int k) {
        // 步骤 1：使用哈希表统计每个单词的出现频率
        Map<String, Integer> frequencyMap = new HashMap<>();
        for (String word : words) {
            frequencyMap.put(word, frequencyMap.getOrDefault(word, 0) + 1);
        }

        // 步骤 2：使用自定义比较器的小根堆对单词进行排序
        PriorityQueue<String> minHeap = new PriorityQueue<>((a, b) -> {
            int freqA = frequencyMap.get(a);
            int freqB = frequencyMap.get(b);
            if (freqA == freqB) {
                // 当频率相同时，按照字典序逆序排列（因为小根堆）
                return b.compareTo(a);
            } else {
                return freqA - freqB;
            }
        });

        // 步骤 3：遍历哈希表将单词放入堆中
        for (String word : frequencyMap.keySet()) {
            minHeap.offer(word);
            if (minHeap.size() > k) {
                minHeap.poll();
            }
        }

        // 步骤 4：将堆中的元素添加到结果列表中
        List<String> result = new ArrayList<>();
        while (!minHeap.isEmpty()) {
            result.add(0, minHeap.poll());
        }
        return result;
    }
}
```

![]({{ '/assets/img/PriorityQueue/2.png' | prepend: '' }})
![](.\img\PriorityQueue\2.png)

# 1. 概念

PriorityQueue，直译为优先级队列，它本质上是一个基于优先级堆（通常是二叉堆）实现的无界队列。这意味着，当你向其中插入元素时，这些元素会根据预先定义的优先级规则自动排序，每次从队列中取出元素时，总是获取优先级最高的那个元素。

与普通队列遵循先进先出（FIFO）原则不同，PriorityQueue 打破了这个常规，聚焦于元素的优先级属性，确保高优先级元素优先得到处理，就像医院的急诊室，病情危急的患者会优先得到救治，而不管他们的挂号顺序。

# 2. 原理

PriorityQueue 内部主要依赖于二叉堆数据结构来维持元素的优先级顺序。二叉堆是一种特殊的完全二叉树，分为最大堆和最小堆。在 Java 的 PriorityQueue 中，默认使用的是最小堆实现，即堆顶元素始终是堆中最小的元素。

当插入一个新元素时，它会从底部开始，与父节点比较并不断向上 “上浮”，直至找到合适的位置，以满足最小堆的特性，这个过程被称为 “上滤”。而在删除堆顶元素（优先级最高的元素）时，堆的最后一个元素会被移到堆顶，然后它与子节点比较并不断向下 “下沉”，直到整个堆重新满足最小堆的性质，这一过程叫做 “下滤”。通过这两个核心操作，PriorityQueue 能够高效地动态维护元素的优先级顺序。

假设，我们有4个元素：ele1，ele2，ele3，ele4，他们的优先级分别为2,1,4,3，在二叉堆的结构下，将会以下图方式排列：

![]({{ '/assets/img/PriorityQueue/3.png' | prepend: '' }})
![](.\img\PriorityQueue\3.png)

可以看到二叉堆这种数据结构符合以下几种性质:

1. 完全二叉树结构：除了最后一层外，所有其他层都是满的，并且最后一层的所有节点都尽可能靠左排列。如上图最后一层只有一个ele 4，就把它放在左边。
2. 堆性质：小顶堆的情况下，父节点的值永远小于两个子节点，大顶堆反之。
3. 二叉堆的大小关系永远只针对父子孙这样的层级，假设我们用的是小顶堆，这并不能说明第二层节点就一定比第三层节点小，如上图的ele 3 和ele 4。

# 3. 特性

1. 无界性：理论上它可以容纳无限个元素，只要内存允许。但在实际使用中，仍需考虑内存资源的限制，避免内存溢出。
2. 自动排序：插入元素无需手动排序，它会依据优先级规则自动调整元素位置，保证取出元素时的顺序正确性。
3. 非线程安全：多个线程同时操作 PriorityQueue 可能导致数据不一致等并发问题，若在多线程环境下使用，需要额外的同步措施，如使用 Collections.synchronizedQueue 方法进行包装。
4. 不允许null元素：因为无法确定null的优先级。

# 4. 源码分析

查看 java.util.PriorityQueue 的源码，其核心成员变量包含一个用于存储元素的 Object[] queue 数组，记录当前元素个数的 size 变量，以及一个用于比较元素优先级的 Comparator<? super E> 比较器。

构造函数中，可以传入自定义的比较器，如果没有传入，会使用元素的自然顺序（要求元素实现 Comparable 接口）。例如：

```java
public PriorityQueue(int initialCapacity, Comparator<? super E> comparator) {
    // 初始化容量校验等操作
    this.queue = new Object[initialCapacity];
    this.comparator = comparator;
}
```

在插入元素的 offer(E e) 方法中，会先确保队列容量足够，然后执行 “上滤” 操作，将新元素放到合适位置：

```java
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    modCount++;
    int i = size;
    if (i >= queue.length)
        grow(i + 1);
    size = i + 1;
    if (i == 0)
        queue[0] = e;
    else
        siftUp(i, e);
    return true;
}
```

其中 siftUp 方法实现了 “上滤” 逻辑，通过不断与父节点比较并交换，使新元素上浮到正确位置。

删除堆顶元素的 poll() 方法，首先获取堆顶元素，然后将最后一个元素移到堆顶，再执行 “下滤” 操作：

```java
public E poll() {
    if (size == 0)
        return null;
    int s = --size;
    modCount++;
    E result = (E) queue[0];
    E x = (E) queue[s];
    queue[s] = null;
    if (s!= 0)
        siftDown(0, x);
    return result;
}
```

这里的 siftDown 方法完成 “下滤”，让堆顶元素下沉到合适层级，以恢复最小堆特性。

# 5. 核心方法解析

1. offer(E e)：向队列中插入元素 e，成功返回 true，若因容量不足等原因插入失败则返回 false。如前面源码分析所示，它内部处理了扩容和 “上滤” 操作，确保元素正确入队。时间复杂度O(log(n))。
2. poll()：移除并返回队列头部（优先级最高）的元素，如果队列为空则返回 null。它通过 “下滤” 维持堆的性质，高效获取并删除最高优先级元素。时间复杂度O(log(n))。
3. peek()：返回队列头部元素，但不移除它，队列为空时返回 null。可用于查看当前优先级最高的元素，而不影响队列结构。时间复杂度O(1)。

```java
PriorityQueue<String> namesPQ = new PriorityQueue<>();
namesPQ.add("Alice");
namesPQ.add("Bob");
System.out.println(namesPQ.peek()); // 输出 "Alice"，队列元素不变
```

# 6. 如何使用

创建一个PriorityQueue对象非常简单，可以直接使用默认构造函数，也可以指定初始容量或提供一个比较器。

创建一个简单的 PriorityQueue 示例，存储整数并按照从小到大的顺序取出（默认最小堆）：

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();
pq.add(5);
pq.add(3);
pq.add(8);
while (!pq.isEmpty()) {
    System.out.println(pq.poll());
}
```

上述代码会依次输出 3、5、8，展示了按照优先级（数值大小）排序输出的效果。

若要改变优先级规则，比如按照从大到小存储整数，可以自定义比较器：

```java
PriorityQueue<Integer> pqCustom = new PriorityQueue<>(Comparator.reverseOrder());
pqCustom.add(5);
pqCustom.add(3);
pqCustom.add(8);
while (!pqCustom.isEmpty()) {
    System.out.println(pqCustom.poll());
}
```

这次输出将是 8、5、3。

# 7. 注意事项

1. 元素的可比性：要么元素自身实现 Comparable 接口，要么在创建 PriorityQueue 时传入自定义的 Comparator，否则运行时会抛出 ClassCastException。
2. 性能考虑：虽然插入和删除操作的时间复杂度在堆实现下相对较低（平均为 ），但频繁的插入和删除大量元素时，仍要留意性能消耗，尤其是对实时性要求较高的场景。
3. 非线程安全：在多线程并发访问场景，务必进行同步处理，避免数据混乱，不要错误地认为它能自动保证线程安全。
4. PriorityQueue不保证同优先级元素的顺序。
5. 不支持随机访问操作，如get(int index)。

# 8. 使用场景

当需要频繁地获取最小/最大值时，PriorityQueue是一个很好的选择。

1. **任务调度**：例如操作系统中的进程调度，每个进程有不同的优先级，高优先级进程优先获得 CPU 资源。假设有如下简单的任务类：

```java
class Task {
    private String name;
    private int priority;

    public Task(String name, int priority) {
        this.name = name;
        this.priority = priority;
    }

    public int getPriority() {
        return priority;
    }

    @Override
    public String toString() {
        return "Task{" +
                "name='" + name + '\'' +
                ", priority=" + priority +
                '}';
    }
}
```

使用 PriorityQueue 进行任务调度：  

```java
PriorityQueue<Task> taskQueue = new PriorityQueue<>(Comparator.comparingInt(Task::getPriority));
taskQueue.add(new Task("Task1", 3));
taskQueue.add(new Task("Task2", 1));
taskQueue.add(new Task("Task3", 5));

while (!taskQueue.isEmpty()) {
    Task currentTask = taskQueue.poll();
    System.out.println("Executing: " + currentTask);
}

```

输出会按照任务优先级从高到低依次执行任务，即先执行 Task3，再是 Task1，最后 Task2。



2. **数据排序**：当需要对大量数据进行部分排序，比如找出前 k 个最大或最小的元素时，PriorityQueue 能高效解决。例如找出数组中前 3 个最小的数：

```java
int[] nums = {4, 2, 7, 1, 9, 3, 5};
PriorityQueue<Integer> minPQ = new PriorityQueue<>();
for (int num : nums) {
    minPQ.offer(num);
    if (minPQ.size() > 3) {
        minPQ.poll();
    }
}
while (!minPQ.isEmpty()) {
    System.out.println(minPQ.poll());
}

```

这段代码利用 PriorityQueue 维护一个大小为 3 的最小堆，最终输出数组中的前 3 个最小数。

3. **事件驱动系统**

- 在事件处理系统中，不同的事件可能有不同的优先级。例如，在一个游戏中，玩家输入事件（如控制角色移动）可能具有较高优先级，而一些后台更新场景的事件优先级较低。将这些事件放入 PriorityQueue，游戏引擎可以按照优先级来处理事件。

# 9. PriorityQueue常见问题

1. **PriorityQueue 是线程安全的吗？如果不是，如何使其线程安全？**

- PriorityQueue 不是线程安全的。如果多个线程同时访问一个 PriorityQueue，并且其中至少一个线程修改了队列（例如插入或删除元素），可能会导致数据不一致或者其他并发问题。
- 要使其线程安全，可以使用以下两种常见方法：
  - **使用 Collections.synchronizedCollection 方法**

```java
PriorityQueue<Integer> priorityQueue = new PriorityQueue<>();
Queue<Integer> synchronizedQueue = Collections.synchronizedCollection(priorityQueue);

```

```
    * 这样得到的 synchronizedQueue 是一个线程安全的队列，但是在使用迭代器遍历队列时，需要手动进行同步，以避免在迭代过程中其他线程修改队列导致的问题。
- **使用并发包中的 PriorityBlockingQueue**
    * PriorityBlockingQueue 是一个阻塞式的优先级队列，它实现了 BlockingQueue 接口，不仅提供了线程安全的插入和删除操作，还在队列为空或者队列满时提供了阻塞功能。例如，当从一个空的 PriorityBlockingQueue 中获取元素时，线程会阻塞，直到有元素被放入队列。

```

```java
PriorityBlockingQueue<Integer> blockingQueue = new PriorityBlockingQueue<>();

```



2. **为什么 PriorityQueue 底层用数组构成小顶堆而不是使用链表呢？**

先说结论： **使用数组避免了为维护父子及左邻右舍等节点关系的内存空间占用** 。

**为什么还要维护左邻右舍关系呢？** 我们都知道 `PriorityQueue` 支持传入一个集合生成优先队列，假如我们传入的是一个无序的 `List`,那么在数组转二叉堆时需要经过一个 `heapify` 的操作，该操作需要从最后一个非叶子节点开始，直到根节点为止,不断 `siftDown` 维持自己以及子孙节点间的优先级关系。

如果使用链表这些关系的维护就变得繁琐且占用内存空间，使用数组就不一样了，因为地址的连续性和明确性，我们定位邻节点只需按照公式获得最后一个非叶子节点的索引，然后不断减一就能到达邻节点了。

综上所述，使用数组可以以`O(1)` 的时间复杂度定位到最后一个非叶子节点,通过一个减 1 操作即到达下一个非叶子节点，这种轻量级的关系维护是链表所不具备的。







<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>
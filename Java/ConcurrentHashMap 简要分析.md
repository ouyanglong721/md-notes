# ConcurrentHashMap 简要分析

## JDK1.7和1.8，不同之处？

### 1. JDK1.7

  在JDK1.7中ConcurrentHashMap底层结构为Segment数组。

<img src="https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/image-20220303142553710.png" alt="image-20220303142553710" style="zoom:67%;" />

而Segment继承自ReentrantLock，因此Segment是一个可重入的锁，我们对Segment的线程安全操作就是通过Segment（ReentrantLock）的lock方法来保证的，同时我们也就可以凭借Segment数组的大小来确定ConcurrentHashMap的并发能力，数组越大我们冲突的概率就可能越小，默认情况下并发级别为16(`DEFAULT_CONCURRENCY_LEVEL`)，也就是说我们默认情况下**最多**可以支持16个线程对ConcurrentHashMap进行并发操作。

````java
    static final class Segment<K,V> extends ReentrantLock implements Serializable {
        ...
    }
````

Segment内部主要结构为HashEntry数组，HashEntry的结构就是我们熟悉的Key-Value了，同时解决Hash冲突使用的是**链地址法**，因此有next节点

<img src="https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/image-20220303142857244.png" alt="image-20220303142857244" style="zoom:67%;" />

最终可以简要概括为如下图：

<img src="https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/image-20220303144049224.png" alt="image-20220303144049224" style="zoom:67%;" />

### 2.JDK1.8

而在JDK1.8中，发生了很大的变化。

首先是底层结构的改变，1.8中不再使用Segment，而是与1.8中的HashMap相同，采用了Node与TreeNode(红黑树)结构。

````JAVA
 static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
     ...
 }
````

![image-20220303150041195](https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/image-20220303150041195.png)

其实Node节点与1.7中的HashEntry非常相似。Node如何保证操作线程安全呢？与Segment不同，Node没有实现Lock，因此只能借助其他的工具，也就是**CAS+synchronized**。为什么不继续使用Lock呢，主要是因为在JDK1.6之后，**synchronized**有了一定的优化，如锁升级，锁粗化等等，因此使用synchronized性能不一定比Lock差。同时，我们使用Node数组+(CAS+synchronized)的方式，也能够保证最大的并发度，能保证table数组多大就能有多大的并发度，与1.7的限制并发级别的方法相比并发能力强了不少。

同时1.8中还使用了红黑树结构，在**Node链表**的长度大于8并且table的长度大于64时，还会将链表进化成红黑树，以提高查找效率。

## JDK1.8中如何通过synchronized和cas保证线程安全？

在1.8中，ConcurrentHashMap主要通过synchronized关键字+CAS来保证操作的线程安全。

通过源码可知，在putVal方法中，我们看到了CAS

<img src="https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/image-20220303151317310.png" alt="image-20220303151317310" style="zoom:67%;" />

很明显可以看出，在我们put值进去时，如果这个对象的hash值对应在**table数组中的对应位置为null**，也就是table数组的某个值为null，那么我们会通过CAS来给table数组赋值。

同时，也可以看到在对sizeCtl变量进行操作时，也大量的使用到了CAS操作。

![](https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/image-20220303151940895.png)

![image-20220303151958155](https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/image-20220303151958155.png)

> Table initialization and resizing control. When negative, the table is being initialized or resized: -1 for initialization, else -(1 + the number of active resizing threads). Otherwise, when table is null, holds the initial table size to use upon creation, or 0 for default. After initialization, holds the next element count value upon which to resize the table.
>
> sizeCtl变量主要用于table的初始化以及扩容时的控制。

因此可以总结，在对某些定义好的变量进行修改时，会使用CAS。

而synchronized主要是在Node节点或TreeNode节点的操作上，比如要新增一个Node，将其加在某个Node后面，此时我们会使用synchronized锁住table上对应的Node节点。

<img src="https://oylong-blog-pic.oss-cn-shenzhen.aliyuncs.com/blog/img/image-20220303152903962.png" alt="image-20220303152903962" style="zoom:67%;" />
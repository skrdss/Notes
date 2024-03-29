## java对象布局

java对象在堆上，大小不固定

在堆上为java对象开辟多大的内存空间？

取决于：

1.java对象的实例变量）--->大小不固定

2.对象头--->大小固定

3.数据对齐

​	64位的JVM (64bit 的java虚拟机) 分配内存时必须是	8的整数倍字节（byte）



**synchronized上锁就是改变对象的对象头**

`ClassLayout.parseInstance(实例对象).toPrintable()`可以查看该实例对象的对象布局

![image-20211229213710626](synchronized关键字底层.assets/image-20211229213710626.png)

## java对象头由什么组成

JVM	:	规范/标准，其中规定了对象头由什么组成

Hotpot	:	产品/实现，大部分代码使用的都是openjdk，部分代码商用不开源

openjdk	:	代码（C++/开源）

对象头大小：96 bit / 12 byte

1. Mark Word (64 bit/ 8 byte)

   根据对象的状态（下部分）不同，内容不同

2. klass pointer/Class Metadata Address (32 bit/ 4 byte)

   使用的java虚拟机如果开启**指针压缩**，那么klass pointer就是**32bit**,如果没有，则klass pointer为**64bit**

## 乐观锁与悲观锁

乐观锁总是假设资源没有竞争。典型的乐观锁为CAS操作，直接尝试进行操作，如果操作成功则对资源进行修改，如果操作失败则证明有锁。

悲观锁总是假设资源有竞争。典型的悲观锁为synchronized锁，在访问资源时直接加上锁。

## CAS操作

内存地址V，旧变量值A，新变量值B。先获取地址V储存的值A，然后对A进行操作，得到新值B，此时再次获取地址V储存的值，如果与A相等，则证明该值没有被修改过，就将B值放入地址V中。但是这样会导致**ABA问题**。

## ABA问题

即线程1读取内存变量为A,线程2修改内存变量为B,然后线程2又修改内存变量为A,这时T1再CAS操作A时是可行的。但实际上在T1第二次操作A时，已经被其他线程修改过了。这就是ABA问题。



## 使用synchronized关键字时锁对象有哪些状态？

`synchronized(l){....}`

对象`l`的状态有5种：

1. 无状态

   刚 new 出来的对象

2. 偏向锁

3. 轻量锁

4. 重量锁（synchronized加的就是重量锁）

5. gc标记

   例子：

   ```java
   public void function_1(){
   
   ​	A a = new A();
   
   .......
   
   }
   ```

   a为堆中的对象的引用

   当方法运行完后，对象的引用结束，方法的实例对象会被gc标记，证明该对象将要被回收

   

   **对象头：**

   128bit 或 96bit ，取决于虚拟机是否指针压缩

![image-20211230210244895](synchronized关键字底层.assets/image-20211230210244895.png)

lock 为 2 位，怎么能表示上述5种状态？

加了 1 位 biased_lock 位，偏向锁状态和无锁状态的 lock 2位表示是一样的，通过 biased_lock 位区分，后面三种状态无需biased_lock 位

Intel架构采用小端存储，低地址存在高位，高地址存在低位，因此对应查看输出的对象布局时，应该倒着对应。hashcode本身不存在，因此该31位都是0，如果调用`l.hashcode()`方法计算过hashcode后，该对应位就会有值

## JVM一般是这样使用锁和Mark Word的：

1. 当没有被当成锁时，这就是一个普通的对象，Mark Word记录对象的HashCode，锁标志位是01，是否偏向锁那一位是0。
2. 当对象被当做同步锁并有一个线程A抢到了锁时，锁标志位还是01，但是否偏向锁那一位改成1，前23bit记录抢到锁的线程id，表示进入偏向锁状态。
3. 当线程A再次试图来获得锁时，JVM发现同步锁对象的标志位是01，是否偏向锁是1，也就是偏向状态，Mark Word中记录的线程id就是线程A自己的id，表示线程A已经获得了这个偏向锁，可以执行同步锁的代码。
4. 当线程B试图获得这个锁时，JVM发现同步锁处于偏向状态，但是Mark Word中的线程id记录的不是B，那么线程B会先用CAS操作试图获得锁，这里的获得锁操作是有可能成功的，因为线程A一般不会自动释放偏向锁。如果抢锁成功，就把Mark Word里的线程id改为线程B的id，代表线程B获得了这个偏向锁，可以执行同步锁代码。如果抢锁失败，则继续执行步骤5。
5. 偏向锁状态抢锁失败，代表当前锁有一定的竞争，偏向锁将升级为轻量级锁。JVM会在当前线程的线程栈中开辟一块单独的空间，里面保存指向对象锁Mark Word的指针，同时在对象锁Mark Word中保存指向这片空间的指针。上述两个保存操作都是CAS操作，如果保存成功，代表线程抢到了同步锁，就把Mark Word中的锁标志位改成00，可以执行同步锁代码。如果保存失败，表示抢锁失败，竞争太激烈，继续执行步骤6。
6. 轻量级锁抢锁失败，JVM会使用自旋锁，自旋锁不是一个锁状态，只是代表不断的重试，尝试抢锁。从JDK1.7开始，自旋锁默认启用，自旋次数由JVM决定。如果抢锁成功则执行同步锁代码，如果失败则继续执行步骤7。
7. 自旋锁重试之后如果抢锁依然失败，同步锁会升级至重量级锁，锁标志位改为10。在这个状态下，未抢到锁的线程都会被阻塞。

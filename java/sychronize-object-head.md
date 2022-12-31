# Sychronize-Object-Head

sychronize是java中开启锁同步的的关键字,用于修饰在对象、静态和非静态称之为'同步锁',以达到在程序操作临界资源不受其他线程影响的目的。在代码中sychronize 和 cas中lock 及 unlock  相似:

````java
sychroinzed(对象){
   //临界区操作
}
````

或

```` java
public void sychronized sync(){
   //临界区操作
}
````

类似于

````c
lock();
//临界区操作
unlock();
````



### sychronized 锁的是对象还是代码块？

#### 对象布局

####  mark word(first word of every object header)

**原文:**

> The first word of every object header. Usually a set of bitfields including synchronization state and identity hash code. May also be a pointer (with characteristic low bit encoding) to synchronization related information. During GC, may contain GC state bits.

对象头的第一部分,随着情况不同包含了与以下四个信息: 

1.synchronization状态(synchronization state)

2.对象的hashcode(identity hash code)

3.与synchronization相关的线程id(pointer synchronization related information)

4.gc期间的分代年龄(GC state bit)





#### klass pointer(second word of every object header)

**原文:**

> The second word of every object header. Points to another object (a metaobject) which describes the layout and behavior of the original object. For Java objects, the "klass" contains a C++ style "vtable".

对象头的第二部分, 指向另一个对象metaobject 元对象

metaobject 元对象:描述原来对象的布局和行为,也可以理解为描述java对象的C++对象



#### object header

**原文:**

> Common structure at the beginning of every GC-managed heap object. (Every oop points to an object header.) Includes fundamental information about the heap object's layout, type, GC state, synchronization state, and identity hash code. Consists of two words. In arrays it is immediately followed by a length field. Note that both Java objects and VM-internal objects have a common object header format.

交由gc 管理的对象开头的公共结构部分,包括:

1.关于堆对象布局的基本信息(fundamental information about the heap object's layout)

2.类型(type)

3.gc 状态/分代年龄(gc state)

4.sychronized 状态(synchronization state)

5.identity hash code

> object header = (mark word + klass pointer)



#### biased locking(偏向锁)

> An optimization in the VM that leaves an object as logically locked by a given thread even after the thread has released the lock. The premise is that if the thread subsequently reacquires the lock (as often happens), then reacquisition can be achieved at very low cost. If a different thread tries to acquire a biased lock then the bias must be revoked from the current bias owner.

jvm对锁的优化,线程释放锁后保留逻辑上的锁定

如果线程释放锁后重新获取锁(经常发生),可以以非常低的成本重新获取。只有一个线程能持有biased locking。



#### Lightweight locking(轻量锁)

轻量锁也是用来提升程序的性能的。对于交替执行的程序，轻量锁通过JVM平台的CAS操作来规避频繁触发MUTEX的情况。轻量锁的锁标识锁00 

#### heavyweight lock(重量锁)

触发系统级调用的锁(MUTEX)



##### JOL-CORE

用openjdk提供的开源工具 org.openjdk.jol:jol-core(github上有使用教程)工具查看实际的对象头布局

设置jvm参数对指针压缩进行关闭，以及开启偏向锁，注意jol工具包的版本，这里用的是0.9

````java
-XX:-UseCompressedOops // 关闭指针压缩,把减号改成加号为开启
-XX:-BiasedLockingStartupDelay // 关闭偏向延迟
````

```java
@Test
public void testObjectHeader() throws IOException {
  log.info("hashcode:"+ Integer.toHexString(j.hashCode()));
  log.info(ClassLayout.parseInstance(j).toPrintable());
}
```



通过搜索 object header 在MonitorSnippets C++类中对状态进行了这样的描述

>    Note also that the biased state contains the age bits normally
>    contained in the object header. Large increases in scavenge
>    times were seen when these bits were absent and an arbitrary age
>    assigned to all biased objects, because they tended to consume a
>    significant fraction of the eden semispaces and were not
>    promoted promptly, causing an increase in the amount of copying
>    performed. The runtime system aligns all JavaThread* pointers to
>    a very large value (currently 128 bytes (32bVM) or 256 bytes (64bVM))
>    to make room for the age bits & the epoch bits (used in support of
>    biased locking), and for the CMS "freeness" bit in the 64bVM (+COOPs).

1.normal object

````java
未使用     hashcode  （56 bit）     分代年龄   偏向锁标识       锁  (8 bit) 共 64
unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
````

普通对象表示无锁状态的对象布局，因为没有计算hashcode，所以

2.baised object

````java
线程id                     (56 bit)
JavaThread*:54 epoch:2  -->|  unused:1   age:4    biased_lock:1 lock:2 (biased object)
JavaThread*:54 epoch:2  -->|  cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
````



被synchronize 关键字修饰后的锁状态有以下几种状态

1.无锁状态:计算了hashcode ,不满足偏向条件

````c++
hash code 2cccf134 
OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           19 34 f1 cc 
                                                     (00011'001' 00110100 11110001 11001100)
      4     4        (object header)                           2c 00 00 00
                                                       (00101100 00000000 00000000 00000000)
      8     4        (object header)                           78 5a 14 00 
                                                       (01111000 01011010 00010100 00000000)
     12     4        (loss due to the next object alignment)
````

001是无锁的状态，当计算了hashcode之后,后面后31位填充的是hashcode ，因此对象头没有办法装下54位的线程id。所以不满足偏向锁的上锁条件，无法膨胀成偏向锁，一旦发生了资源交替执行的状态,就会由无锁边成轻量锁。

这里值得注意的是此处的， hashcode 是由右往左从下往上输出,这是因为机器使用了小端存储的机制在小端存储模式的情况下锁体现的顺序的如下： 

​            unsed  age biased_lock    lock        hashcode     unsed     

 bit:        1         4        1                     2               31                    25

​           unsed age biased_lock    lock          ThreadId

 bit:       1         4          1                    2                  54



2.无锁状态,  没有计算hashcode  满足偏向条件

````java
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           25 00 00 00 
                                                     (00100'101' 00000000 00000000 00000000)
      4     4        (object header)                           00 00 00 00
                                                      (00000000 00000000 00000000 00000000)
      8     4        (object header)                           78 5a 14 00 
                                                      (01111000 01011010 00010100 00000000) 
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
````

可以偏向，当发生资源资源交替执行时，就会变成偏向锁状态。

````java
2022-12-27 01:07:35.518  INFO 11852 --- [           main] com.example.demo.DemoApplicationTests    : com.example.demo.threadlearn.mylock.J object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           15 30 a1 28 
                                                      (00010'101' 00110000 10100001 00101000)
      4     4        (object header)                           01 00 00 00 
                                                      (00000001 00000000 00000000 00000000)
      8     4        (object header)                           80 62 bd 26 
                                                      (10000000 01100010 10111101 00100110)
     12     4        (object header)                           01 00 00 00 
                                                      (00000001 00000000 00000000 00000000) 
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
````

101 为可偏向,后面提示的是线程id，这时候该对象已经被上锁，当线程结束后，再次进入线程加锁，系统缓存了线程id不在需要创建线程，只需要沿用原来的线程id 即可。

3.膨胀为 非偏向 轻量锁 :000 第一位是偏向为，后面两位是锁的状态，这是一把轻量锁

````java
java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           00 be 3e 6f 
                                                     (00000'000' 10111110 00111110 01101111) 
      4     4        (object header)                           01 00 00 00 
                                                     (00000001 00000000 00000000 00000000)
      8     4        (object header)                           00 10 00 00 
                                                    (00000000 00010000 00000000 00000000)  
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
````

当发生资源交替执行时锁会自动膨胀为轻量锁

4.膨胀为 非偏向重量锁 :0 10

````java
2022-12-30 22:25:24.771  INFO 12905 --- [      Thread-55] com.example.demo.DemoApplicationTests    : com.example.demo.threadlearn.mylock.J object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           02 49 c2 39
                                                    (00000'010' 01001001 11000010 00111001) 
      4     4        (object header)                           01 00 00 00 
                                                    (00000001 00000000 00000000 00000000)  
      8     4        (object header)                           d0 6f 13 00 
                                                    (11010000 01101111 00010011 00000000) 
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

````

当发生资源竞争时锁会自动膨胀为重量锁

以上是当sychronize 修饰方法时，会给当前方法所在的对象头markwork上的锁状态进行修改，按照以上的结果来看 synchronize 因为改的是object header 里面的状态，所以认为是一把对象锁。根据object header 里面的状态对代码进行临界区操作.





#### synchronized的字节码

synchronized 和native方法一样是系统的关键字，都知道java是通过javac将.java文件翻译成.class文件交由jvm对文件进行执行。

```java
synchronized (mySync){
        synchronized (mySync){
             synchronized (mySync){
                    //to do
                   
             }
        }
 }
```

当尝试使用javac -p 命令反编译上述代码，有一段字节码整理后

可以确定 monitorenter `与 `monitorexit为synchronize 的字节码



总结：

sychronized 进行修饰时，会对当前修饰的对象头里面进行相应的操作，当发生对象被反复加锁的时候。会判断对象有没有计算hashcode ，在没有计算hashcode 的情况下，偏向锁的标记位变成1，无锁标记为为01不进行改变，所以标记码是101，果线程释放锁后重新获取锁(经常发生),可以以非常低的成本重新获取。只有一个线程能持有biased locking。

如果计算了hashcode 因为64位的对象头已经没有办法装下线程id，所以不在进行逻辑锁定，直接膨胀为轻量锁,标记位是000，还有一种情况是在偏向锁的倾向下。程序发生了交替执行轻量锁通过JVM平台的CAS操作来规避频繁触发MUTEX的情况。

如果发生了资源竞争，多个资源抢占式执行。会由轻量锁膨胀为重量锁。膨胀为重量锁后会发生系统级调用，程序会进入内核台。在平台切换为内核态时会产生开销。



To be continue...

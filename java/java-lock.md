# Java 锁

作者：[AzathoX](https://github.com/AzathoX)

日期：Nov 16 2022

曾经发生过这么一件事：我去参加了一次大型的羽毛球比赛，虽然不是队中的主力，但为了不去辜负和浪费对手的时间，每场比赛都有在认真的打。当时是正在打的是团体赛的四进二，我清楚的记得我的队伍打的最后一分，也就是赛点打的 20 比 17。可是打着打着却发现了不对劲，比分变成了 3 比 21。在比赛结束的时我们都满头问号，要搞清楚这外挂似的 17 分，之后才知道又个锁的这个概念。

## 临界资源和临界区

在现代的常用计算机中，为了提高工作效率，能同时更好利用现代计算机的多核资源采用了多个进程和多个线程，多程序的运行。并发进程/线程有助于提高性能，就在多个进程/线程同时操作同一个资源的时候，需要保证这个被同时访问的共享资源不会受到的不正确状态和不一致状态的影响，避免多道程序环境下扰乱共享资源的正常数据。这些共享资源称为临界资源，而对这些共享资源进行操作的代码称为临界代码或者临界区。



### 临界资源(critical resource)

临界资源也被叫做 **共享资源**，不是所有的共享资源都是临界资源，临界资源每次只被一个进程使用。

举个例子：交叉铁道口就是一个典型的临界资源，每个道口只允许一个方向一台列车通过。

### 临界区(critical section)

临界区即是对临界资源的操作代码片段。临界区交错运行时就会出现两个进程同时争夺一个资源称为：**资源竞争**。

---

在临界区中需要通过什么样的手段保证资源的正确状态与一致性？

在临界资源被占用时通过某种方式对其他想要访问的资源进行阻挡，等临界资源释放后在给下一个进程占用。这个概念叫 **锁**。

## 锁

#### 在 Linux 中锁的机制方式有三种：



### 自旋锁(spin lock)

**发生资源竞争的情况下拿不到锁，空转**。自旋锁是相对低级的锁机制，适用于现代多核 CPU。他的实现机制是其他线程在进行操作临界资源之前尝试拿锁，如果拿不到锁使循环条件就让永远成立进行死循环反复的尝试拿锁的过程(空转)。这种机制因为循环条件永远成立而导致在操作临界资源的过程中因空转(摸鱼)而长期占用了 CPU 造成在操作过程中的资源浪费。当上一个资源操作完成后死循环条件不在成立遍拿到了锁可以进行临界资源的操作。很明显自选锁的优点是能够时刻保证锁状态的一致性，而缺点是因为条件成立的死循环而占据 CPU 导致资源浪费。

`pthread_spin_*`

```c
/* 自旋锁的初始化 */
int  pthread_spin_init(pthread_spinlock_t *lock, int pshared);

/* 锁 */
ret = pthread_spin_lock(&lock);
// 或
ret = pthread_spin_trylock(&lock);

/* 解锁 */
ret = pthread_spin_unlock(&lock);
```

### 互斥锁(Mutual Exclusion Lock)

**发生资源竞争的情况下拿不到锁，睡眠**。个人的观点是，互斥锁是自旋锁的改进版，因为自旋锁上锁的时候进入了死循环占据了 CPU，用睡眠将 CPU 的控制权还给了内核，从而 CPU 不在处理当前拿不到锁的这个进程。确保了同一时间只有一个进程操作临界区。也确保了但线程的操作代码。因为睡眠的机制的缺点主要有：进程处理睡眠时需要切换到内核态，让出 CPU 的控制权，在切换状态的时候需要做大量的存储工作，比如：保存当前运行的代码行数，当前的运行状态等。醒来查看是否拿到锁拿不到继续睡眠，时间设置不当会造成性能问题。因为拿不到锁就睡眠，即时上一个资源完成操作，释放了锁，睡眠时间没有到也需要等到睡眠时间到了之后才能继续进行拿锁操作。而优点是能够让出 CPU 的控制权，更有效率的执行任务。

```c
/* 初始化 mutex 锁 */
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

/* 锁 */
ret = pthread_mutex_lock(&mutex);
// 或
ret = pthread_mutex_trylock(&mutex);

/* 解锁 */
ret = pthread_mutex_unlock(&mutex);
```

### 信号量(semaphore)

**对变量进行自增或自减操作如果该变量小于 0，阻塞**。信号量是一种编程模式，他的原型正是上文所提到的铁路运营模式。当该段铁路正在进行通车时，信号灯转成不允许通车的状态。从而阻止其他类车进入该铁道。然而当列车经过铁道以后，信号灯转成正在通车的状态，此时可以允许其他列车进入。在操作系统中信号量以简单的整数变量形式体现，当线程要使用某个资源时。信号量的整数就会减 1 用 P 表示，当信号量为负数时，代表目前所有资源正在使用，因此信号量在递减之前必须是整数。从而进程会等待。当使用完成后操作后给信号量进行加 1 操作用 V 表示。也可以称为 PV 操作。适合用于多个同类型的资源的情况下。信号量功能相当强大,但很容易使用在不确定的非结构体当中。

```c
/* 信号量初始化 */
sem_t *sem*;
int count = 4; // 信号总量
*ret* = sem_init(&*sem*, 0, count);

/* 减 1 操作 */
ret = sem_post(&*sem*);

/* 加 1 操作 */
ret = sem_trywait(&sem);
```

### CAS 锁

#### 条件变量(Condition Variables)

**对锁状态的标识**。使用条件变量当条件成立时就代表获取到锁。条件变量是上锁状态的一个判定。条件变量始终与互斥锁一起使用。但需要注意的是条件变量一定是 CPU 别的原语满足不可拆分的条件。

#### 对比和交换(Compare and Swap)

按照字面上的意思，是先对比后交换，基本的操作如下:

1. 首先定义一个条件变量；
2. 给条件变量赋一个初始值为 0；
3. 判断条件变量是否是上锁状态，如果上锁状态就让死循环条件成立，反之；
4. 解锁让条件变量初始为 0。

#### 实现一把 CAS 锁

Java 要想操作真内存是需要 `Unsafe` 这个类。

1. 先定义变量：

```java
private volatile int status = 0;
private static final Unsafe unsafe = getUnsafe();
private static long valueOffset = 0; // CPU 级别原子性保证条件变量
```

2. 利用反射获取 `Unsafe` 的 `theStatus` 属性并初始化 `Unsafe`：

```java
public static Unsafe getUnsafe() {
    try {
        Field field = Unsafe.class.getDeclaredField("theUnsafe");
        field.setAccessible(true);
        return (Unsafe) field.get(null);
    } catch (NoSuchFieldException e) {
        e.printStackTrace();
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    }
    return null;
}
```

3. 获取 `status` 的偏移量：

```java
valueOffset = unsafe.objectFieldOffset(MySync.class.getDeclaredField("status"));
```

4. 判断是否修改成功：

```java
public boolean compareAndSet(int except,int newValue) {
    // 原子操作
    // 操作，修改status 成功则返回true
    return unsafe.compareAndSwapInt(this, valueOffset, except, newValue);
}
```

5. 尝试拿锁：

```java
public void lock() {
    // 判断如果是0 改为1 返回true 如果不为0 改不成功返回false
    while (!compareAndSet(0,1)) {
        // Thread.yield();
        // Thread.sleep();
    }
    return;
}
```

6. 解锁：

```java
public void unlock() {
    status = 0;
}
```

还有另一种方式通过 `AtomicInteger` 这个类作为条件变量来进行实现：

1. 先将条件变量设为 0：

```java
private static AtomicInteger atomicInteger = new AtomicInteger(0);
```

2. 进行尝试拿锁进行循环：

```java
while (!atomicInteger.compareAndSet(0,1)) {
    Thread.sleep();
}
return;
```

3. 解锁：

```java
atomicInteger.set(0);
```

## 保护环(Protect Rings)

### 用户态(User Mode)

用户只能在操作系统给予的有限权限内进行资源的访问，用户状态下 CPU 没有办法被单独的程序占据，这是一种用来在发生故障时保护数据和功能，提升容错度，避免用户的恶意操作。

### 内核态

处于内核态的 CPU 可以访问任意的数据。处于内核态的 CPU 可以从一个程序切换到另外一个程序，并且占用 CPU 不会发生抢占情况，一般处于特权级 Rings 0 的状态我们称之为内核态。

### 保护环升级

保护环升级，一旦 MMU 内存搜索找到是内核空间的相关内容时 CPU 会升级成为 Ring 0 内核态，保存当前运行的所有程序运行的信息，包括当前内存位置，当前执行编码行数灯所有信息，然后升级，进入内核态。在进行内核态对应的操作。当执行完成后还原回用户态时，将所有存储的信息还原并继续运行。这个过程是比较损耗新能的。

## 总结

总的来说，在多道程序中，锁是为了防止在程序运行过程中防止临界资源的混乱。这些锁都是一个变量，当这个变量设置为某个值时代表锁的状态。比如可以设置状态是 1 代表有锁，那就证明当前临界资源存在锁。在操作系统中有三种锁，**自选锁**，先尝试拿锁，拿不到锁就进入循环，这种自旋锁优点是可以及时知道锁什么时候解锁，切点也很明显长期占用 CPU。**互斥锁**，同样是先尝试锁，拿不到锁就睡觉等待，然后每次醒来查看一次锁的状态，优点是在睡眠的时候可以释放 CPU 的资源给其他程序使用，提高了其他 CPU 的使用率。缺点是睡眠时稍稍有等待。**信号量**，这是一种非常强大的多线程编程模式。

如果不足或者错误欢迎更正和指点。

To be continued...

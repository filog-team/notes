# Java 线程模型

作者：[AzathoX](https://github.com/AzathoX)

日期：Nov 7 2022

## Java 线程映射至操作系统线程

### 1. 进程和线程

#### 进程(Process):

进程是程序的动态执行过程，是由 CPU 分配的运行单位。

*进程 = 进程控制块 + 获得时间片的程序 + 数据*

进程赋予代码在系统中工作的资格。进程作为操作系统实现程序的基本工作单元（实体），操作系统收到申请后会在内存中给程序划分一块带有结构的专属内存空间，当程序获得 CPU 时间片后便赋予了生命，程序也就变成进程开始执行。

1. 程序被编译为二进制机器码后加载到 *文本段(Text region)*；
2. 当获取到 CPU 分配的时间片后，编译器倒叙地将执行指令输送到栈中，然后 CPU 开始由倒叙的顺序执行指令并将机器码的执行通过计数器来记录行数；
3. 当 CPU 从栈获取到指令后，根据指令从 *数据段(Data region)* 中获取对应地址并拿到实际的数据来进行运算；
4. 在内核中有一个记录所有进程的执行状态的数据表 `Process Control Block`。

#### 线程(Thread):

线程是进程的逻辑体现，进程执行过程中的每一个任务都是线程。

*线程 = 线程控制快 + 进程每个任务 + 进程数据*

起初线程的概念是由 Windows 实现并提出，称为轻量级进程。线程作为进程内部的工作单元，每个进程可以有一个或多个线程，每个线程可以共享同任务中的进程数据。因为线程的创建在进程内部完成而无需向系统申请资源，所以开销项比子进程小很多。因为线程能按照相关的任务逻辑来使用进程的资源结构和进程基本一致。在cpu指令的是多个线程交替指令。



## 线程对应的模型

### 1. 一对一模型

一个用户线程对应一个内核线程，内核负责每个线程的调度。

- **优点**：几乎把所有线程操作都交给了操作系统，实现线程模型容器相对简单；
- **缺点**：引起用户态和内核态频繁切换内核为每个线程都对应了相关的调度实体。

如果大量使用线程造成两态互相切换，会对系统造成影响。在 Java 中使用线程还是需要控制。

### 2. 多对一模型

用户内部实现了线程的相关模型，只有需要用到内核调度，才使用状态切换，此时内核只有一条模型。

内部实现线程，只有需要才调度内核，内核只有一个线程。

- **优点**：用户态和用户态切换较少；
- **缺点**：一个线程异常导致内核线程停掉，会影响其他线程，无法很好的兼容现代多核 CPU。

### 3. 多对多模型

用多路复用的手段，多对一模型和一对一模型的特点，将多个用户线程对应到不止一个内核线程上。

- **优点**：不限制应用的线程数，多个线程可以并发，当一个线程被阻塞时，内核可以调用另一个线程来执行；
- **缺点**：复杂度高。

## 

### Java 线程(Java thread)和操作系统线程(OS thread)

> java中hotspot 规范是典型的一对一线程模型，jvm平台上只做少量的管理

#### 1.POXIS 规范

向外界应用提供源码级别接口调用的标准，规定了调用内核的接口规范。特别强调各种商业应用程序所需的功能和设施，目前市面上大部分操作系统都遵循这套规范。

#### 2.OS: `pthread_create`

在操作系统中申请线程资源创建线程的命令：

`pthread_create(pthread_t *thread, const pthread_attr_t *attr,void *(*start_routine)(void *), void *arg);`

- `pthread_t *thread` 线程指针；
- `const pthread_attr_t *attr` 线程的属性，如果传入 `NULL` 值代表使用内核默认属性；
- `void *(*start_routine)(void *)` 线程的工作任务；
- The `pthread_create()` function conforms to ISO/IEC 9945-1:1996 ("POSIX.1").

#### 3.Java: `new Thread().start()`

```java
private static native void registerNatives();

static {
    registerNatives();
}

private native void start0();
```

- Java 的线程最终会调用一个叫 `start0` 方法；
- `start0` 方法是由 `native` 进行修饰的方法；
- `native` 方法是由 JVM 进行处理的方法。

似乎 Java 不太想公开线程的秘密，根据 `Thread` 类的注释，可以看到这是调用 Java 虚拟机内部的线程方法，所以整个线程似乎并不想交由 Java 来处理。

### 映射

JAVA的 `start0()` 方法是通过JVM 调用操作系统的线程 `pthread_create` ,最后是使用了操作系统内核的线程。

尝试在 JDK 搜索 `start0` 相关字样，在JDK 源码中 Thread.c 文件中有这么一段代码：

1. 定义数组

```c
static JNINativeMethod methods[] = {
    {"start0",           "()V",        (void *)&JVM_StartThread},
    // ...
};

JNIEXPORT void JNICALL
Java_java_lang_Thread_registerNatives(JNIEnv *env, jclass cls)
{
    (*env)->RegisterNatives(env, cls, methods, ARRAY_LENGTH(methods));
}
```

在jvm.cpp 这个文件中使用了这个结构体 `JVM_StartThread` 。

2. jvm.cpp 开启线程的逻辑

```c++
JVM_ENTRY(void, JVM_StartThread(JNIEnv* env, jobject jthread)){
JavaThread *native_thread = NULL;
native_thread = new JavaThread(&thread_entry, sz);
Thread::start(native_thread)
}
```

```c++
JavaThread::JavaThread(ThreadFunction entry_point, size_t stack_sz) :
                       Thread() {
    os::ThreadType thr_type = os::java_thread;
    os::create_thread(this, thr_type, stack_sz);
}
```

3. 对应到 os_linux.cpp

这里调用了上述说的 Linux 内核创建线程的函数。

```c++
static void *thread_native_entry(Thread *thread) {
    thread->call_run();
}
bool os::create_thread(Thread* thread, ThreadType thr_type,
                       size_t req_stack_size) {
   int ret = pthread_create(&tid, &attr, (void* (*)(void*)) thread_native_entry, thread);
}
```

所以 ，JVM 偷懒将所有实现的线程实现细节交给了内核进行实现，这样一来达到了 Java 线程与内核线程一一对应的关系。当使用 `start()` 方法后 JVM 会调用 `pthread_create` 方法创建线程。Java 虚拟机维护了 Java 线程和 OS 线程的一对一关系。

### 4. JNI(Java native interface) 对 Java 线程的模拟

#### 模仿线程写普通的java 类

```java
private native void start1();
```

根据 System 这个类的提示补加载 ddl 或者 so 文件的语句，直接加载生成的文件。

```java
static {
    System.load(".so文件的绝对路径");
}
```

但是在下面有一个 `loadLibrary` 方法根据注释是指定文件夹。但目前本人还没有尝试出来，一直报的没找到文件库，乐意接受指点。

然后进行调用：

```java
MyThread myThread = new MyThread();
myThread.start1();
```

最后实现 `Runnable` 重写 `run` 方法。

#### Java 生成头文件

```shell
javac -h . xxx.java
```

编译 Java 文件为 class 并生成头文件。

```c
JNIEXPORT void JNICALL Java_MyThread_start1
    (JNIEnv *, jobject);
```

头文件中有一段非常眼熟的代码，其命名方式就是 `JAVA\_包名\_native\_方法名(JNIEnv *env, jclass cls)`。

```c
JNIEXPORT void JNICALL
Java_java_lang_Thread_registerNatives(JNIEnv *env, jclass cls)
```

然后按照这个头文件编写一个实现的方法里面使用 `pthread` 启动线程，而线程的任务就是调用 `run` 的方法体。

#### 通过 JNI 调用 Java 的 `run` 方法

```c
//获取类
jclass jcls = (*env)->GetObjectClass(env,jobj);
//得到方法
jmethodID jmethodId = (*env)->GetMethodID(env,jcls, "run","()V");
//实例化
jobject obj = (*env)->AllocObject(env,jcls);
(*env)->CallVoidMethod(env,obj,jmethodId,0);
```

#### 将函数封装后丢给 `pthread_create` 线程执行

```c
pthread_create(&pid,NULL,thread_entity,NULL);
```

#### 生成 so 文件

`gcc -shared -fPIC -o libnative.so -I ${JAVA_HOME}/include -I ${JAVA_HOME}/include/{操作系统目录/linux是linux mac是darwin} [文件].c`

所以，Java 线程是通过 Native JNI 方式和操作系统意义一一对应，这也解释了 Java 本身对线程有一定的局限性的根本原因。

## 线程间的上下文切换

1. 进程内部发生了系统调用，保存当前执行到的信息（内核态和用户态的切换）；
2. 进程间的上下文切换，保存进程和软件的信息（代价大,保存进程运行信息）；
3. 线程间的切换，进程内部线程间切换；
4. 线程间的切换，外部线程的线程（跨进程）。

To be continued...

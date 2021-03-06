# 线程基础知识



## 1、sleep() 和 wait() 的区别

sleep() 是 Thread 的 static 方法，随时可以 进行调用，`Thread.sleep()`，它跟线程是否持有锁的状态无关，即就算持有锁也不会释放锁，只是 park 挂起后不会跟其他的线程争夺 CPU



wait() 是 Object 的方法，底层调用的是 另一个 本地方法 wait(0)，它必须在获取 sync 锁的情况下使用，会释放锁，同时会进入加锁对象的 ObjectMonitor 中的 WaitSet 队列中

```java
public final void wait() throws InterruptedException {
    wait(0);
}
public final native void wait(long timeout) throws InterruptedException;
```



它们都需要 捕获 中断异常

（wait()、sleep()、join()、tryLock(long)、lockInterruptibly() 都是可以被中断的，而 sync 锁阻塞、lock() 阻塞都是不能被中断的）



> ### wait(1000)与sleep(1000)的区别

sleep(1000) 表示将线程挂起，在未来的 1000ms 内不参与竞争 CPU，在 1000ms 后，会取消挂起状态，参与竞争 CPU ，这时不一定 CPU 就立马调度它，因此它**等待的时间的 >= 1000ms**



wait(1000) 跟 wait() 的差别在于 **wait(1000) 不需要 notify() 来唤醒**，它等待 1000ms 后，如果此时没有线程占有锁，那么它会自动唤醒尝试获取锁





## 2、线程等待的四种方式（了解即可）

- 使用 CountDownLatch
- 使用 CyclicBarrier
- 使用 thread.join() 等待线程死亡
- 使用 线程池的 submit 提交任务后获取 Future 调用 get()

各自的使用场景：

- CountDownLatch 适合指定特定个数的线程执行完毕后主线程才能执行的场景（追求任意，不针对哪个线程）
  -  new CountDownLatch(5)：指定等待 5 个线程执行完
  -  在子线程尾部结束前调用  countDown() 减值，在主线程中调用 await() 进行阻塞，当减到 0 的时候，那么 await() 停止阻塞，主线程开始执行，
- CyclicBarrier 指定特定数量的线程在其指定的某个点暂停，然后统一执行（追求任意，不针对哪个线程）
  - new CyclicBarrier(5)：指定在某个暂停点等待 5 个线程
  - 各个子线程在某个位置调用 await() 陷入阻塞，当调用 await() 的线程数量到达 5 时，那么停止阻塞，开始执行，类似赛跑，跑道有 5 个位置，那么任意来齐 5 个人后，统一开始赛跑
- join() 适合少量线程的情况，因为需要调用特定线程的 join()（线程的数量确定，针对某个特定线程）
  - 线程 A 调用 线程 B 的 join()，那么线程 A 会陷入阻塞，直到线程 B 执行完毕
- 线程池的 submit + future.get() 跟 join() 差不多，同样是针对某个线程的，由于使用的是线程池，因此使用的情况是线程的数量是不确定的时候（线程的数量不确定，针对某个特定线程）
  - 提交任务时使用 submit，返回一个 Future，调用 get() 等待线程完成任务返回结果



> ### 能使用 sleep() 或者 while() 么

如果是仅仅的进行线程等待，那么可以，但是这里要求的是统一放行，**要从 能否实现 和 效率 的方面看**

如果使用的是 sleep() ，它需要指定睡眠的时间，各个线程的执行任务的时间都是不确定的，可能一个执行 100ms，一个执行 500ms，所以无法确定睡眠时间，即无法做到统一放行

如果使用的是 while()，其实也是可以的，可以使用一个 volatile boolean 变量，各个线程 自旋读取这个变量，修改后就能统一放行，但是自旋的话会一直占用 CPU，会降低 CPU 的执行效率去做这种无意义的事，当然，可能会说使用 wait() 什么的进行阻塞，但是使用 wait() 就需要用到锁，而一次只能有一个线程获取锁，怎么做到统一放行。。。





## 3、创建线程池的两种方式

1、通过 Executors 工厂创建，比如上面的 newCachedThreadPool，这里创建的都是内部定义参数

2、通过 new ThreadPoolExecutor() 自定义参数





## 4、创建线程的三种方式

这里的线程 指代的是执行体，比如 run() 或者 call()，而不是 new 出来的一个对象

- 继承 Thread，由于 Thread 本身实现了  Runnable 接口，所以它有 run()，可以直接重写 run()
- 实现 Runnable 接口 重写 run()，并通过 Thread 来运行
- 实现 Callable 接口 并实现 call()（类似 run()），同时需要使用 FutureTask 进行封装



Runnable 和 Callable 的区别：

1. Runnable 的方法体为 run()，没有返回值，Callable 的方法体为 call()，有返回值
2. Callable 由于获取返回值需要进行阻塞等待，所以需要结合 FutureTask 使用，不能单独使用





## 5、java 线程 和 操作系统线程的关系（了解即可）

[java 线程如何产生？]( https://www.cnblogs.com/lusaisai/p/12729334.html)



在 linux 系统下启动一个线程，代码如下：

```C
#include <pthread.h>//头文件
#include <stdio.h>
pthread_t pid;//定义一个变量，接受创建线程后的线程id

//定义线程的主体函数，类似于 java 中的 run()
void* thread_entity(void* arg)
{   
    printf("i am new Thread!");
}

//main方法，程序入口，main和java的main一样会产生一个进程，继而产生一个main线程
int main()
{
    /*
    调用 操作系统 的 pthread_create() 函数 创建线程，注意四个参数， 
    pid 是指针，内部创建后会将让这个指针指向线程的 id 
    NULL
    thread_entity 相当于线程指向的主体，即 java 中的 run()
    NULL
    */
    pthread_create(&pid,NULL,thread_entity,NULL);
    //usleep是睡眠的意思，那么这里的睡眠是让谁睡眠呢？
    //为什么需要睡眠？如果不睡眠会出现什么情况?? 不清楚，不想知道
    usleep(100);
}
```

我们可以看出，Linux 中创建线程是调用 pthread_create() 函数来创建线程的



在 java 中，线程的创建又是如何的呢？

```java
public static void main(String[] args) {
    new Thread().start();
}

public synchronized void start() {
    start0();
}

private native void start0();
```

当我们指向 t.start() 的时候，实际上内部调用的是一个 本地方法 start0()，本地方法不是 java 语言写的，这里的 start0() 是使用 C 写的

当我们 new Thread()  的时候实际上就是跟上面的 linux demo 一样，创建出来的只是一个线程的主体，即线程要执行的内容，而不是一个真正的线程，只有在调用 start() 后，jvm 才会创建出一个线程

我们可以猜测 java 线程创建的调用链 start() -> start0() -> pthread_create()



通过查看 openJDK，可以发现以下调用链：

```C
static JNINativeMethod methods[] = {
    {"start0",           "()V",        (void *)&JVM_StartThread},
};
```

这是一个本地方法表，start0 这个本地方法对应的就是 JVM_StartThread 方法

可以在 jvm.h 头文件（类似于 java 接口）中找到这个方法

```C
/*
 * java.lang.Thread
 */
JNIEXPORT void JNICALL
JVM_StartThread(JNIEnv *env, jobject thread);
```

然后在 jvm.h 的实现文件 jvm.cpp 中找到这个方法的具体实现

```C
JVM_StartThread(JNIEnv* env, jobject jthread){
    //xxxxx，省略代码
    
    JavaThread *native_thread = NULL;
    native_thread = new JavaThread(&thread_entry, sz);//关键代码，创建一个 javaThread
    
    //xxxxx，省略代码
}
```

再看 new JavaThread() 内部的逻辑

```C
JavaThread::JavaThread(ThreadFunction entry_point, size_t stack_sz){
    //xxx. 代码省略      
    os::create_thread(this, thr_type, stack_sz);//这里创建线程

    //xxx. 代码省略      
}
```

调用了 linux 的 create_thread() 方法，再跟进这个方法查看内部逻辑

```C
bool os::create_thread(Thread* thread, ThreadType thr_type, size_t stack_size) {
	//xxx. 代码省略      
    pthread_t tid;
    int ret = pthread_create(&tid, &attr, (void* (*)(void*)) java_start, thread);//linux系统的线程调用函数
    
    //xxx. 代码省略      
}

```

我们可以发现，最终在 create_Thread() 里调用了 pthread_create() 创建了一个线程



因此，java 线程的创建底层是调用 linux 的 pthread_create() 创建的

java 线程就是 linux 系统的线程





## 6、java 程序启动时会创建几个线程（了解即可）

可以通过打印出 JVM 中的所有线程信息

```java
public class ThreadNumDemo {
    public static void main(String[] args) {
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(false, false);
        for (ThreadInfo threadInfo : threadInfos) {
            System.out.println(threadInfo.getThreadId() + "-" + threadInfo.getThreadName());
        }
    }
}
```

输出结果为：

```java
6-Monitor Ctrl-Break	//检测死锁
5-Attach Listener		//接收外部的 jvm 命令，比如 java -version
4-Signal Dispatcher		//Attach 线程接收命令后，会交给这个线程分发到不同的模块去处理
3-Finalizer				//执行 finalize() 方法的线程
2-Reference Handler		//垃圾回收时处理 强软弱虚 引用
1-main					//主线程
```

在 JDK1.8 中，可以发现一个 java 程序启动时，会创建 6 个线程



## 7、守护线程

守护线程 跟 普通线程 的区别：守护线程 本身不会影响程序的生命周期，当一个程序的所有普通线程都执行完毕时，那么守护线程也会相应退出

一个程序的生命周期是看这些普通线程的，一旦所有的普通线程执行完成（包括主线程），那么程序也就走到头了，同时守护线程也会销毁

Thread 类中有一个 setDaemon() 方法，能将线程设置为守护线程

**不过 setDaemon() 需要在线程 start() 前进行调用，否则会抛出异常 IllegalThreadStateException**



下面是我进行测试守护线程生命周期的例子：

当初是在看守护线程是跟 父线程的生命周期有关，还是跟整个程序的生命周期有关

显然当 t1 退出后，t2 仍在继续打印，而当 main 线程退出后，t2 也退出了

```java
public abstract class A {

    public static void main(String[] args) throws Exception {
        Thread t1 = new Thread(() -> {
            Thread t2 = new Thread(() -> {
                while (true) {
                    try {
                        Thread.sleep(2000);
                        System.out.println("1");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
            //必须在现在 start() 前设置，如果线程已经运行，那么会抛出异常 IllegalThreadStateException
            t2.setDaemon(true);
            t2.start();
        });
        t1.start();
        t1.join();
        System.out.println("t1 end");
        new Scanner(System.in).nextLine();
    }
}
```



如果不将 t2 设置为守护线程，那么 t1 和 main 线程都退出后，t2 也会继续执行，导致程序不会退出

这也就是为什么 **GC 线程要设置为 守护线程** 的原因了，因为如果程序逻辑都执行完了，结果由于 GC 线程还在执行而导致程序无法正常结束，这显然是有毛病的



## 8、interrupt()、interrupted()、isInterrupted()

interrupt()：设置中断标志位，但是对于线程是否中断执行不是由操作系统决定的，而是用户自己决定的，即调用了 interrupt() 线程并不会立即中断执行

```java
public void interrupt() {
    if (this != Thread.currentThread())
        checkAccess();

    synchronized (blockerLock) {
        Interruptible b = blocker;
        if (b != null) {
            //调用 本地方法 interrupt0()，该方法不是用来中断线程，而是用来设置中断标志位
            interrupt0();           // Just to set the interrupt flag
            b.interrupt(this);
            return;
        }
    }
    interrupt0();
}
```



interrupted()：该方法是一个 static 静态方法，判断中断标志位是否为 true，并且如果为 true 那么顺便重置中断标志位为 false

```java
public static boolean interrupted() {
    return currentThread().isInterrupted(true);
}
```



isInterrupted()：判断中断标志位是否为 true，但不会重置中断标志位

```java
public boolean isInterrupted() {
    return isInterrupted(false);
}
```



可以看到，设置中断标志位的只有 interrupt()

而 interrupted() 和 isInterrupted() 只是判断中断标志位是否为 true，区别就在于一个会重置中断标志位，一个不会重置中断标志位它们内部调用的都是 方法重载的 `isInterrupted(boolean ClearInterrupted)`，它是一个本地方法

```java
private native boolean isInterrupted(boolean ClearInterrupted);
```



> #### 关于 InterruptedException 中断异常

通过上面的说法，interrupt() 并不会直接中断线程，而是通知应该中断线程，**具体的中断时机由用户决定**

我们可以发现，在很多地方的 for() ，比如 AQS、CyclicBarrier，都会去调用 isInterrupted() 去判断中断标志位，进而判断是否需要中断线程，这种都是用户自己来决定是否中断线程的

像 wait()、lock() 这种线程是直接阻塞了的，所以调用 interrupt() 也不会去中断线程，只有唤醒后它们发现设置了中断标志位了才会抛出中断异常，但是在它们 park 的过程中是无法感知的

像 wait(1000)、sleep(1000)、tryLock(1000) 这种，它们没有 park，可以当作是一个 for，每一次循环它们都会去判断是否存在中断标志位，如果存在那么直接 抛出异常
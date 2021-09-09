# CPU 缓存一致性



## 1、CPU 和 cache 的关系

随着时间的推移，CPU 和 内存的速度之间差异越来越大，CPU 每次都 读写数据都直接跟 内存打交道，那么效率将会很低

因此为了解决这个问题，在 CPU 和 内存之间引入了 高速缓冲区 cache

每个 CPU 和 内存之间都存在 3 个高速缓冲区，L1 cache、L2 cache、L3 cache

在 多核 CPU 中，**L1 cache 和 L2 cache 每个 CPU 核心 私有，L3 cache 同个 CPU 多个核心 共享**

​	L1 cache 包含 dCache（数据缓存）和 iCache（指令缓存）

L1 cache 与 CPU 核心 直接交互，读写效率最高，同时内部容量也是最小的

<img src="https://mmbiz.qpic.cn/mmbiz_png/J0g14CUwaZf0RnQxwibdcyFOTw0NvInPPKJan1icpeMMyiawV2UvVwcCayaDLWJ00D3rh78LYZqBwOv9tSTYCvRog/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" style="zoom:60%;" />





## 2、CPU 数据写回内存的方式

CPU 修改完数据将数据写回内存的方式有两种：直写 和 回写

> #### 直写

在这个方法中，当 CPU 修改了变量 a，修改完后会判断 cache 中是否存在 CPU cache 中

- 如果 CPU cache 中缓存了该变量，那么更新 CPU cache 的数据，然后再写回内存

可以看出，该方法只要进行过数据修改，都会立即将数据写回到内存中，这样的话每当 CPU 修改数据，都需要跟内存打交道，显然效率很低



> #### 写回（目前 CPU 使用的方式）

每个 cache line 都存在 1bit 来记录该 cache line 是否被修改过，如果没有被修改过，是干净的，那么为 0，如果被修改过，那么就是脏的，置为 1。	这个 1bit 称作 dirty bit

在这个方法中，CPU 修改了变量 a

- 将数据更新到 cache line 中，并且将 dirty bit 置为 1，然后不会立马将数据写回到内存中，只会在后面才写回内存

- 如果 已经被置为 1 的 cache line 需要别替换为别的 cache line 数据，那么需要将 脏的 cache line 数据写回内存中，然后再读取新的 cache line 

可以看出，该方法如果 cache line 之前没有修改过数据的话，是无需立马写回内存，会在后面的时间再写回内存，**存在 cache 和 内存的数据不一致问题**



## 3、CPU 和 cache 缓存一致性问题

```java
这里讲的 缓存一致性 只能是发生在一个 CPU 中，因为一个 CPU 同一时间只能调用一个进程，而一个 CPU 有多个 CPU 核心
一个 CPU 核心可以调用一个线程，而一个进程中的多个线程是数据共享的，并且共享的是当前进程的数据

对于不同 CPU 来说，它们调用的是不同的进程，一般情况下进程之间的数据是隔离的，是不可能互相访问的，因此不存在什么数据一致性问题
```



每个 cache 划分为多个 缓存行 (cache line)，每个缓存行 64B，之前也讲过了，这里就不讲了

问题是，内存中的每个 cache line 数据不一定只存在于 一个 CPU 核心的 cache line 中，同时也可能存在于 同一个 CPU 的其他核心 ，如果一个 CPU 核心 修改了某个 cache line 的数据，那么如果没有通知其他 CPU 核心数据失效，那么就会导致数据不一致

例子：

假设 CPU A 和 CPU B 中的 cache 中都存在 变量 i，而 CPU A 将 i = 0 修改为 i = 1，由于使用的是 写回机制，因此不会立马写回到内存中，同样的也没有通知 CPU B 它的 cache 失效

因此 CPU B 会继续使用自己 cache 中的 i = 0，导致使用的仍然是旧值

<img src="https://mmbiz.qpic.cn/mmbiz_png/J0g14CUwaZf0RnQxwibdcyFOTw0NvInPPYibKuToa682yhIE7RiaUq0KLxRNtib9EBGUe1L8ZNCBMYtVxL5EgHIfMg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" style="zoom:60%;" />





**要解决这个问题，需要引入一种机制，这个机制必须保证以下两点：**

- 1、某个 CPU 核心进行数据更新的时候，需要通知其他的 CPU 核心，让它们知道数据更新了，称作 **写传播**
- 2、多个 CPU 核心 修改同一个数据，在其他 CPU 核心中看到的数据修改顺序消息必须是一致的，即 **事务串行化**

第一条很容易理解，第二条是什么意思呢？

举例：

假设 CPU 存在 4 个 核心，它们共同操作 变量 i

假设 CPU A 将 i = 0 修改为 i = 100，CPU B 将 i = 0 修改为 i = 200，然后它们通知 CPU C 和 CPU D 知道此事

CPU C 和 CPU D 都会对数据进行同步，那么此时就存在一个问题，如果 CPU C  和 CPU D 收到数据更新的消息顺序不是同步的

比如  CPU C 先收到 i = 100 的修改消息，再收到 i = 200 的修改消息，那么最终 CPU C 的 i = 200

CPU D 先收到 i = 200 的修改消息，再收到 i = 100 的修改消息，那么最终 CPU D 的 i = 100

因此导致数据不一致

<img src="https://mmbiz.qpic.cn/mmbiz_png/J0g14CUwaZf0RnQxwibdcyFOTw0NvInPPAvJH3fcHDgr9GcU9icCCDM8mHKnQYyQ9p0JicUqEicjV4IMbfVhBETp8w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1" style="zoom:60%;" />



因此，为了防止这种事的发生，就必须保证 CPU C 和 CPU D 收到数据变化的消息顺序是一致的

事务串行化的实现必须保证以下两点：

- 实现 写传播（即 写传播 是 事务串行化的必要条件）
- 引入类似 锁机制 的机制，如果存在多个 CPU 核心存储相同的数据，那么只有获取到锁的 CPU 核心才能够修改数据，这样可以保证广播消息的顺序



## 4、总线嗅探（写传播实现）

为了解决 CPU A 修改后的数据不能被其他的 CPU 核心读取到的问题，引入了一种通知机制：总线嗅探

总线是用来传播消息的，所有的 CPU 都会去监听总线上的事件，当 CPU A 修改了变量 i 时，那么它会将修改的这个消息发布到总线上，其他 CPU 收到消息后会修改自己 CPU cache 上变量 i



总线风暴：

```
所有的 CPU 核心都会去监听总线上的数据，并且 CPU 所有的数据修改都会去发布一条消息，可能这次修改的数据只存在于修改的 CPU 上，其他 CPU 核心并没有存储该数据，那么这条消息就成了无效消息，这种情况频繁发生会加重总线的负担
```



总线嗅探仅仅是实现了写传播，它并没有实现事务的串行化，所以还是可能导致数据的不一致性



因此，引入了 **基于总线嗅探机制，同时能够解决写传播和事务串行化的 CPU 缓存一致性协议 MESI**



## 5、CPU 缓存一致性协议 MESI（写传播和事务串行化实现）

MESI 分别代表 缓存行 的 4 种状态，可以使用 2bit 来表示

- *Modified*，已修改
- *Exclusive*，独占
- *Shared*，共享
- *Invalidated*，已失效

 「已修改」 指代 cache line 被修改过，但是还没有写回到内存中，即我们上面说的 dirty bit = 1 的情况

 「已失效」 表示该 cache line 的数据已经失效，不可再使用，如果需要的话需要重新读取内存获取新值

 「独占」 表示该 cache line 中的数据只有当前 CPU 核心才具有，其他 CPU 核心没有该 cache line

 「共享」 表示该 cache line 在 多个 CPU 核心中都存在



 「独占」和「共享」 差别在于 由于独占状态时只有 CPU A 存在 i 变量，所以当 CPU A 修改这个变量时，**不需要发布广播事件**

1. CPU A 变量 i 独占 E，修改的时候直接将数据刷新到 cache，不需要发布消息，修改状态为 【已修改】
2. CPU A 变量 i 共享 S，修改的时候发布 cache 无效消息，**无效消息上包含数据的物理内存地址**，让其他 CPU 核心持有该内存地址的 cache line 修改为无效，再把数据刷新到 cache，把 cache line 设置为 【已修改】
3. CPU A 变量 i 已修改 M，再次修改时不需要设置修改
4. CPU A 读取 cache 无效或者不存在的数据，那么总线发布 read 消息，其他 CPU 没有该数据，从主存中读取，cache line 设置为 【独占】E，其他 CPU 有数据，持有该数据的 CPU 核心会将自己的 cache line 返回给 CPU A，同时都 cache line 都设置为共享 S
5. CPU A 持有 已修改 M cache line，其他 CPU 核心读取时，CPU 核心 A 将 cache line 返回给其他 CPU 核心，再将 cache 数据刷新回主存，然后都设置为 【共享】状态；其他 CPU 核心写时，CPU 核心 A 将 cache line 返回给其他 CPU 核心，再将 cache 刷新回主存，然后设置为 【无效】状态



在使用了 MESI 协议后，**「已修改」和  「独占」两种状态下的修改不会去总线发布广播事件，避免了 CPU 资源的浪费 和 减少了总线的负载**

| 状态                     | 描述                                                         | 监听任务                                                     |
| :----------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| M 修改 (Modified)        | 该Cache line有效，数据被修改了，和内存中的数据不一致，数据只存在于本 CPU Cache中。 | 缓存行必须时刻监听所有试图读该缓存行相对就主存的操作，这种操作必须在缓存将该缓存行写回主存并将状态变成S（共享）状态之前被延迟执行。 |
| E 独占、互斥 (Exclusive) | 该Cache line有效，数据和内存中的数据一致，数据只存在于本 CPU Cache中。 | 缓存行也必须监听其它缓存读主存中该缓存行的操作，一旦有这种操作，该缓存行需要变成S（共享）状态。 |
| S 共享 (Shared)          | 该Cache line有效，数据和内存中的数据一致，数据存在于多个 CPU Cache中。 | 缓存行也必须监听其它缓存使该缓存行无效或者独享该缓存行的请求，并将该缓存行变成无效（Invalid）。 |
| I 无效 (Invalid)         | 该Cache line无效。                                           | 无                                                           |





## 6、MESI 引入的问题 以及 解法方法

[MESI 的问题 以及 解决方法 - 博客园](https://www.cnblogs.com/xmzJava/p/11417943.html)

### 6.1、Store Buffers

上面说了，CPU 在将数据写回缓存时，如果 cache line 是 「共享」，那么会向总线发布广播消息，让其他 CPU 核心中的 cache line 无效，而这个过程不是跟看着的这么简单：

- CPU A 修改 i，向总线发布广播消息，CPU A 进入等待状态，不执行指令
- CPU B 和 CPU C 都监听到无效消息，将 cache line 标记为无效，然后给 CPU A 返回 ACK 反馈，表示处理完毕
- CPU A 收到 其他 CPU 的 ACK 反馈，停止等待，将数据写入到 cache 中

很显然，问题很大，当 CPU A 持有相当一部分 cache line 都处于 「共享」时，那么修改写回缓存后需要 频繁进入等待，等待的时间比执行指令的时间要长的多，这显然降低了 CPU 的效率

解决问题的思路是将数据刷回 cache line 的处理由同步转换为异步，因此出现了 Store Buffers，它在 CPU 和 cache 之间又加了一层小小的 cache，CPU 修改数据时，不会直接写回 cache，而是会写入到 Store Buffers 中，然后等待收到其他 CPU 核心的 ACK 后再将 Store Buffers 中的数据刷新回 cache



问题：为什么 CPU 需要等待收到所有的 ACK 后才将数据刷新回 cache，而不是直接刷新 cache？

```

```



对于当前 CPU 来讲，读取数据的逻辑就变成：

1. 先读取 Store Buffers
2. 再读取 CPU cache
3. 再读取主存



Store Buffers 在多 CPU 核心间的数据可见性问题：

```
Store Buffers 是 CPU 核心私有的，对于 CPU A 来说，它不存在数据可见性问题，因为它总能够读取到最新值，但是对其他 CPU 核心来说，它发布消息后 CPU A 返回的数据是 cache line 中的数据，而不是 Store Buffers 中的数据，这也就导致了其他 CPU 核心读取到的是 CPU A cache line 中的旧数据，而不是 Store Buffers 中的新数据，从而导致数据可见性问题
```

例子：

```C++
int a = 0;
int b = 0;
void foo(void)
{
 a = 1;
 b = 1;
}

void bar(void)
{
 while (b == 0) continue;
 assert(a == 1);
}
```

CPU A 访问 foo()，CPU B 访问 bar()，假设 a 被 CPU A 和 CPU B 共享，b 被 CPU A 独占

- CPU A 修改了 a，将 a = 1 存储进 Store Buffers 中，发布无效消息
- CPU B 收到无效消息，将 a cache line 修改为 【无效】状态
- CPU A 修改了 b，由于 b 是独占，所以直接修改 cache，再修改 【已修改】状态，不需要发布消息
- CPU B 读取 b，由于 b 不存在，所以发布 read b 消息，CPU A 返回 b = 1 的 cache line
- CPU B  发现 b != 0，退出循环，发现 a cache line 无效，发送 read a 消息
- CPU A 收到后返回 a = 0 的 cache line，CPU B 收到后，发现 a == 0，断言失败
- CPU A 后续再将 Store Buffers 中 a = 1 的数据写回到 cache 也已经晚了

出现上述原因是 Store Buffers 中的新值只能  CPU A 看得到，其他 CPU 核心获取到的是 cache line 的旧值

因此出现了内存屏障



### 6.3、内存屏障（Memory Barriers）

```C++
int a = 0;
int b = 0;
void foo(void)
{
 a = 1;
 smp_mb();	//内存屏障
 b = 1;
}

void bar(void)
{
 while (b == 0) continue;
 assert(a == 1);
}
```

在 a 和 b 之间添加 内存屏障 smp_mb()，它的语义是：在执行 内存屏障的下一条指令之前，需要先将 Store Buffers 中的数据写回到 cache

这样在 b = 1 执行前会将 Store Buffers 中 a = 1 的值刷回 cache，这样其他 CPU 核心读取的 a cache line 就是新值了





### 6.4、无效队列（Invalidate Queues）

引入 Store Buffers 实际上还是存在一个问题：Store Buffers 是最接近 CPU 核心的，它的容量大小并不大，在引入内存屏障后基本所有的写操作都会堆积到 Store Buffers  中，这样在 Store Buffers  满了后，还是需要等待 ACK，回到了原来的问题

解决这个问题的思路就是将 ACK 的处理由同步转变为异步，因此引入了 Invalidate Queues，每个 CPU 核心都维护一个 Invalidate Queues，用来接收无效消息

当 CPU A 发布无效消息后，CPU B 收到后不会立马去执行，而是将无效消息放入到 Invalidate Queues 中，然后立马返回一个 ACK，这样 CPU A 就大大缩短了等待的时间，而 CPU B 的 Invalidate Queues 内的无效消息会在后续进行处理

但这样也带来了一个问题，**如果 CPU B 没有及时将 cache line 设置为无效的话，那么就又会导致 数据可见性 问题**

例子：

```java
int a = 0;
int b = 0;
void foo(void)
{
 a = 1;
 smp_mb();	//内存屏障
 b = 1;
}

void bar(void)
{
 while (b == 0) continue;
 assert(a == 1);
}
```

CPU A 访问 foo()，CPU B 访问 bar()，假设 a 被 CPU A 和 CPU B 共享，b 被 CPU A 独占

- CPU A 修改 a 的值，将修改数据放入到 Store Buffers，发布 a 无效消息，修改为 【已修改】状态
- CPU B 执行 while (b == 0)，发现 cache 没有 b，发布 read b 消息
- CPU A 执行内存屏障，将 Store Buffers 中的数据刷新回 cache，发布失效消息
- CPU B 监听到失效消息，将失效消息放到 Invalidate Queues，返回一个 ACK，此时 CPU B 没有将 a cache line 设置为无效
- CPU A 修改 b 的值，发现是独占的，标记为 【已修改】，再刷新回 cache
- CPU A 收到 read b 消息，将 b cache line 返回给 CPU B
- CPU B 获取 b 的值退出循环，但是由于没有执行失效消息， cache 中 a = 0 并没有失效，导致执行读取的 cache 中 a = 0 的旧值，导致断言失败

解决方法是在读操作前面添加一个 内存屏障

```C++
int a = 0;
int b = 0;
void foo(void)
{
 a = 1;
 smp_mb();	//内存屏障
 b = 1;
}

void bar(void)
{
 while (b == 0) continue;
 smp_mb();	//内存屏障
 assert(a == 1);
}
```

这里的内存屏障的语义是 在执行下面的读操作前，先处理 Invalidate Queues 所有的失效消息，这样就会让 cache 中 a = 0 失效了，然后去到内存中同步新的值，解决数据可见性问题



### 6.5、总结

在硬件层面上， 默认实现了 总线嗅探 + CPU 缓存一致性协议 MESI，在软件层面上实现了内存屏障

**Java 中的 volatile 就是通过添加内存屏障来帮助实现可见性**

（在以前没有 MESI 的时候 voldatile 是在总线上加锁，CPU A 操作的时候不允许其他 CPU 核心读取该变量）
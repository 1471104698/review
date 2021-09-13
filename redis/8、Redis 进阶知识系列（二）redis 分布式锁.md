# redis 分布式锁



[细说分布式锁🔒](https://juejin.cn/post/6844904082860146695)

```java
1、单纯的 setnx：
最开始使用 setnx key value 指令进行加锁，此时无论 value 是什么都无所谓
问题在于此时的锁是无过期时间的，只有加锁的线程执行完成后释放锁，别的线程才能去获取锁
如果线程执行过程中宕机了，那么这把锁将得不到释放，只有程序员去手动释放



2、setnx + expire 组合
这里使用 setnx key value + expire key seconds 两条指令来解决宕机时锁的释放问题
在加锁后进行设置锁的过期时间，这样就算宕机了，那么同样也会因为过期了而得到释放

但是这个方案存在问题：setnx 和 expire 指令的执行不是原子性的，如果在执行完 setnx 后宕机了，那么同样还是会导致锁无法释放



3、使用 set 指令实现 setnx + expire 的原子性
redisTemplate 中有一个 api：Boolean setIfAbsent(K key, V value, long timeout, TimeUnit unit); 
它能够将 setnx 和 expire 两条指令原子性执行，而实际上它们并不是利用 setnx 和 expire 来实现的，而是利用 set 来实现的。
随着 redis 的发展，set 指令添加了很多的参数，其中 NX 参数可以实现 setnx 的功能，EX 参数可以实现 expire 的功能。
set 通过添加不同参数的方式来实现 setnx 和 expire 的功能组合以及原子性操作

set 命令参数如下：
	//SET key value [EX seconds|PX milliseconds] [NX|XX] [KEEPTTL]
比如我们执行：
	//set key value EX 30 NX
表示设置 如果 key 不存在，那么进行存储，同时设置 30 秒的过期时间

但是这个方案还是存在问题：线程 A 获取锁后，在过期时间内没有完成业务操作，那么锁就会释放了，这样的话假设线程 B 获取了锁，而在线程 A 执行完后，会在 finally 中将线程 B 获取的锁给释放掉，然后线程 C 获取锁，线程 B 执行完后又会将线程 C 的锁给释放掉，这样下来，这把分布式锁实际上没有起到任何的作用



4、value 参数的作用
我们上面的所有方案中，都是通过 key 来控制分布式锁的获取的，而 value 我们并没有特别的声明，而在这里，我们可以通过 value 来识别哪一个线程获取了锁。
线程在获取锁前，生成一个 uuid，作为自己的标识，然后在获取锁的时候，将 uuid 作为 value 设置进去：
	set key uuid EX 30 NX
这样在获取锁的时候，如果线程 A 获取了锁，而在过期时间内没有完成业务操作，那么自动释放锁后，线程 B 获取锁，将 value 设置为自己的 uuid，等到线程 A 执行完后，会执行 finally 中的代码块进行锁的释放：
	try{
		redisTemplate.setIfAbsent(key, uuid, 30, TimeUnit.SECOND);
	}finally{
		if(redisTemplate.get(key) == uuid){
			redisTemplate.delete(key);
		}
	}
在 finally 中它先执行 get(key)，如果 value 不是自己的 uuid，那么就不执行 delete
如果是自己的 uuid，那么再执行 delete 删除 key，这样就可以避免删除了别的线程的锁

但是这个方案还是存在问题：在 finally 中，get() 和 delete 不是原子性的，这样的话，当 get() 的时候发现 value 是自己的 uuid，但是在执行 delete 前 key 过期了，导致其他的线程先加锁了，这样就会导致删除了别的线程刚加的锁。
（这个问题是有可能发生的，因为 redis 是单线程执行用户指令，只要在线程 A get() 后别的线程在线程 A 提交 delete 指令前先一步提交 set 指令，那么就会导致这种情况）



5、lua 脚本
上面的问题的产生就是因为 get() 和 delete() 之间不存在原子性，即 get() 和 delete() 提交的时机不同，它们之间可能会插入别的线程的指令。
redis 支持 lua 脚本，它能够将多条指令作为一条指令提交给 redis 执行，我们可以通过 lua 脚本，将 get()、delete() 作为一条指令让 redis 去执行

lua 脚本如下：
    -- lua 脚本删除锁：
    -- KEYS 和 ARGV 分别是以集合方式传入的参数。
    -- 这个集合是索引下标是以 1 开头的，KEYS[1] 是我们传入的 key，ARGV[1] 是我们传入的 uuid
    -- 如果对应的 value 等于传入的 uuid。
    if redis.call('get', KEYS[1]) == ARGV[1] 
        then 
        -- 执行删除操作
            return redis.call('del', KEYS[1]) 
        else 
        -- 删除不成功，返回 0
            return 0 
    end

lua 脚本用起来很麻烦，所以出现了 redission，它是 JAVA 层面的 redis 客户端之一，可以直接用来分布式锁的加锁和解锁
redission 对外提供了 lock() 和 unlock() 两个接口，用于实现加锁和解锁，这个封装就很像 ReentrantLock
不同的是它是对 lua 脚本的封装，加锁和解锁都是使用的 lua 脚本。

这里可能有个疑问：加锁使用 set 指令不就能保证 原子性 setnx 和 expire 了吗？ 为什么还要使用 lua 脚本？
因为它内部不仅考虑到加锁，还考虑到了分布式锁的重入性，加锁的 lua 脚本大致如下：
    /*
    	KEYS[1] 是 key
    	ARGV[1] 是 过期时间
    	ARGV[2] 是 uuid
    	这里由于需要记录 key、uuid 和 重入度，所以仅仅使用一个 set 是不够的，所以这里使用 hash
    	hash 的结构为 [key field value]，在这里表示 [key, uuid, 重入度]
		
    */
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                                          //exists 命令，判断 key 是否存在，如果不存在, 执行下面代码
                                          "if (redis.call('exists', KEYS[1]) == 0) then " +
                                          	//设置 [key, field, value] 为 [key, uuid, 1]
                                          "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                                          	//设置过期时间
                                          "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                                          	//返回 null
                                          "return nil; " +
                                          	//结束
                                          "end; " +
                                          //如果存在
                                          //那么执行 hexists 命令，判断 key 下的 field = uuid 是否存在
                                          "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                                          	//如果存在，那么调用 incr 将它的重入度 +1
                                          "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                                          	//设置过期时间
                                          "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                                          "return nil; " +
                                          "end; " +
                                          //已经有线程加锁，并且加锁线程不是自己，调用 pttl 返回锁的过期时间
                                          "return redis.call('pttl', KEYS[1]);",

redission 的加锁是一段 lua 脚本，由于需要记录重入度，所以简单的的一个 set 是不够的，所以它使用 hash 来存储
	[key filed value] 表示 [key, uuid, 重入度]
加锁过程：
    1）判断 key 是否存在
    2）如果 key 不存在，表示没有线程获取锁，那么将 uuid 和 重入度 1 设置进去
    3）如果 key 存在，判断 filed 是否是当前 uuid
    4）如果 field 是当前 uuid，那么将它的重入度 +1
    5）如果 field 不是当前 uuid，那么调用 pttl 给线程返回锁的过期时间
                                          

6、redLock 红锁
redLock 出现的目的是为了解决以下问题：
	redis 集群加锁情况下，主节点 set 后告知服务器加锁成功，然而还没有将数据同步给从节点就宕机了，这样选举了一个从节点作为主节点后，另一个线程也可以获取锁成功，导致同时两个线程自认为处于持有锁的状态

redLock 要求 5 个 redis 实例，它们之间不存在任何的主从关系，即是平级的，类似 redis cluster 中的多个主节点，每次服务器请求加锁的时候，只有 (N / 2) + 1 个 redis 实例都 set 成功了，该线程才会加锁成功。解锁时需要所有实例都进行解锁才成功。
需要注意的是，加锁花费的时候必须小于锁设置的过期时间，比如设置锁的有效时间为 30s，加锁花费了 31s，那么加锁也是失败的，同时加锁的花费时间也是算在有效时间里的，比如有效时间为 30s，但是加锁花费了 3s，那么实际上线程持有锁的时间为 27s。
```


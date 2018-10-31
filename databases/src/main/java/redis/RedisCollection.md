# Redis

## 数据淘汰策略
在Redis中，允许用户设置最大使用内存大小通过配置`redis.conf`中的`maxmemory`这个值来开启内存淘汰功能，在内存限定的情况下是很有用的。设置最大内存大小可以保证Redis对外提供稳健服务。

Redis内存数据集大小上升到一定大小的时候，就会施行数据淘汰策略。Redis提供6种数据淘汰策略通过`maxmemory-policy`设置策略：
1. `volatile-lru`：从已设置过期时间的数据集（`server.db[i].expires`）中挑选最近最少使用的数据淘汰
1. `volatile-ttl`：从已设置过期时间的数据集（`server.db[i].expires`）中挑选将要过期的数据淘汰
1. `volatile-random`：从已设置过期时间的数据集（`server.db[i].expires`）中任意选择数据淘汰
1. `allkeys-lru`：从数据集（`server.db[i].dict`）中挑选最近最少使用的数据淘汰
1. `allkeys-random`：从数据集（`server.db[i].dict`）中任意选择数据淘汰
1. `no-enviction`（驱逐）：禁止驱逐数据

### 适用场景
几种策略的适用场景：
- `allkeys-lru`： 如果我们的应用对缓存的访问符合幂律分布（也就是存在相对热点数据），或者我们不太清楚我们应用的缓存访问分布状况，可以选择`allkeys-lru`策略。
- `allkeys-random`： 如果我们的应用对于缓存key的访问概率相等，则可以使用这个策略。
- `volatile-ttl`： 这种策略使得我们可以向Redis提示哪些key更适合被eviction。
- `volatile-lru`策略和`volatile-random`策略适合我们将一个Redis实例既应用于缓存和又应用于持久化存储的时候，然而我们也可以通过使用两个Redis实例来达到相同的效果，值得一提的是将key设置过期时间实际上会消耗更多的内存，因此我们建议使用`allkeys-lru`策略从而更有效率的使用内存。

## 分布式锁
分布式锁一般有数据库乐观锁、基于Redis的分布式锁以及基于ZooKeeper的分布式锁三种实现方式，而下边是第二种基于Redis的分布式锁正确的实现方法

### 加锁
```
jedis.set(lockKey, requestId, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime);
```
`set()`方法一共有五个形参：
- 第一个为`key`，我们使用`key`来当锁，因为`key`是唯一的。
- 第二个为`value`，我们传的是`requestId`，很多童鞋可能不明白，有`key`作为锁不就够了吗，为什么还要用到`value`？原因就是我们在上面讲到可靠性时，分布式锁要满足第四个条件解铃还须系铃人，通过给`value`赋值为`requestId`，我们就知道这把锁是哪个请求加的了，在解锁的时候就可以有依据。`requestId`可以使用`UUID.randomUUID().toString()`方法生成。
- 第三个为`nxxx`，这个参数我们填的是`NX`，意思是SET IF NOT EXIST，即当`key`不存在时，我们进行`set`操作；若`key`已经存在，则不做任何操作；
- 第四个为`expx`，这个参数我们传的是`PX`，意思是我们要给这个`key`加一个过期的设置，具体时间由第五个参数决定。
- 第五个为`time`，与第四个参数相呼应，代表`key`的过期时间。

总的来说，执行上面的`set()`方法就只会导致两种结果：
1. 当前没有锁（`key`不存在），那么就进行加锁操作，并对锁设置个有效期，同时`value`表示加锁的客户端。
1. 已有锁存在，不做任何操作。

### 解锁
```
String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(requestId));
```
- 第一行代码，我们写了一个简单的Lua脚本代码。Lua代码的功能：
    - 首先获取锁对应的`value`值，检查是否与`requestId`相等，如果相等则删除锁（解锁）。
    - 使用Lua语言来确保`eval()`方法是原子性的。在`eval`命令执行Lua代码的时候，Lua代码将被当成一个命令去执行，并且直到`eval`命令执行完成，Redis才会执行其他命令。
    - `Eval`命令的基本语法如下：`EVAL script numkeys key [key ...] arg [arg ...]`
- 第二行代码，我们将Lua代码传到`jedis.eval()`方法里，并使参数`KEYS[1]`赋值为`lockKey`，`ARGV[1]`赋值为`requestId`。`eval()`方法是将Lua代码交给Redis服务端执行。

### 示例
- `RedisTool`

## References
- [redis学习笔记（三）redis数据淘汰策略](https://blog.csdn.net/liubenlong007/article/details/53690103) 
- [Redis分布式锁正确的实现方法](https://www.w3cschool.cn/redis/redis-yj3f2p0c.html) 
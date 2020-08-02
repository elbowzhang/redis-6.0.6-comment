本README只是一个"快速开始"文档。你可以从这里[redis.io](https://redis.io)找到更多的细节。

什么是Redis？
--------------

Redis通常被称为*data structures*服务器。这意味着Redis可以通过一组命令提供对可变数据结构的访问，这些命令使用带有TCP套接字的*server-client*模型和一个简单的协议发送。因此，不同的流程可以以共享的方式查询和修改相同的数据结构。

在Redis中实现的数据结构有一些特殊的属性：

* Redis同样关注于将它们存储在硬盘上，即使它们总是在服务器内存中被提供和修改。这意味着Redis速度很快，同时数据也不易丢失。
* 数据结构的实现强调内存效率，因此与使用高级编程语言的数据结构相比，Redis内部的数据结构可能使用更少的内存。
* Redis提供了很多在数据库中很自然可以找到的特性，比如复制、级别可调的持久性、集群和高可用性。

另一个很好的例子是将Redis看作Memcached的一个更复杂的版本，在这个版本中，操作不仅仅是SET和GET，而是处理复杂数据类型（如列表、集合、有序数据结构等）的操作。


Redis的内部
===

如果你正在阅读README文件，其实你离源码只有一步之遥，我们在这里将解释Redis源码布局、每一个文件中的一般概念、Redis服务器中最重要的函数和结构等等。这里会面向一个较高层次而不深入细节进行讨论

源码布局
---

Redis根目录只包含了这个README文件，调用在`src`目录中真正Makefile的Makefile以及一个Redis和Sentinel（哨兵）的配置例子。你可以找到一些用于Redis、Redis集群和Redis哨兵单元测试的脚本，他们是在`tests`目录中实现的。

在根目录下有以下这些重要的目录：

* `src`：包含Redis实现，使用C语言实现
* `tests`：包含单元测试，使用Tc1实现
* `deps`：包含Redis使用到的一些库。编译Redis所需要的一切都在这个目录里；你的系统只需要提供`libc`，一个POSIO兼容的接口和一个C编译器。

注意：最新的Redis被重构了很多。函数名称和文件名已更改。

server.h
---

理解程序如何工作的最简单办法就是理解它使用的数据结构。我们将从Redis的主头文件开始，也就是`server.h`。

所有的服务器配置以及通常所有的共享状态都定义在一个名为`server`的全局结构中，类型为`struct redisServer`。这个结构中的几个重要字段为：

* `server.db`是Redis数据库的数组，数据保存在这里
* `server.commands`是命令表
* `server.clients`是连接到服务器的客户端的链表
* `server.master`是一个特殊的客户端，如果实例是一个副本，则是master。

还有很多其他的字段。大部分字段都在结构定义中有注释。

另一个重要的Redis数据结构是定义一个客户端，过去叫做`redisClient`，现在叫做`client`。这里比较重要的有：

```c
struct client {
    int fd;
    sds querybuf;
    int argc;
    robj **argv;
    redisDb *db;
    int flags;
    list *reply;
    char buf[PROTO_REPLY_CHUNK_BYTES];
    ... many other fields ...
}
```

客户端结构中定义了一个*connected client*：

* `fd`字段是客户端套接字文件描述符
* `argc`和`argv`由客户端正在执行的命令填充，以便Redis命令函数可以读取参数
* `querybuf`收集来自客户端的请求，这些请求由Redis服务器根据协议处理之后，并通过调用命令函数来执行
* `reply`和`buf`是动态和静态缓冲区，用于累计服务器发给客户端的回复。当文件描述符状态为可写时，这些缓冲区就会增量的写入套接字。

正如上面的客户端结构体中描述的那样，命令中的参数为`robj`结构。下面是完整的`robj`结构，它定义了一个*Redis object*：

```c
typedef struct redisObject{
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; // lru时间（和server.lruclock关联）
    int refcount;
    void *ptr;
}robj;
```

这个结构基本上可以表示所有基本的Redis数据类型，比如字符串，链表，集合，有序结合等等。有趣的是，它还有一个`type`字段，这样就可以知道给定对象的类型，还有一个`refcount`字段，这样同一个对象就可以在多个地方进行引用，而不需要多次分配它。最后，`ptr`字段指向对象的实际表示，即使对于相同的类型，它也可能会因为使用的`encoding`的不同而不同。

Redis对象在Redis内部被广泛使用，但是为了避免间接访问的开销，最近在很多地方我们只是用没有封装在Redis对象中的SDS。

server.c
---

这时Redis服务器的入口点，这里定义了`main()`函数。为了启动Redis服务器，以下为最重要的步骤：

* `initServerConfig()`为`server`结构体设置默认值
* `initServer()`分配操作所需要的数据结构，设置监听套接字等等
* `aeMain()`开启一个监听新连接的事件循环

在`server.c`中，你还可以找到处理Redis服务器其他重要事情的代码：

* `call()`用于在给定客户端上下文中调用给定命令
* `activeExpireCycle()`通过`EXPIRE`命令设置存活时间并处理键的回收
* `freeMemoryIfNeeded()`当需要执行一个新的写命令但是Redis内存不足时，根据`maxmemory`指令调用
* 全局变量`redisCommandTable`定义了所有的Redis命令，指定了命令的名称，命令的实现函数，命令所需的参数个数和每个命令的其他属性。

networking.c
---

该文件定义了所有的I/O函数，包括客户端、主服务器和副本（特殊客户端）：

* `createClient()`分配和初始化一个新的客户端
* `addReply*()`函数族用于命令实现，以便于向客户端结构追加数据，这些数据将被传输给客户端，作为执行给定命令的响应。
* `writeToClient()`将输出缓冲区等待的数据传输给客户端，并由*writable event handler*`sendReplyToClient()`调用
* `readQueryFromClient()`是*readable event handler*，从客户端读取数据到查询缓冲区中
* `processInputBuffer()`是一个入口，以便于根据Redis协议解析客户端查询缓冲区。一旦准备好处理命令，它就会调用在`server.c`内部定义的`processCommand()`函数来实际执行命令
* `freeClient()`释放，断开和删除一个客户端

aof.c和rdb.c
---

从名称中就可以看出这些文件实现Redis的RDB和AOF持久化。Redis使用基于`fork()`系统调用的持久性模型来创建具有与主Redis线程相同（共享）内存内容的线程。这个辅助线程将内存的内容转存到磁盘上。`rdb.c`使用它在磁盘上创建快照，`aof.c`使用它在仅追加文件太大时执行AOF重写。

`aof.c`内部实现还有额外的函数以实现一个API，该API允许命令在客户端执行新命令时追加到AOF文件中。

定义在`server.c`内部的`call()`函数负责调用将命令写入AOF的函数。

db.c
---

某些Redis命令只能对特定的数据类型进行操作，其他的都是通用的。通用型的命令比如`DEL`和`EXPIRE`。他们专门键进行操作而不是对值进行从操作。所有的这些通用型命令都定义在`db.c`内部。

此外，`db.c`实现了一个API，以便在不直接访问内部数据结构的情况下对Redis数据继进行某些操作。

在很多命令实现中都使用到了`db.c`的内部函数，其中最重要的一些是：

* `lookupKeyRead()`和`lookupKeyWrite()`用于获取一个给定键的值指针，如果键不存在返回`NULL`
* `dbAdd()`和它的高阶函数`setKey()`在一个Redis数据库中创建一个新键
* `dbDelete()`移除一个键和相对应的值
* `emptyDb()`移除整个数据库或者已定义的所有数据库

文件的其余部分实现向客户端公开的通用命令。

object.c
---

`robj`结构体定义了已经描述过的Redis对象。在`object.c`里有所有在基础层面上操作Redis对象的函数，比如分配新对象的函数，处理引用计数等。在该文件中一些值得注意的函数有：

* `incrRefcount()`和`decrRefCount()`被用来增加或者减少一个对象的引用计数。当引用计数为0时该对象被释放。
* `createObject()`分配一个新对象。还有一些专门的函数用来分配具有特定内容的字符串对象，比如`createStringObjectFromLongLong()`以及其他的类似函数。

这个文件同时还实现了`OBJECT`命令。

replication.c
---

这是Redis内部实现最复杂的文件之一，建议在熟悉了代码库的其余部分之后再使用它。这个文件实现了Redis中的master和replica角色。

`replicationFeedSlaves()`是本文件中最重要的函数之一，它向连接到master的表示为replica实例的客户端写入命令，以便replicas可以由客户端执行写操作：通过这种方式，它们的数据集将与主服务器中的数据保持同步。

该文件同时还实现了`SYNC`和`PSYNC`命令，用于在masters和replicas之间执行第一次同步，或者在断开连接后继续复制。

其他的C文件
---

* `t_hash.c`, `t_list.c`, `t_set.c`, `t_string.c`, `t_zset.c` 和 `t_stream.c` 包含了Redis数据类型的实现。它们同时还实现了访问给定数据类型的API，还有对这些数据类型的客户端命令实现。
* `ae.c`实现了Redis的事件循环，这是一个自包含的库，易于阅读和理解
* `sds.c`是Redis的字符串库
* `anet.c`是一个POSIX网络的库，与内核公开的原始结构相比，使用POSIX网络的方式更简单
* `dict.c`是增量rehash的非阻塞哈希表的实现
* `scripting.c`实现了LUA脚本。他是完全自包含的
* `cluster.c`实现了Redis哨兵。
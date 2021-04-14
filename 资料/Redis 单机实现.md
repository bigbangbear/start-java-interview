# Redis 单机数据库设计与源码赏析

> Redis 是一个开源的、基于内存的数据结构存储系统，可以作为数据库、缓存和消息中间件。在各大厂商都有广泛的应用。本篇文章是对 Redis 单机数据设计与实现的学习总结。

## 主流程

本节主要分析 Redis 启动的主流程，对其工作流程有一个整体的了解。首先从主函数 `main()` 开始。注：Redis 的源码版本为 6.0.1。

### 主函数

直接上源码：

```c
// server.c 4917
int main(int argc, char **argv) {
  ...
  initServerConfig();
  loadServerConfig(configfile,options); 
  initServer();
  aeMain(server.el);
  retunr 0;
}
```

主函数是 redis 服务启动的入口，主要进行以下操作：

![](https://github.com/yuhuibin123/imageDB/blob/master/redis_flow.png?raw=true)

### 初始化 - 默认配置

初始化一个 Redis 服务器变量 `server` 保存服务器的全局状态。`server` 的数据结构为 `redisServer` ，如下：

```c
// server.h:1029
struct redisServer {
	redisDb *db;
  int dbnum;
  dict *commands;
  aeEventLoop *el;
  int port;
}
```

初始化 server 默认配置。进行通用配置 (默认端口号、配置文件路径)、复制、集群默认、Lua 脚本、保存策略、命令表等基础配置。
```c
// server.c: 2288
void initServerConfig(void) {
	// 初始化配置文件
  server.configfile = NULL;
	// 初始化日志文件
  server.logfile = zstrdup(CONFIG_DEFAULT_LOGFILE);
	// 主从复制配置
  server.masterhost = NULL;
  ..
}
```

### 初始化 - 命令表

初始化命令表，比如 get、set、hset 等各自的处理函数，通过 hash 表进行存储，应用于后续处理请求。

Redis  中命令处理函数用 `redisCommand` 表示，如下

```c
// server.h 1445
struct redisCommand {
  char *name;	// 名称
  redisCommandProc *proc; // 处理函数
  // ...
}
```

Redis 在代码中预设命令表 `redisCommandTable` ，如下。

```c
// server.c 182
struct redisCommand redisCommandTable[] = {
	// ...
  {"get",getCommand,2,
     "read-only fast @string",
     0,NULL,1,1,1,0,0,0}, // GET 命令 
	// ...
}
```

在 `initServerConfig()` 函数中为 `redisServer` 的命令表变量 `commands` 进行初始化，如下。

```c
// server.c 2384
void initServerConfig(void) {
  server.commands = dictCreate(&commandTableDictType,NULL); // 初始化命令表
  populateCommandTable();	// 填充命令表
	// ...
}
// server.c 2982
void populateCommandTable(void) {
  for (j = 0; j < numcommands; j++) {
    struct redisCommand *c = redisCommandTable+j;
    retval1 = dictAdd(server.commands, sdsnew(c->name), c);
  }
}
```

### 初始化 - 数据库

Redis 数据库的结构体为 `redisDb` 。如下：

```c
// server.h 643
typedef struct redisDb {
    dict *dict; // 数据库键空间，保存所有的键值对
    dict *expires; // 键的过期时间
    dict *blocking_keys; // 处于阻塞状态的键
    dict *ready_keys; // 处于就绪状态的键
    dict *watched_keys; // 被 watch 命令监视的键
    int id; // 数据库 ID
    long long avg_ttl; // 评价 TTL
}
```

创建 Redis 数据库，默认创建 16 个数据库对象。

```c
void initServer(void) {
	// server.c 2750
  server.db = zmalloc(sizeof(redisDb)*server.dbnum);

  // server.c 2778
  for (j = 0; j < server.dbnum; j++) {
    server.db[j].dict = dictCreate(&dbDictType,NULL);
    server.db[j].expires = dictCreate(&keyptrDictType,NULL);
		...
  }
}
```

### 初始化 - 共享对象

Redis 基于数据结构创建了一个对象系统，包含字符串对象、列表对象、哈希对象、集合对象、有序集合对象。通过以上对象对数据进行存储等操作。

Redis 中的每个对象都由一个 redisObject 结构表示，该结构中和保存数据有关的三个属性分别是type属性、encoding 属性和 pt r属性。如下：

[https://cachecloud.github.io/2017/02/16/Redis%E5%86%85%E5%AD%98%E4%BC%98%E5%8C%96/](https://cachecloud.github.io/2017/02/16/Redis内存优化/)

```c
typedef struct redisObject {
    unsigned type:4; // 类型
    unsigned encoding:4; //
    unsigned lru:LRU_BITS; // 对象所使用的编码
    int refcount; // 引用计数
    void *ptr; // 指向对象底层实现数据结构
} robj;
```

redisObject 通过引用计数的内存回收机制，当不再使用某个对象的时候进行内存的自动释放操作。另外基于引用计数机制，可以通过共享对象来节约内存。在初始化时，还会创建一些共享变量来节约内存的使用，比如创建 10000 个整数对象。如下：

```c
// server.c 2173
void createSharedObjects(void) {
	// 客户端和服务器发送的命令 (CRLF）结尾
  shared.crlf = createObject(OBJ_STRING,sdsnew("\r\n"));
  shared.ok = createObject(OBJ_STRING,sdsnew("+OK\r\n"));
  shared.err = createObject(OBJ_STRING,sdsnew("-ERR\r\n"));
  
  // 创建 10000 个共享整数
  for (j = 0; j < OBJ_SHARED_INTEGERS; j++) {
    shared.integers[j] =
      makeObjectShared(createObject(OBJ_STRING,(void*)(long)j));
    shared.integers[j]->encoding = OBJ_ENCODING_INT;
  }
}
```

### 初始化 - 事件循环

> "**Event Loop是一个程序结构，用于等待和发送消息和事件。**（a programming construct that waits for and dispatches events or messages in a program.）"

在 Redis 进入循环事件之前，会先初始化 aeEventLoop 对象。aeEventLoop 具有两个作用：(1) 保存待处理的文件事件合时间事件；(2) 选择适合的 I/O 多路复用程序的实现。

```c
// server.c
void initServer(void) {
  // 2743
  server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);
}

// ae.c 63
aeEventLoop *aeCreateEventLoop(int setsize) {
  eventLoop->events = zmalloc(sizeof(aeFileEvent)*setsize);
  eventLoop->fired = zmalloc(sizeof(aeFiredEvent)*setsize);

  if (aeApiCreate(eventLoop) == -1) goto err;
}
```

以 epoll 作为例子，介绍 Redis 对 I/O 多路复用的封装。首先 aeApiCreate 函数，会初始化 aeApiState 对象，初始化了epoll就绪事件表，然后调用 epoll_create 创建 epoll 实例，最后赋值给 aeEventLoop 。

```c

// ae_epool.c 39
static int aeApiCreate(aeEventLoop *eventLoop) {
  aeApiState *state = zmalloc(sizeof(aeApiState));
  state->events = zmalloc(sizeof(struct epoll_event)*eventLoop->setsize);
  state->epfd = epoll_create(1024); 
  eventLoop->apidata = state;
}
```

Redis 此外还封装了 epoll 的其他几个函数：

* aeApiCreate
* aeApiAddEvent，使用 `epoll_ctl` 向 `epfd` 中添加需要监控的 FD 以及监听的事件
* aeApiPoll，将 `epoll_event` 数组中存储的信息加入 `eventLoop` 的 `fired` 数组中，将信息传递给上层模

```c
// ae.c 47
// 选择性能最好的 I/O 多路复用实现
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```

### 初始化 - 注册连接处理器

![](https://github.com/yuhuibin123/imageDB/blob/master/redis_connect.png?raw=true)

当 Redis 客户端连接服务器的时候，Redis 服务器监听的套接字准备好执行连接应答 (accept) 操作，会调用事前关联好的连接处理器进行处理。在初始化的阶段会初始化文件事件描述符 `server.ipfd`，来监听 server 配置的地址与端口。然后为对应的文件描述符注册连接处理器。

```c
// 监听地址与端口，生成文件描述符
// server.c
void initServer(void) {
  // 2754
  listenToPort(server.port,server.ipfd,&server.ipfd_count) == C_ERR)
}

// server.c 2600
int listenToPort(int port, int *fds, int *count) {
  fds[*count] = anetTcpServer(server.neterr,port,server.bindaddr[j],server.tcp_backlog);
}
```

为文件描述符注册连接处理器。

```c
// server.c
void initServer(void) {
  // 2847
  for (j = 0; j < server.ipfd_count; j++) {
    if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
                          acceptTcpHandler,NULL) 
  }
}
```

aeCreateFileEvent 接受一个套接字描述符、一个事件类型，以及事件处理器作为参数，将给定的套接字的给定事件加入到 I/O 多路复用程序的监听范围，并对事件处理器进行关联。

```c
// ae.c 153
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,aeFileProc *proc, void *clientData)
{
  aeFileEvent *fe = &eventLoop->events[fd];
  if (aeApiAddEvent(eventLoop, fd, mask) == -1)
  if (mask & AE_READABLE) fe->rfileProc = proc;
  if (mask & AE_WRITABLE) fe->wfileProc = proc;
}
```

### 开启事件循环

![](https://github.com/yuhuibin123/imageDB/blob/master/redis_loop.png?raw=true)

Redis 通过 `aeMain()` 函数进入事件主循环。当时间事件与文件事件需要被处理的时候，循环事件就会唤起各自的事件处理器。在 aeProcessEvents() 中概括了对应的处理逻辑，时间事件被自定义的逻辑唤起，文件事件在 I/O 多路复用程序通知的时被处理。在 beforesleep(eventLoop) 中会进行发送响应给客户端、刷新 AOF文件等操作。

```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```

## 线程模型

> 根据一次完整的命令请求过程了解 Redis 的线程模型。Redis 在完成初始化后，会进入事件循环，不停的进行调用 `aeProcessEvents()` 函数进行事件处理。

### 连接应答处理器

![](https://github.com/yuhuibin123/imageDB/blob/master/redis_link.png?raw=true)

当客户端与服务器建立连接的时候，初始化过程创建的文件描述符，会产生 `AE_READABLE` 事件。此时 `aeProcessEvents()` 函数会调用 I/O 多路复用程序，去获取发生待处理的事件。

```c
// ae.c 375
int aeProcessEvents(aeEventLoop *eventLoop, int flags) {
  // 433
  numevents = aeApiPoll(eventLoop, tvp);
  for (j = 0; j < numevents; j++) {
    ...
  }
}
```

此时，`aeApiPoll()` 获取到待处理的事件，`eventLoop->fired[j]`，其中文件描述符，为初始化阶段监听。根据事件掩码，调用初始化时注册的连接应答处理器 `acceptTcpHandler`进行处理。

```c
// ae.c 375
for (j = 0; j < numevents; j++) {
  aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
  int mask = eventLoop->fired[j].mask;
  int fd = eventLoop->fired[j].fd;

  fe->rfileProc(eventLoop,fd,fe->clientData,mask);
}

```

连接应答处理器，用于对连接服务器监听套接字的客户端进行应答。

```c
// networking.c
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
  while(max--) {
        cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
      acceptCommonHandler(connCreateAcceptedSocket(cfd),0,cip);
    }
}
```

首先调用 anetTcpAccept，接受客户端连接，生成文件事件描述符。

```c
// anet.c 562
int anetTcpAccept(char *err, int s, char *ip, size_t ip_len, int *port) {
    if ((fd = anetGenericAccept(err,s,(struct sockaddr*)&sa,&salen)) == -1)

      return fd;
}
```

然后调用 acceptcommonhandler 创建 redisClient 对象。

最后，注册命令请求处理器，并且把对象增加到 redisServer 的链表上。

```c
// networking.c 857
static void acceptCommonHandler(connection *conn, int flags, char *ip) {

  // 881
  if ((c = createClient(conn)) == NULL) {

  }
}

// networking.c 88
client *createClient(connection *conn) {
  client *c = zmalloc(sizeof(client));
  if (conn) {
		// 100
    connSetReadHandler(conn, readQueryFromClient);
  }

  // 167
  if (conn) linkClient(c);
} 

// connection.c 161
static inline int connSetReadHandler(connection *conn, ConnectionCallbackFunc func) {
    return conn->type->set_read_handler(conn, func);
}

// conection.c
struct connection {
    ConnectionType *type;
    int fd;
};

typedef struct ConnectionType {
    int (*set_read_handler)(struct connection *conn, ConnectionCallbackFunc handler);
} ConnectionType;

ConnectionType CT_Socket = {
    .set_read_handler = connSocketSetReadHandler,
};

static int connSocketSetReadHandler(connection *conn, ConnectionCallbackFunc func) {
    if (func == conn->read_handler) return C_OK;

    conn->read_handler = func;
    if (!conn->read_handler)
        aeDeleteFileEvent(server.el,conn->fd,AE_READABLE);
    else
        if (aeCreateFileEvent(server.el,conn->fd,
                    AE_READABLE,conn->type->ae_handler,conn) == AE_ERR) return C_ERR;
    return C_OK;
}
```

### 命令请求处理器

完成连接处理之后，服务器可以处理客户端发起的命令请求。比如，此时客户端发起命令 `SET KEY VALUE` 的命令。此时，服务器监听的文件事件描述符会产生可读的事件：

![](https://github.com/yuhuibin123/imageDB/blob/master/redis_request_command.png?raw=true)

然后，连接处理器运行时，关联的命令请求处理器 `readQueryFromClient` 会接手进行处理。

```c
// networking.c 1827
void readQueryFromClient(connection *conn) {
  // 1856
  nread = connRead(c->conn, c->querybuf+qblen, readlen);

  processInputBuffer(c);
}
```

首先通过 `connRead()` 读取输入缓冲区的数据到 `clientServer.querybuf` 。接着通过 `processInputBuffer()` 进行处理。

```c
// networking.c 1744
void processInputBuffer(client *c) {
  while(c->qb_pos < sdslen(c->querybuf)) {
    // #1
    if (c->reqtype == PROTO_REQ_INLINE) {
      if (processInlineBuffer(c) != C_OK) break;
    }

    if (c->argc == 0) {
      resetClient(c);
    } else {
    // #2
      if (processCommandAndResetClient(c) == C_ERR) {
        return;
      }
    }
  }
}
```

首先，进行请求解析`#1`，将输入缓冲区的数据进行参数解析，将结果存储在 `redisClient.argv` 字段中。然后进行命令处理，并且充值客服端数据`#2`。具体操作如下：

```c
// networking.c 1726
int processCommandAndResetClient(client *c) {
  if (processCommand(c) == C_OK) {
    commandProcessed(c);
  }
}
```

当获取到一个完整的请求命令后，会进行命令的处理流程。首先会对判断是否为退出名称，如果是会进行退出处理。然后调用 ` lookupCommand()`  在命令字典中查询命令，并且赋值给 `redisClient` 的 `cmd` 与 `lastcmd`。接着进行一系列的检查判断 Redis 是否可以执行命令，比如是否具有权限。

```c
// server.c 3368
int processCommand(client *c) {
	// 处理退出的命令
  if (!strcasecmp(c->argv[0]->ptr,"quit")) {
        addReply(c,shared.ok);
        c->flags |= CLIENT_CLOSE_AFTER_REPLY;
        return C_ERR;
    }
  
    c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
  
  // 检查 Redis 是否可以执行命令
	// 3601
 if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
    {
   // 事务
        queueMultiCommand(c);
        addReply(c,shared.queued);
    } else {
   // 直接执行命令     
   call(c,CMD_CALL_FULL);
        c->woff = server.master_repl_offset;
        if (listLength(server.ready_keys))
            handleClientsBlockedOnKeys();
    }
	}

// server.c 3058
struct redisCommand *lookupCommand(sds name) {
    return dictFetchValue(server.commands, name);
}
```

然后调用 `call()` 真正执行命令。调用 redisClient 的处理函数进行处理。

```c
// server.c 3200
void call(client *c, int flags) {
// 3227 执行命令
  c->cmd->proc(c);
}
```

进入 `setCommand()` 的处理流程，会调用 `setGenericCommand()` 进行处理

```c
// t_string.c 
void setCommand(client *c) {
	// 调用
  setGenericCommand(c,flags,c->argv[1],c->argv[2],expire,unit,NULL,NULL);
}
```

在对应的数据库添加键值对，然后增加命令回复处理。

```c
// t_string.c 68
void setGenericCommand(client *c, int flags, robj *key, robj *val, robj *expire, int unit, robj *ok_reply, robj *abort_reply) {
//87
  genericSetKey(c,c->db,key,val,flags & OBJ_SET_KEEPTTL,1);
// 93
      addReply(c, ok_reply ? ok_reply : shared.ok);

}
```

```c
// networking.c
void addReply(client *c, robj *obj) {
    if (prepareClientToWrite(c) != C_OK) return;
    if (sdsEncodedObject(obj)) {
        // 需要将响应内容添加到output buffer中。总体思路是，先尝试向固定buffer添加，添加失败的话，在尝试添加到响应链表
        if (_addReplyToBuffer(c,obj->ptr,sdslen(obj->ptr)) != C_OK)
            _addReplyObjectToList(c,obj);
    } else if (obj->encoding == OBJ_ENCODING_INT) {
        .... // 特殊情况的优化
    } else {
        serverPanic("Wrong obj->encoding in addReply()");
    }
}
```

### 命令回复

具体介绍一下，命令回复的处理流程。在命令请求处理器的最后阶段执行 `addReply()`。

首先，调用 `prepareClientToWrite()` 判断了当前 client是否需要返回数据。将client 加入到等待写入返回值队列中，下次事件周期会进行返回值写入。然后调用 `_addReplyToBuffer() `将输出的内容填充到输出缓冲区。

```c
// networking.c 220
int prepareClientToWrite(client *c) {

  if (!clientHasPendingReplies(c)) clientInstallWriteHandler(c);

}
```

然后，在下一个 eventLoop 的 beforeSleep 函数会进行处理。

```c
// server.c 2087
void beforeSleep(struct aeEventLoop *eventLoop) {
  // 2152
handleClientsWithPendingWritesUsingThreads();
}

// networking.c 
int handleClientsWithPendingWritesUsingThreads(void) {

  if (server.io_threads_num == 1 || stopThreadedIOIfNeeded()) {
        return handleClientsWithPendingWrites();
    }
}
```

如果是单线程，调用 `handleClientsWithPendingWrites` 处理 client 上等待回复的数据。

```c
// networking.c 1336
int handleClientsWithPendingWrites(void) {

      while((ln = listNext(&li))) {
      	        if (writeToClient(c,0) == C_ERR) continue;
      }
}
```

调用 `writeToClient` 把输出缓冲区的数据写入到 socket 中。

```c
// networking.c 1230
int writeToClient(client *c, int handler_installed) {
	while(clientHasPendingReplies(c)) {
        if (c->bufpos > 0) {
            nwritten = connWrite(c->conn,c->buf+c->sentlen,c->bufpos-c->sentlen);
        }
  }
}
```

调用 `connWrite` 把输出缓冲区的数据写入到 socket 传递给客户端。

```c
// connection.h 135
static inline int connWrite(connection *conn, const void *data, size_t data_len) {
    return conn->type->write(conn, data, data_len);
}

// connection.c 98
static int connSocketWrite(connection *conn, const void *data, size_t data_len) {
    int ret = write(conn->fd, data, data_len);
    if (ret < 0 && errno != EAGAIN) {
        conn->last_errno = errno;
        conn->state = CONN_STATE_ERROR;
    }

    return ret;
}
```

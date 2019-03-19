---
title: Redis PSYNC2 Bug定位
date: 2017-09-29 21:20:37
tags: golang 
---

# Redis 3.x/4.x 内部同步机制简介

## **0. 序**
本篇文章着重于介绍Redis主从复制的内部实现机制，并从源码分析和实验测试两个角度，阐述3.x和4.x两个重要版本之间的具体差异。同时，文章还简要介绍了在新版本中发现的BUG。  
在本文中，3.x具体使用的版本为3.2.9；4.x具体使用的版本为4.0.1。
## **1. Redis简介**
[Redis](https://redis.io/)是一个使用广泛的开源内存数据库，由[Salvatore Sanfilippo](http://invece.org/)创立，并由[Redis Lab](https://redislabs.com/)及开源社区共同维护开发。截至目前（2017-9-29），其最新版本为4.0.2。
## **2. Redis 3.x 主从简介**
Redis实现了一个功能精简的主从复制机制。它支持一主多从的树形主从结构，也支持链式主从结构。一主多从，或者一主一从的树形结构较为常见，通常用来做读写分离，或者热备份。
### **2.1 应用场景**
作为数据库，灾备必不可少。对于Redis来说，除了直接使用集群功能，也可以通过编写监测脚本、使用keepalived等第三方工具，以及为Redis设置合理的配置文件等方法，实现更为灵活、功能更为全面的数据库灾备方案。  
为了切合主题并精简文章篇幅，本节不详细叙述与Redis本身运行机制无关，但切实在灾备中与Redis紧密相关的其他涉众及实现方案，例如keepalived及其配置等。
#### **1) Hot standby and failover**
通常，对于一个Redis实例，会为其添加一个或多个hot standby实例，以实现热备份：
```
T(0)	Master(A) ---> 	Slave(B)
```
Master在运行过程中不断向Slave更新数据，以最大程度保证主从数据一致。当Master出现故障宕机或其他异常而无法正常工作后，Slave便可以通过执行灾备系统（如keepalived脚本）发出的指令，而将自己提升为Master:
```
T(0)	Master(A) ---> 	Slave(B)
T(1)			Master(B)
```
由于keepalived对外提供一个VIP并在内部自动做IP漂移，因此外部业务通常在经历了一小段Redis服务器无响应以后，便能够继续正常使用Redis服务。
#### **2) switchover**
另外，对调Redis主从之间的关系也非常有用。例如手动下线主服务器（A）进行维护，提升热备服务器（B）的身份为主，并继续对外提供服务。
```
T(0)	Master(A) ---> 	Slave(B)
T(1)	Master(A)	Master(B)
T(2)	Slave(A)  <---	Master(B)
```
值得注意的是，在3.x版本及更老的版本中，T(2)阶段会进行一次代价很高的全同步过程。
#### **3) failover + switchover**
如果因为某些因素，我们希望在灾备成功恢复之后，Redis服务器的主备关系能够与正常运行时一致，那么就可以通过结合1)和2)所阐述的场景进行灾备方案设计。
```
/* 灾备发生 */
T(0)	Master(A) ---> 	Slave(B)
T(1)			Master(B)		

/* 主服务恢复工作，拉取数据 */
T(2)	Master(A) 	Master(B)		
T(3)	Slave(A)  --->	Master(B)		

/* 恢复主备关系 */
T(4)	Master(A) 	Master(B)		
T(5)	Master(A) ---> 	Slave(B)
```

### **2.2 内部实现机制与PSYNC命令**
Redis主从关系的建立，以及数据在Redis实例之间的的复制都离不开一个内部指令：`PSYNC`。本小节将详细介绍`PSYNC`命令的实现，以及与其紧密相关的其他命令的原理或者设计思路。
#### **1) SLAVEOF命令**
Redis并不支持两个或多个**独立实例**之间的**直接数据同步**。若希望将*实例A*的数据复制到*实例B*上，只能通过建立两者之间的主从关系间接实现，即在B端使用`SLAVEOF`命令：
```
unix $) redis-cli SLAVEOF IP_A Port_A
```
在B端运行的Redis服务器实例收到客户端发来的`SLAVEOF`指令后，会将相关信息记录下来：即将自身的角色设置为*slave*，将master的IP地址和端口号分别记录，并将自身状态机的状态设为`REPL_STATE_CONNECT`，表示需要与master进行连接。`REPL_STATE_*`作为状态机FLAG标识，区分Redis实例所处的不同状态：
``` c
/* Set replication to the specified master address and port. */
void replicationSetMaster(char *ip, int port) {
    // ...
    server.masterhost = sdsnew(ip);
    server.masterport = port;
    // ...
    server.repl_state = REPL_STATE_CONNECT;
    // ...
}
```
至此，`SLAVEOF`命令完成了其所有任务，它既不会检测IP地址/端口号的正确性，也不会检测与master的链接是否成功，更不会开始数据同步工作。
幸运的是，一个功能与unix crontab类似的常驻函数`serverCron()`帮助Redis及时检测到用户建立主从关系的需求。`serverCron()`每1000毫秒会调用一次`replicationCron()`，`replicationCron()`会主动检查状态机标志，并接管所有与维护主从连接相关的实际任务，如超时处理等：
``` c
/* Replication cron function, called 1 time per second. */
void replicationCron(void) {
	// ...
	/* Check if we should connect to a MASTER */
    if (server.repl_state == REPL_STATE_CONNECT) {
        serverLog(LL_NOTICE,"Connecting to MASTER %s:%d",
            server.masterhost, server.masterport);
        if (connectWithMaster() == C_OK) {
            serverLog(LL_NOTICE,"MASTER <-> SLAVE sync started");
        }
    }
    // ...
}
```
函数`connectWithMaster()`实际建立起slave与master之间的非阻塞连接，并在Redis事件池中注册一个回调函数为`syncWithMaster()`的读写事件用于数据的传输：
``` c
int connectWithMaster(void) {
    int fd;

	// ...
    fd = anetTcpNonBlockBestEffortBindConnect(NULL,
        server.masterhost,server.masterport,NET_FIRST_BIND_ADDR);
	// ...

    if (aeCreateFileEvent(server.el,fd,AE_READABLE|AE_WRITABLE,syncWithMaster,NULL) ==
            AE_ERR)
    {
        // ...
    }

    server.repl_transfer_lastio = server.unixtime;
    server.repl_transfer_s = fd;
    server.repl_state = REPL_STATE_CONNECTING;
    return C_OK;
}
```
当对应的事件被Redis调度算法选中后，slave会与master进行通信协商，以完成连接检测、权限认证等通用例程。同时，slave会随之更改自身的状态机状态，并最终向master发送PSYNC命令。

#### **2) PSYNC命令**
`PSYNC`是Redis的一个内部命令，不可以被用户调用。它是驱动并维护Redis实例之间数据同步过程的最核心支柱，其格式为：
```
PSYNC   psync_runid   psync_offset
```
其中：
- `psync_runid`: redis实例唯一标识符，每次redis启动随机生成；这里填写的是slave所需连接的master 的`runid`；
- `psync_offset`:  数据复制偏移量（下一个传输字符的偏移量）。   

slave将`PSYNC`命令组装完成之后发送给master，并等待master的应答。master在收到slave发来的`PSYNC`命令之后，对其进行解析，并根据命令中的`runid`和`offset`决定如何回应此次同步请求：  

- 如果`PSYNC`命令中`psync_runid`的信息与master的`runid`不一致，或者`psync_offset`大于master能够提供的最大数据偏移，则触发全同步（FULL SYNC）处理流程：在有盘传输的配置情景下，master将fork出一个新的进程，将数据库数据以RDB文件格式写盘，并为其在调度器中注册一个事件参与调度，以实现异步传输RDB文件到slave端；
- 如果`psync_runid`与master `runid`一致，并且master能够提供足够的数据，则触发局部同步（PARTIAL SYNC）处理流程：master回复一个`+CONTINUE`命令，并根据`psync_offset`将存储在backlog中的相应数据发送给slave。

#### **3) backlog缓存区**
由上文可知，Redis实例之间的数据同步，无论是全同步，还是局部同步，主要有两种形式参与其中：RDB文件的生成和传输，以及backlog缓存区中数据的传输。其中backlog缓存区与`PSYNC`命令有着极为紧密的联系。  
backlog缓存区存在于每个redis实例，其底层实现可看作为一个环形队列，通过几个offset数值联合进行操作和维护。对于每一个对数据库数据进行更改的命令，以及一些维护实例状态的命令（如PING等），redis都会将其写入backlog。它的存在有两个目的：
1. 在进行全同步的过程中，master是异步的，其能够继续执行客户端所请求的命令。这些没有在子进程中被写入RDB文件的指令，可能依旧需要被传输到slave端以保持数据库数据一致；
2. 在某些情境下，slave端已经有了一部分，甚至绝大部分master的数据，如果因为网络分区等原因导致主从连接中断，而重新连接之后必须进行全同步，则开销太大。通过使用缓存区及维护其偏移量，以判断主从之间已经保持一致的数据量，从而略过这一部分数据的同步，减少开销。
`PSYNC`命令中所携带的`psync_offset`参数，为master指定了进行局部同步时数据传输的起始位置；master为其所有slave都维护了一个`replicaiton_offset`，以区分不同实例。

#### **4) Put it together**
##### **a. 从slave的角度**
1. `SLAVEOF`命令设置master相关信息，并设置状态机为尝试连接态；
2. `Cron`例程通过检查状态机，以启动主从连接；
3. 组装`PSYNC`命令，向master端发送；
4. 解析master返回的命令，启动对应同步例程并接受数据。
##### **b. 从master的角度**
1. 收到`PSYNC`命令请求，检查其携带的相关参数；
2. 若能够进行局部同步，则根据`offset`通过操作backlog进行少量数据传输；
3. 若只能进行全同步，则启动数据库写盘子进程，完成后在主进程中通过调度器进行异步数据传输。

### **2.3 缺陷及不足**
考虑以下场景：  
现有一个master实例和一个slave实例，在某个时刻，master挂了，slave与master失去连接。但很快，master正常重新启动。由于redis的`runid`在每次启动时随机生成，`PSYNC`检测到slave发送来的`runid`与当前不一致，因此无法继续进行局部同步，此时将开始一次全同步过程。  
在这种情境下，对于slave来说，由于其连接的始终是同一个master，且由于slave不可写的特性使得其数据不会比master更新，因此全同步是完全多余的。同理，在switchover的过程中，master和slave所维护的相同一份数据依旧需要进行全同步。

## **3. Redis 4.x 主从介绍**
### **3.1 What's new**
Redis在2017年7月正式发布了4.0.0版本，除了提供了一套完整的module API以外，其内部的同步机制得到了重大升级，新增了`PSYNC2`命令以替代老的`PSYNC`命令，但同时向下兼容3.x和4.x不同版本的混合使用。  
得益于`PSYNC2`及相关内部机制和协议的升级，2.3小节中所提到的问题在4.x版本中被妥善解决。
### **3.2 内部实现机制及PSYNC2命令**
#### **replication-id**
在4.0中，redis在原有的基础上，对主从复制的协议和机制进行的修改。其中最核心的改动，是使用`replication id`(下称`replid`)标识主从之间的关系，以代替3.x中`runid`的职责。与`runid`一致，`replid`也是一个由redis调用`/dev/random`生成的随机数，但在内部使用方面发生了较为显著的变化。  
在3.x版本中，`runid`用于唯一标识每个**实例**，相当于*身份证*。master实例通过判断`PSYNC`命令中所携带的`runid`身份信息，结合`offset`以选择使用`SYNC`还是`PSYNC`进行同步操作。也正因如此，才会导致2.3小节所提到的缺陷。  
4.x的`replid`则有了完全不同的意义，其用于唯一标识每一段**主从关系**，相当于*结婚证*。同时，每一个redis实例都会保存两个`replid`，一个标识当前主从关系，一个用于记录上一段主从关系。与之相对应的，redis也会对这两个`replid`维护两个`offset`。    
另外，不同于`runid`生成于redis的启动init阶段，并终身保持不变，`replid/replid2`的指定有多途径，并且在redis的生命周期中能够对其进行更改，其使用起来更加灵活。这些途径或时机包括：
1. 从RDB文件读取`replid`；
2. 启动时随机生成；
3. redis主从角色从slave变为master时，更换新的`replid`；
4. slave接受master指定的`replid`;
5. 每当有新的`replid`生成或被指定，使用`replid2`和`offset2`记录`replid/offset`。

#### **PSYNC2命令**
事实上，`PSYNC2`并不是一个实际存在的命令，它只是一个概念，用来与`PSYNC`的机制做出区别。slave端在同步过程中所发出的数据包中的字符串依旧包含`"PSYNC"`。不同的是，其所携带的参数由`runid`变成了上文所叙述的`replid`。不过，在主从尝试连接的过程中，`PSYNC2`被当做Redis的一种能力用于进行redis协议握手验证。
``` c
    /* Inform the master of our (slave) capabilities.
     *
     * EOF: supports EOF-style RDB transfer for diskless replication.
     * PSYNC2: supports PSYNC v2, so understands +CONTINUE <new repl ID>.
     *
     * The master will ignore capabilities it does not understand. */
    if (server.repl_state == REPL_STATE_SEND_CAPA) {
        err = sendSynchronousCommand(SYNC_CMD_WRITE,fd,"REPLCONF",
                "capa","eof","capa","psync2",NULL);
        if (err) goto write_error;
        sdsfree(err);
        server.repl_state = REPL_STATE_RECEIVE_CAPA;
        return;
    }
```

#### **replid，PSYNC2及4.x同步过程**
`replid`在主从数据复制时，所参与的几个流程大致如下：    
##### **a. slave的角度**
1. redis在启动时试图从RDB文件中读取`replid`信息，如果存在，则使用它；如果不存在，则为自己生成一个新的`replid`（潜在bug，见3.4节）；
2. 根据自身的`repli`及backlog `offset`组装`PSYNC`命令并发送；
3. 等待master回复；
4. 根据回复，遵从master进行全同步或者局部同步处理流程。若master返回的报文中更改了`replid`，则随之更改自身的`repli`。

##### **b. master的角度**
1. redis在启动时试图从RDB文件中读取`repli`信息，如果存在，则使用它；如果不存在，则为自己生成一个新的`replid`；
2. 等待slave的同步请求；
3. 请求到达，解析数据包，获取`PSYNC`命令中slave所期待的`replid`和`offset`：
	- 若命令中的`replid`与master的当前`replid`一致，则继续分析`offset`以决定进行局部同步还是全同步；
	- 若`replid`与当前`replid`不一致，则与自身维护的`replid2`及`offset2`进行分析，决定进行局部同步还是全同步。
4. 根据3.的结果给予slave应答。

### **3.3 应用场景及性能提升**
针对4.x主从同步的新特性，在具体分析过源码之后，本文对其性能提升做出了分析假设，并最终加以实验进行分析验证。
#### **1) 对于switchover的提升分析假设**
由于使用两对`replid/offset`记录维护主从关系，因此在slave变成master后，其会记录上一个主从关系的`replid`；当master更变为slave后，其`replid`保持不变。因此，在switchover之后的slave实例试图进行同步请求时，当前的master便有足够信息以触发局部同步例程。如下：
```
T(0)	Master(A)        --->       Slave(B) 
	  |- [replid0]	              |- [replid0]
		  
// B turns into a master, get a new replid
T(1)	Master(A)   	     	    Master(B) 
          |- [replid0]                |- [replid1, replid0]

// send PSYNC command with replid0, offset0
T(2)			PSYNC2(:replid0, offset0)
	Slave(A)     =============================>    Master(B) 
          |- [replid0]                                   |- [replid1, replid0]

// master agree with PSYNC, and change slave's replid
T(3)	Slave(A)        <---       Master(B) 
          |- [replid1]               |- [replid1, replid0]
```
相较于3.x，由4.x的switchover能够使用局部同步，因此在性能方面的提升应该是显著的。
#### **2) 对于failover+switchover的提升分析假设**
考虑2.1节和3.2节的内容，并结合假设1)的结论，failover+switchover的数据同步过程，应当由3.x的执行两次全同步，简化为两次局部同步，因此性能的提升也应当是显著的：
#### **3) 性能测试及结果**
##### **a. 实验环境：**  
- 环境A： 8Core-i7-6700-3.4GHz/32GB/256GB SSD  
- 环境B： 4Core-i3-3220-3.3GHz/4GB/320GB HDD  

##### **b. 实验步骤：**
0. 以环境A作为主，B作为备，B的redis保持运行，模拟A的崩溃与重启；  
1. A的redis启动后，B的redis备份A，数据同步耗时记为T1；  
2. B备份成功后，将A杀掉，将B设为主；  
3. 重启A，从B拉取数据，耗时记为T2；  
4. 对调A/B角色，即将A恢复为主，B恢复为备，耗时记为T3；  

##### **c. 实验数据：**
1. rdb file size： 44MB；
2. 重复10次failover，取各阶段平均值为一个实验数据。以下数据为最终计算后的平均数据；

| redis版本 |	T1 |	T2 |	T3 |
| ---- | ---- | ---- | ---- |
| 3.x |	4.34s |	7.448s | 4.117s |
|4.0 |4.36s | 0.2s | 4.18s |

##### **d. 实验数据分析：**
1. T1阶段，这是主备第一次建立关系，因此无论是3.x还是4.0都需要进行全同步，耗时几乎一致；
2. T2阶段，由于4.0引入了`replid`机制，此时将进行局部同步，因此耗时极短；3.x需要再一次进行全同步；
3. T3阶段，4.0从理论上说有很大可能进行局部同步，但试验中并没有抓取到任何出现局部同步的情况，实验log显示的原因是B的`offset`始终大于A。

### **3.4 存在的问题**
#### **1) 异常的failover+switchover性能**
由3.3的性能提升假设分析，并对照最终性能测试结果可以看到，redis 4.x在failover+switchover情景下的实际性能与推测不一致。具体体现在第二次局部同步没有发生，取而代之的是一次全同步的过程。  
经过对Redis源码的反复推敲，最终发现了一个出现在4.0.1及之前版本中的bug：slave的backlog没有正确初始化，导致`offset`数据维护出现异常，`PSYNC`命令在master端被驳回。详情见 [Issue 4268](https://github.com/antirez/redis/issues/4268) 及[bug fix](https://github.com/antirez/redis/commit/b75ae0bbea794dbfd6549d308ab29f9b963e0722)。
#### **2) 可能存在的数据不一致性**
除了`PSYNC`指令被错误的驳回之外，redis对于`replid`的不完善处理也有可能会导致数据不一致的情况。  
考虑两个主从实例Master A和Slave B，在A、B数据一致后停止对外服务，设此时两者的backlog `offset`都为`offset0`。然后，以主的身份重启实例B，由于redis能够从RBD文件中读取`replid`，因此B在重启后与A有着相同的`replid`。如果此时分别让A和B重新对外提供写服务，并假设在随后的某个时刻A的缓存区`backlogA`的偏移量`offsetA`，比`backlogB`的偏移量`offsetB`大，那么如果此时让B成为A的slave，redis理应将会进行一次局部同步。  
然而，但在同步后，由于`backlogA`和`backlogB`由不同的客户端操作，因此`backlogB[offset0, offsetB]`这部分数据无法保证与`backlogA[offset0, offsetB]`保持一致，数据库存在数据不一致的可能。详情见 [Issue#4316](https://github.com/antirez/redis/issues/4316)。


## **4. 写在最后**
从主从复制的角度来说，redis 4.x是一个非常大的更新和进步。它虽然巧妙的重用了绝大部分遗留代码，并兼容新老版本redis互联，但针对几个经典场景很大程度上修改了Redis内部主从复制业务逻辑。新的协议和机制增加了执行局部同步的可能性，随之为主从复制的性能带来的长足的提高，是一个非常值得使用的版本。但同时，其也存在着一些潜在的bug有待在使用过程中进行定位和修复。

## **4. 写在最后**
&emsp;&emsp;从主从复制的角度来说，redis 4.x是一个非常大的更新和进步。它虽然巧妙的重用了绝大部分遗留代码，并兼容新老版本redis互联，但针对几个经典场景很大程度上修改了Redis内部主从复制业务逻辑。新的协议和机制增加了执行局部同步的可能性，随之为主从复制的性能带来的长足的提高，是一个非常值得使用的版本。但同时，其也存在着一些潜在的bug有待在使用过程中进行定位和修复。

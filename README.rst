Redis Cluster
#######################

基于版本6.2.1

1. https://redis.io/topics/cluster-spec

2. https://redis.io/topics/cluster-tutorial

scylladb vs redis https://stackoverflow.com/questions/65257229/redis-vs-memcached-vs-scylla-cache-which-one-to-choose


overview
=====================


hash slot
---------------

redis将key分散到16384个slot中, 也就是说每个key会属于某个slot

**一个slot当然是有很多个key了, 所有hash值相等的key都在一个slot里面, 所以info keyspace中的key数量和slot不是一个东西**

.. code-block::

    HASH_SLOT = CRC16(key) mod 16384

    # 使用CLUSTER KEYSLOT命令得到key的hash值
    127.0.0.1:7000> CLUSTER KEYSLOT hello
    (integer) 866

每一个instance都会负责存储某一部分的slot(某些slot的key比较多, 某些比较少)

.. code-block::

    allenling@allenling-ubuntu:~/redis-cluster-test$ redis-cli --cluster check 127.0.0.1:7000
    127.0.0.1:7000 (53730837...) -> 0 keys | 5461 slots | 1 slaves.
    127.0.0.1:7001 (8388c0ad...) -> 0 keys | 5462 slots | 1 slaves.
    127.0.0.1:7002 (66300181...) -> 0 keys | 5461 slots | 1 slaves.
    [OK] 0 keys in 3 masters.
    0.00 keys per slot on average.
    >>> Performing Cluster Check (using node 127.0.0.1:7000)
    M: 53730837a5cf6cefd020097aa6f59351954ce679 127.0.0.1:7000
       slots:[0-5460] (5461 slots) master
       1 additional replica(s)
    M: 8388c0ad395f8e16cc810aed764ef53824807364 127.0.0.1:7001
       slots:[5461-10922] (5462 slots) master
       1 additional replica(s)
    M: 663001812cec2a4396b03e1f5904f3a780dc31b0 127.0.0.1:7002
       slots:[10923-16383] (5461 slots) master
       1 additional replica(s)
    S: 5457321529e25fc8fe73392fb7af33cbdc98f971 127.0.0.1:7005
       slots: (0 slots) slave
       replicates 53730837a5cf6cefd020097aa6f59351954ce679
    S: 27b63a8ce86942719253e1d14ae9bc0365c6a83f 127.0.0.1:7003
       slots: (0 slots) slave
       replicates 8388c0ad395f8e16cc810aed764ef53824807364
    S: 2ac082e714a16afa1c8e12bed4e26cceb6c22e51 127.0.0.1:7004
       slots: (0 slots) slave
       replicates 663001812cec2a4396b03e1f5904f3a780dc31b0
    [OK] All nodes agree about slots configuration.
    >>> Check for open slots...
    >>> Check slots coverage...
    [OK] All 16384 slots covered.

7000这个instance负责0-5461这些slot

然后我们添加10w个key, 这些key都是fooNumber=Number的形式, 比如foo0=0, foo1=1, ..., 然我们看到每个instance的keyspace(也就是key的总数)

.. code-block::

    allenling@allenling-ubuntu:~$ redis-cli -c -p 7000
    127.0.0.1:7000> info keyspace
    # Keyspace
    db0:keys=33327,expires=0,avg_ttl=0
    127.0.0.1:7000>
    allenling@allenling-ubuntu:~$ redis-cli -c -p 7001
    127.0.0.1:7001> info keyspace
    # Keyspace
    db0:keys=33370,expires=0,avg_ttl=0
    127.0.0.1:7001>
    allenling@allenling-ubuntu:~$ redis-cli -c -p 7002
    127.0.0.1:7002> info keyspace
    # Keyspace
    db0:keys=33304,expires=0,avg_ttl=0
    127.0.0.1:7002>

可以看到每个instance中的key总数差了不多, 很平均

**redis中的resharding都是基于slot的而不是key的**

multikey operation
----------------------

多key的操作(比如mget)所涉及到的key如果不在同一个slot, 那么redis cluster将会禁止该操作, 同时redis cluster提供了一个强制把某些key放置到同一个slot的办法

redis cluster只会使用key中{}中的来作为hash的内容, 所以{foo}1和{foo}2将会被放置到同一个slot, {user100}.address和{user100}.name将会放置到同一个slot

.. code-block::

    allenling@allenling-ubuntu:~$ redis-cli -c -p 7000
    127.0.0.1:7000> CLUSTER KEYSLOT {foo}1
    (integer) 12182
    127.0.0.1:7000> CLUSTER KEYSLOT {foo}2
    (integer) 12182
    127.0.0.1:7000> CLUSTER KEYSLOT {user100}.address
    (integer) 8831
    127.0.0.1:7000> CLUSTER KEYSLOT {user100}.name
    (integer) 8831

MOVE
-------------

因为客户端可以从任意一个节点去请求任何一个key, 如果该key不在客户端连接的节点中的话, 节点将会返回一个MOVE指令, 表明该key在哪个instance中

客户端需用记住这些映射关系(或者通过cluster nodes指令获取整个集群的slot和instance的映射关系)

.. code-block::

    allenling@allenling-ubuntu:~$ redis-cli -p 7001
    127.0.0.1:7001> get foo1
    (error) MOVED 13431 127.0.0.1:7002
    127.0.0.1:7001> get foo2
    "2"
    127.0.0.1:7001> get foo3
    (error) MOVED 5173 127.0.0.1:7002
    127.0.0.1:7001> get foo4
    "4"

redis-cli提供了一个-c参数, 可以切换到其他节点

.. code-block::

    allenling@allenling-ubuntu:~$ redis-cli -c  -p 7001
    127.0.0.1:7001> get foo1
    -> Redirected to slot [13431] located at 127.0.0.1:7002
    "1"
    127.0.0.1:7002> get foo2
    -> Redirected to slot [1044] located at 127.0.0.1:7001
    "2"
    127.0.0.1:7001> get foo3
    -> Redirected to slot [5173] located at 127.0.0.1:7002
    "3"
    127.0.0.1:7002> get foo4
    -> Redirected to slot [9426] located at 127.0.0.1:7001
    "4"


拓扑结构
-------------

redis中所有的instance组成网状结构, 包括master和slave, 为了可用性, 一般每个master都至少有一个slave, 这样每个master都会和其他master以及所有的slave相连, slave也是被其他master感知的

slave们会master掉线之后会选举出新的slave作为master. 每个instance都会把当前cluster的状态, 包括master, slave及其他们的状态都保存到node.conf中, 其中内容和CLUSTER NODES命令一样

.. code-block::

    127.0.0.1:7000> CLUSTER nodes
    8388c0ad395f8e16cc810aed764ef53824807364 127.0.0.1:7001@17001 master - 0 1618908104616 2 connected 5461-10922
    663001812cec2a4396b03e1f5904f3a780dc31b0 127.0.0.1:7002@17002 master - 0 1618908104515 3 connected 10923-16383
    5457321529e25fc8fe73392fb7af33cbdc98f971 127.0.0.1:7005@17005 slave 53730837a5cf6cefd020097aa6f59351954ce679 0 1618908104515 1 connected
    53730837a5cf6cefd020097aa6f59351954ce679 127.0.0.1:7000@17000 myself,master - 0 1618908104000 1 connected 0-5460
    27b63a8ce86942719253e1d14ae9bc0365c6a83f 127.0.0.1:7003@17003 slave 8388c0ad395f8e16cc810aed764ef53824807364 0 1618908104000 2 connected
    2ac082e714a16afa1c8e12bed4e26cceb6c22e51 127.0.0.1:7004@17004 slave 663001812cec2a4396b03e1f5904f3a780dc31b0 0 1618908103513 3 connected


resharding
-------------------

**redis cluster中的resharding很直接, 就是将属于某些instance的slot移动到其他某些instance中**

手动resharding的例子, 我们将1000个slot从7001, 7002移动到7000, 这个1000一般是从其他master中均与拿, 也就是7001拿500个, 7001拿500个

.. code-block::

    allenling@allenling-ubuntu:~$ redis-cli --cluster reshard 127.0.0.1:7000
    >>> Performing Cluster Check (using node 127.0.0.1:7000)
    M: 0e369abf5db945838dfb494f868f0b84cdb3789c 127.0.0.1:7000
       slots:[0-5460] (5461 slots) master
       1 additional replica(s)
    S: 845802354ad48190681991b0a0f9b1e3fc34d715 127.0.0.1:7005
       slots: (0 slots) slave
       replicates 45d60c962cc34d1ee8e5b86ea77fe23db3d8693c
    S: 204643b642876ced181b7fd9878cbceb9868627d 127.0.0.1:7004
       slots: (0 slots) slave
       replicates 0e369abf5db945838dfb494f868f0b84cdb3789c
    S: ac2a375db088eb02552e9a63b489780a9496101b 127.0.0.1:7003
       slots: (0 slots) slave
       replicates c59784aef8d409e227290df9f684f6253089142c
    M: c59784aef8d409e227290df9f684f6253089142c 127.0.0.1:7002
       slots:[10923-16383] (5461 slots) master
       1 additional replica(s)
    M: 45d60c962cc34d1ee8e5b86ea77fe23db3d8693c 127.0.0.1:7001
       slots:[5461-10922] (5462 slots) master
       1 additional replica(s)
    [OK] All nodes agree about slots configuration.
    >>> Check for open slots...
    >>> Check slots coverage...
    [OK] All 16384 slots covered
    How many slots do you want to move (from 1 to 16384)? 1000
    What is the receiving node ID? 0e369abf5db945838dfb494f868f0b84cdb3789c  # 这里是目标instance, 这里些7000的id <================================
    Please enter all the source node IDs.
      Type 'all' to use all the nodes as source nodes for the hash slots.
      Type 'done' once you entered all the source nodes IDs.
    Source node #1: all  # 这里all表示这个1000个slot是从所有其他master中拿 <==================================

    Ready to move 1000 slots.
      Source nodes:
        M: c59784aef8d409e227290df9f684f6253089142c 127.0.0.1:7002
           slots:[10923-16383] (5461 slots) master
           1 additional replica(s)
        M: 45d60c962cc34d1ee8e5b86ea77fe23db3d8693c 127.0.0.1:7001
           slots:[5461-10922] (5462 slots) master
           1 additional replica(s)
      Destination node:
        M: 0e369abf5db945838dfb494f868f0b84cdb3789c 127.0.0.1:7000
           slots:[0-5460] (5461 slots) master
           1 additional replica(s)
      Resharding plan:
        Moving slot 5461 from 45d60c962cc34d1ee8e5b86ea77fe23db3d8693c
        Moving slot 5462 from 45d60c962cc34d1ee8e5b86ea77fe23db3d8693c
        Moving slot 5463 from 45d60c962cc34d1ee8e5b86ea77fe23db3d8693c
        .... # 这里会显示从7001和7002分别拿500个slot <========================

上面会显示从7001拿5461-5961这些slot, 从7002拿10923-11421这些slot, 然后再看看cluster的nodes信息

.. code-block::

    allenling@allenling-ubuntu:~$ redis-cli -c -p 7000
    127.0.0.1:7000> CLUSTER NODES
    845802354ad48190681991b0a0f9b1e3fc34d715 127.0.0.1:7005@17005 slave 45d60c962cc34d1ee8e5b86ea77fe23db3d8693c 0 1618989512000 2 connected
    204643b642876ced181b7fd9878cbceb9868627d 127.0.0.1:7004@17004 slave 0e369abf5db945838dfb494f868f0b84cdb3789c 0 1618989512000 7 connected
    ac2a375db088eb02552e9a63b489780a9496101b 127.0.0.1:7003@17003 slave c59784aef8d409e227290df9f684f6253089142c 0 1618989511502 3 connected
    c59784aef8d409e227290df9f684f6253089142c 127.0.0.1:7002@17002 master - 0 1618989512705 3 connected 11422-16383
    0e369abf5db945838dfb494f868f0b84cdb3789c 127.0.0.1:7000@17000 myself,master - 0 1618989511000 7 connected 0-5961 10923-11421
    45d60c962cc34d1ee8e5b86ea77fe23db3d8693c 127.0.0.1:7001@17001 master - 0 1618989511000 2 connected 5962-10922

所以看到7000这个instance负责了0-5961, 以及10923-11421这些slot, 和之前0-5460的slot配置相比, 多了1000个

然后看看keyspace

.. code-block::

    allenling@allenling-ubuntu:~$ redis-cli -c -p 7000
    127.0.0.1:7000> info keyspace
    # Keyspace
    db0:keys=39418,expires=0,avg_ttl=0
    127.0.0.1:7000>
    allenling@allenling-ubuntu:~$ redis-cli -c -p 7001
    127.0.0.1:7001> info keyspace
    # Keyspace
    db0:keys=30316,expires=0,avg_ttl=0
    127.0.0.1:7001>
    allenling@allenling-ubuntu:~$ redis-cli -c -p 7002
    127.0.0.1:7002> info keyspace
    # Keyspace
    db0:keys=30267,expires=0,avg_ttl=0

和之前比, 7001和7002的keys都大概减少了3000个左右

如果master掉线怎么办
-----------------------

slave之间会发起选举, 然后获胜的slave会成为新的master, 这样系统仅仅是在选举的时候, 写入master的操作不可用, 新的master被选举之后会允许写入

.. code-block::

    m1(X) <--promote--- s1           m1(s1)

    m2    <-- s2            =>       m2

    m3    <-- 23                     m3

终止7000这个instance之后

.. code-block::

    allenling@allenling-ubuntu:~$ redis-cli -c -p 7001
    127.0.0.1:7001> CLUSTER NODES
    845802354ad48190681991b0a0f9b1e3fc34d715 127.0.0.1:7005@17005 slave 45d60c962cc34d1ee8e5b86ea77fe23db3d8693c 0 1618990021379 2 connected
    0e369abf5db945838dfb494f868f0b84cdb3789c 127.0.0.1:7000@17000 master,fail - 1618989998301 1618989997000 7 disconnected
    204643b642876ced181b7fd9878cbceb9868627d 127.0.0.1:7004@17004 master - 0 1618990020075 8 connected 0-5961 10923-11421
    ac2a375db088eb02552e9a63b489780a9496101b 127.0.0.1:7003@17003 slave c59784aef8d409e227290df9f684f6253089142c 0 1618990020877 3 connected
    45d60c962cc34d1ee8e5b86ea77fe23db3d8693c 127.0.0.1:7001@17001 myself,master - 0 1618990021000 2 connected 5962-10922
    c59784aef8d409e227290df9f684f6253089142c 127.0.0.1:7002@17002 master - 0 1618990020376 3 connected 11422-16383

看到7004这个slave成为了新的master, 7000重新启动后会变为7004的slave

.. code-block::

    allenling@allenling-ubuntu:~$ redis-cli -c -p 7001
    127.0.0.1:7001> CLUSTER NODES
    845802354ad48190681991b0a0f9b1e3fc34d715 127.0.0.1:7005@17005 slave 45d60c962cc34d1ee8e5b86ea77fe23db3d8693c 0 1618990112000 2 connected
    0e369abf5db945838dfb494f868f0b84cdb3789c 127.0.0.1:7000@17000 slave 204643b642876ced181b7fd9878cbceb9868627d 0 1618990112677 8 connected
    204643b642876ced181b7fd9878cbceb9868627d 127.0.0.1:7004@17004 master - 0 1618990112000 8 connected 0-5961 10923-11421
    ac2a375db088eb02552e9a63b489780a9496101b 127.0.0.1:7003@17003 slave c59784aef8d409e227290df9f684f6253089142c 0 1618990111675 3 connected
    45d60c962cc34d1ee8e5b86ea77fe23db3d8693c 127.0.0.1:7001@17001 myself,master - 0 1618990111000 2 connected 5962-10922
    c59784aef8d409e227290df9f684f6253089142c 127.0.0.1:7002@17002 master - 0 1618990111000 3 connected 11422-16383


如果7000掉线, 7004成为新的master, 只会7000重新上线而7004依然没有上线, 那么此时7000并不会重新变为master, 而是继续保持自己slave的身份去连接7004

**所以slave会选举为master的要求是slave正在和master连接中(或者同步完成的状态), 掉线恢复的slave如果连不到master也是不会自己选举为master的, 因为slave和master不一定是同步状态**

.. code-block::

    4613:S 25 Apr 2021 07:39:52.269 * MASTER <-> REPLICA sync started
    4613:S 25 Apr 2021 07:39:52.269 # Error condition on socket for SYNC: Connection refused
    4613:S 25 Apr 2021 07:39:53.283 * Connecting to MASTER 127.0.0.1:7004
    4613:S 25 Apr 2021 07:39:53.283 * MASTER <-> REPLICA sync started
    4613:S 25 Apr 2021 07:39:53.283 # Error condition on socket for SYNC: Connection refused
    4613:S 25 Apr 2021 07:39:54.285 * Connecting to MASTER 127.0.0.1:7004
    4613:S 25 Apr 2021 07:39:54.285 * MASTER <-> REPLICA sync started
    4613:S 25 Apr 2021 07:39:54.285 # Error condition on socket for SYNC: Connection refused

如果整个master group都掉线怎么办
--------------------------------------

也就是master和其slave都掉线, 显然我们将失去这些slot, 针对所有处于该master的slot, 都不可读, 不可写


.. code-block::

    m1(X)   <--- s1 (X)

    m2    <-- s2            =>       m2

    m3    <-- 23                     m3


新加入node
---------------------

新加入一个node的话, 使用命令 *redis-cli --cluster add-node 127.0.0.1:7006 127.0.0.1:7000*, redis-cli只是向集群中发送CLUSTER MEET

让集群中的节点去主动发现7006这个instance, 同时7006也去发现其他节点, **但是此时7006并不会自动负责slot, 此时7006的slot是空的**

所以需要自己去做一次resharding

.. code-block::

    allenling@allenling-ubuntu:~$ redis-cli -c -p 7000
    127.0.0.1:7000> CLUSTER NODES
    204643b642876ced181b7fd9878cbceb9868627d 127.0.0.1:7004@17004 master - 0 1618991738963 8 connected 0-5961 10923-11421
    45d60c962cc34d1ee8e5b86ea77fe23db3d8693c 127.0.0.1:7001@17001 master - 0 1618991739465 2 connected 5962-10922
    ac2a375db088eb02552e9a63b489780a9496101b 127.0.0.1:7003@17003 slave c59784aef8d409e227290df9f684f6253089142c 0 1618991738000 3 connected
    c59784aef8d409e227290df9f684f6253089142c 127.0.0.1:7002@17002 master - 0 1618991738000 3 connected 11422-16383
    845802354ad48190681991b0a0f9b1e3fc34d715 127.0.0.1:7005@17005 slave 45d60c962cc34d1ee8e5b86ea77fe23db3d8693c 0 1618991739000 2 connected
    0e369abf5db945838dfb494f868f0b84cdb3789c 127.0.0.1:7000@17000 myself,slave 204643b642876ced181b7fd9878cbceb9868627d 0 1618991737000 8 connected
    5b330f892f112e3858031c6931a8f1041e815b25 127.0.0.1:7006@17006 master - 0 1618991738000 0 connected  # 7006并不会去存储任何slot <=================================


删除节点
--------------

删除节点和节点崩溃的情况有点类似, 但不相同. master节点掉线的时候的时候只要有slave存活, 那么还是能保证整个系统可用

而删除节点的意思是踢出集群, 删除的是slave的话, 显然也不会有什么影响, **但是删除的是master的话, 那么显然, 我们将永远失去属于该master的所有slot!**

**所以redis集群是禁止直接把master从集群中移除的, 如果你要移除某个master, 必须对其进行resharding, 使其slot为空, 这样才能移除, 同时其slave会留在集群作为某个master的slave**

.. code-block::

    allenling@allenling-ubuntu:~$ redis-cli --cluster del-node 127.0.0.1:7000 204643b642876ced181b7fd9878cbceb9868627d
    >>> Removing node 204643b642876ced181b7fd9878cbceb9868627d from cluster 127.0.0.1:7000
    [ERR] Node 127.0.0.1:7004 is not empty! Reshard data away and try again.


一种可选的方式则是进行手动failover, 手动failover是将某个master拒绝所有客户端操作(包括读写), 然后将master和所有的slave同时数据

然后某个slave会升级为master, 而master降级为slave, 之后可以安全的踢出该master. 这个方式只是为了替换某个master节点

但是你真的减少整个集群的master个数的话, 就必须resharding然后踢除master

先把7004的slot移动到其他节点

.. code-block::

    allenling@allenling-ubuntu:~$ redis-cli --cluster reshard 127.0.0.1:7000
    >>> Performing Cluster Check (using node 127.0.0.1:7000)
    S: 0e369abf5db945838dfb494f868f0b84cdb3789c 127.0.0.1:7000
       slots: (0 slots) slave
       replicates 204643b642876ced181b7fd9878cbceb9868627d
    M: 204643b642876ced181b7fd9878cbceb9868627d 127.0.0.1:7004
       slots:[0-5961],[10923-11421] (6461 slots) master
       1 additional replica(s)
    M: 45d60c962cc34d1ee8e5b86ea77fe23db3d8693c 127.0.0.1:7001
       slots:[5962-10922] (4961 slots) master
       1 additional replica(s)
    S: ac2a375db088eb02552e9a63b489780a9496101b 127.0.0.1:7003
       slots: (0 slots) slave
       replicates c59784aef8d409e227290df9f684f6253089142c
    M: c59784aef8d409e227290df9f684f6253089142c 127.0.0.1:7002
       slots:[11422-16383] (4962 slots) master
       1 additional replica(s)
    S: 845802354ad48190681991b0a0f9b1e3fc34d715 127.0.0.1:7005
       slots: (0 slots) slave
       replicates 45d60c962cc34d1ee8e5b86ea77fe23db3d8693c
    [OK] All nodes agree about slots configuration.
    >>> Check for open slots...
    >>> Check slots coverage...
    [OK] All 16384 slots covered.
    How many slots do you want to move (from 1 to 16384)? 3230
    What is the receiving node ID? 45d60c962cc34d1ee8e5b86ea77fe23db3d8693c
    Please enter all the source node IDs.
      Type 'all' to use all the nodes as source nodes for the hash slots.
      Type 'done' once you entered all the source nodes IDs.
    Source node #1: 204643b642876ced181b7fd9878cbceb9868627d
    Source node #2: done

    Ready to move 3230 slots.
    .... # 很多key <================================


    allenling@allenling-ubuntu:~$ redis-cli --cluster reshard 127.0.0.1:7000
    >>> Performing Cluster Check (using node 127.0.0.1:7000)
    S: 0e369abf5db945838dfb494f868f0b84cdb3789c 127.0.0.1:7000
       slots: (0 slots) slave
       replicates 204643b642876ced181b7fd9878cbceb9868627d
    M: 204643b642876ced181b7fd9878cbceb9868627d 127.0.0.1:7004
       slots:[3230-5961],[10923-11421] (3231 slots) master
       1 additional replica(s)
    M: 45d60c962cc34d1ee8e5b86ea77fe23db3d8693c 127.0.0.1:7001
       slots:[0-3229],[5962-10922] (8191 slots) master
       1 additional replica(s)
    S: ac2a375db088eb02552e9a63b489780a9496101b 127.0.0.1:7003
       slots: (0 slots) slave
       replicates c59784aef8d409e227290df9f684f6253089142c
    M: c59784aef8d409e227290df9f684f6253089142c 127.0.0.1:7002
       slots:[11422-16383] (4962 slots) master
       1 additional replica(s)
    S: 845802354ad48190681991b0a0f9b1e3fc34d715 127.0.0.1:7005
       slots: (0 slots) slave
       replicates 45d60c962cc34d1ee8e5b86ea77fe23db3d8693c
    [OK] All nodes agree about slots configuration.
    >>> Check for open slots...
    >>> Check slots coverage...
    [OK] All 16384 slots covered.
    How many slots do you want to move (from 1 to 16384)? 3231
    What is the receiving node ID? c59784aef8d409e227290df9f684f6253089142c
    Please enter all the source node IDs.
      Type 'all' to use all the nodes as source nodes for the hash slots.
      Type 'done' once you entered all the source nodes IDs.
    Source node #1: 204643b642876ced181b7fd9878cbceb9868627d
    Source node #2: done

    Ready to move 3231 slots.
      Source nodes:
        M: 204643b642876ced181b7fd9878cbceb9868627d 127.0.0.1:7004
           slots:[3230-5961],[10923-11421] (3231 slots) master
           1 additional replica(s)
      Destination node:
        M: c59784aef8d409e227290df9f684f6253089142c 127.0.0.1:7002
           slots:[11422-16383] (4962 slots) master
           1 additional replica(s)
      Resharding plan:
      ... # 很多很多key <====================================

踢除7004

.. code-block::

    allenling@allenling-ubuntu:~$ redis-cli --cluster del-node 127.0.0.1:7000 204643b642876ced181b7fd9878cbceb9868627d
    >>> Removing node 204643b642876ced181b7fd9878cbceb9868627d from cluster 127.0.0.1:7000
    >>> Sending CLUSTER FORGET messages to the cluster...
    >>> Sending CLUSTER RESET SOFT to the deleted node.

我们看到7004的slave, 也就是7000这个instance, 将会作为其他某个master的slave, 这里7000就变为7002的slave

.. code-block::

    allenling@allenling-ubuntu:~$ redis-cli --cluster check 127.0.0.1:7000
    127.0.0.1:7001 (45d60c96...) -> 50028 keys | 8191 slots | 1 slaves.
    127.0.0.1:7002 (c59784ae...) -> 49973 keys | 8193 slots | 2 slaves.
    [OK] 100001 keys in 2 masters.
    6.10 keys per slot on average.
    >>> Performing Cluster Check (using node 127.0.0.1:7000)
    S: 0e369abf5db945838dfb494f868f0b84cdb3789c 127.0.0.1:7000
       slots: (0 slots) slave
       replicates c59784aef8d409e227290df9f684f6253089142c # 7002的ID <======================
    M: 45d60c962cc34d1ee8e5b86ea77fe23db3d8693c 127.0.0.1:7001
       slots:[0-3229],[5962-10922] (8191 slots) master
       1 additional replica(s)
    S: ac2a375db088eb02552e9a63b489780a9496101b 127.0.0.1:7003
       slots: (0 slots) slave
       replicates c59784aef8d409e227290df9f684f6253089142c
    M: c59784aef8d409e227290df9f684f6253089142c 127.0.0.1:7002
       slots:[3230-5961],[10923-16383] (8193 slots) master
       2 additional replica(s)
    S: 845802354ad48190681991b0a0f9b1e3fc34d715 127.0.0.1:7005
       slots: (0 slots) slave
       replicates 45d60c962cc34d1ee8e5b86ea77fe23db3d8693c

slave读写
==============

一般情况下, slave不会处理任何客户端请求, 会将读写重定向到其master

*Normally slave nodes will redirect clients to the authoritative master for the hash slot involved in a given command*

但是可以设置slave为readonly状态, 去支持大量的读. 当然这个时候会出现数据不一致(数据在master-slave之间, 或者slave-slave之间都可能不一致)


redis-cli和redis命令
=======================

redis中所有的操作都是通过发送命令来实现的, redis-cli只是帮我们把这些命令组合起来实现我们的目的, 同时还提供了一些校验等操作

比如为了添加节点, 我们需要使用CLUSTER REPLICATE, CLUSTER ADDSLOTS等命令, redis-cli就把这些命令组合起来(详细过程在下面)

自动resharding?
==================

https://stackoverflow.com/questions/31782672/why-redis-cluster-resharding-is-not-automatically

如果是指添加instance, 然后自动把其他slot给移动到新的instance, 达到整个集群每个instance的slot数量都比较均匀的话, redis cluster并不会自动resharding

也就是说当你想向集群添加一个新的instance的时候, 你必须手动去配置.

但是redis确实提供了一种另外的auto resharding方案, 在4.2的roadmap中提到

https://gist.github.com/antirez/a3787d538eec3db381a41654e214b31d

但是在这里https://github.com/redis/redis/issues/2206 和 https://github.com/redis/redis/issues/2566

看起来redis cluster只是添加了rebalance工具(命令, redis-cli --cluster rebalance), 但是并没有说你添加一个新的master, 然后集群会帮你自动

rebalance, rebalance需要管理员自己取执行命令, 不是完全"自动"


resharding的过程
========================

这里用添加instance作为例子, 添加instance就是发送概括起来就是

1. 先CLUSTER MEET命令, 让集群发现新的instance, 此时集群已经把instance注册到集群中了, 但是此时

   instance还是空的, 不负责任何slow

2. 发送CLUSTER ADDSLOTS指令去配置该instance需要存储哪些slot

3. 移动(migrating)slot到新instance

instance之间有两种方式去注册/发现集群中的instance

1. 手动发送meet, 比如A向B发送meet, 那么A和B就互相发现了. 这个方式是在创建集群的时候使用

2. 间接发现, 如果A发现B, B发现C, 那么显然A通过B知道了C, A就会去连接C. 这个方式是在添加新的instance的时候使用

新建cluster
---------------

代码cluster.c的clusterManagerCommandCreate函数

进行create之前的一些校验, 比如master_count必须大于等于3

.. code-block:: c

    if (masters_count < 3) {
        # 打印error并且退出
    }

**本地分配master和slave**

.. code-block:: c

        # 设置master
        for (i = 0; i < masters_count; i++) {
            master->dirty = 1;  // 此时是dirty状态
        }
        # 分配slave
        for (i = 0; i < masters_count; i++) {
            if (slave != NULL) {
                assigned_replicas++;
                available_count--;
                if (slave->replicate) sdsfree(slave->replicate);
                slave->replicate = sdsnew(master->name);
                slave->dirty = 1;  // dirty状态
            }
        }

**发送CLUSTER REPLICATE和CLUSTER ADDSLOTS指令**

.. code-block:: c

        if (confirmWithYes("Can I set the above configuration?", ignore_force)) {
            # 输入了yes
            while ((ln = listNext(&li)) != NULL) {
                clusterManagerNode *node = ln->value;
                char *err = NULL;
                // 这里发送指令
                int flushed = clusterManagerFlushNodeConfig(node, &err);
            }
        }


这两个指令在clusterManagerFlushNodeConfig函数中发送

.. code-block:: c

    static int clusterManagerFlushNodeConfig(clusterManagerNode *node, char **err) {
        // 如果dirty不是1, 那么退出
        if (!node->dirty) return 0;
        if (node->replicate != NULL) {
            // 指令1 <==============================
            reply = CLUSTER_MANAGER_COMMAND(node, "CLUSTER REPLICATE %s",
                                            node->replicate);
        } else {
            // 指令2 <================================
            int added = clusterManagerAddSlots(node, err);
            if (!added || *err != NULL) success = 0;
        }
        // 把dirty设置为0 <===========================
        node->dirty = 0;
    cleanup:
        if (reply != NULL) freeReplyObject(reply);
        return success;
    }


**设置每个instance的初始epcho**

发送set-config-epoch epoch_id去设置每个instance的初始epoch, 第一个instance设置为1, 第二个instance设置为2, 以此类推

如果一共由6个instance, 那么epoch就分别是1, 2, 3, 4, 5, 6


.. code-block:: c

        int config_epoch = 1;  // 在while中每次都++
        listRewind(cluster_manager.nodes, &li);
        while ((ln = listNext(&li)) != NULL) {
            clusterManagerNode *node = ln->value;
            redisReply *reply = NULL;
            reply = CLUSTER_MANAGER_COMMAND(node,
                                            "cluster set-config-epoch %d",
                                            config_epoch++);
            if (reply != NULL) freeReplyObject(reply);
        }



**发送CLUSTER MEET**

**这里谁去meet谁呢?** redis-cli向除了第一个instance之外的所有其他的instance发送cluster meet first_node

这样所有其他的instance收到meet指令只会, 就向第一个instance发送meet(ping) packet, 这样整个网络被发现完毕了

.. code-block:: c

        while ((ln = listNext(&li)) != NULL) {
            clusterManagerNode *node = ln->value;
            if (first == NULL) {  // 保存first节点
                first = node;
                continue;
            }
            redisReply *reply = NULL;
            // 每个节点都向first节点发送meet指令
            reply = CLUSTER_MANAGER_COMMAND(node, "cluster meet %s %d",
                                            first->ip, first->port);

        }

        // 等待一秒
        sleep(1);
        // 等待所有节点join完毕
        // 这里会发送cluster nodes给每一个节点
        // 检查每个节点的返回
        clusterManagerWaitForClusterJoin();


添加新的instance作为master
-----------------------------

谁meet谁?

.. code-block:: c

    static int clusterManagerCommandAddNode(int argc, char **argv) {

    clusterManagerNode *first = listFirst(cluster_manager.nodes)->value;
    listAddNodeTail(cluster_manager.nodes, new_node);
    added = 1;

    // Send CLUSTER MEET command to the new node
    clusterManagerLogInfo(">>> Send CLUSTER MEET to node %s:%d to make it "
                          "join the cluster.\n", ip, port);
    reply = CLUSTER_MANAGER_COMMAND(new_node, "CLUSTER MEET %s %d",
                                    first->ip, first->port);

    }

这里cli指示新instance发送meet包给集群的first节点, 所以其他节点会通过first节点发现新添加的instance(第二种发现instance方式)

迁移slot(在线reconfiguration)
===================================

resharding中加入了新的master之后, redis不会为新的master分配slot来达到均匀的, 需要手动为新的master分配slot

所以我们需要把某些slot从集群中移动到新加入的master中

**迁移slot主要是SETSLOT命令, SETSLOT MIGRATING, SETSLOT IMPORTING, SETSLOT slot source_node dest_node**

1. 如果source是all, 那么先均匀地从所有其他master分配要移动的slot个数

2. 逐个迁移这些slot, 每次迁移的时候, 先向目标master, 发送CLUSTER SETSLOT slot_id(slot的数字编号) IMPORTING source

   表示目标master要接收来自source的id为为slot_id的slot

3. 向source master发送CLUSTER SETSLOT slot_id MIGRATING dest, 表示source master需要向dest master迁移id为slot_id的slot

4. 向每个source master发送CLUSTER GETKEYSINSLOT slot count命令, 返回slot中的keys, 然后发送命令

   MIGRATE target_host target_port key target_database id timeout, 将key发送到目标master

   在3.0.6之后支持batch迁移key, migrate命令引入了keys参数

5. 一旦某个key迁移完成, 那么source master将会删除该key

6. 当某个slot迁移完成, 那么cli会向集群种的所有及其发送 SETSLOT <slot> NODE <node-id>命令, 表示需要修改集群配置

   slot从NODE迁移到node-id完成了.

7. 当slot正在迁移的时候, 任何请求访问slot种的key又可能返回ASK错误


ASK
--------

由于迁移某个slot的时候, slot种的key一旦迁移完成, 那么这个key在source中将会被删除

所以client请求(包括读写)slot中的某个key x的时候, 假设我们把slot从A移动到B, 有几种情况

1. 如果A发现该slot是MIGRATING状态, 同时key存在, 那么A就处理了, 表示A正在迁移这个slot但是x还没有被迁移到B

   如果key不存在, 那么向client返回ASK错误, 让客户端去请求B, 因为可能key已经被迁移到B了

2. B发现该slot是IMPORTING状态, 如果客户端发送请求之前没有发送ASK, 那么将会返回MOVE错误给客户端

   这是因为x可能在A, 你应该先问A, 如果客户端发送请求之前发送了ASK, 那么B就会处理该请求

3. 因为迁移是从A到B, 所以必须强制要求先问A, 再问B

4. 所以在slot完成迁移之前, 所有的query都会有额外的ASK消耗

5. 一旦迁移完成, A就会向客户但发送MOVE指令, 表示slot现在由B负责, 客户端需要更新instance/slot的关系

6. 那么迁移的是写入slot呢? 也会走1的步骤, 也就是如果slot是MIGRATING状态, 同时key还在, 那么可以写入, 否则去请求B


Rebalance
==============

rebalance是redis自动将slot"均匀地"分配到各个instance中, 可以指定各个instance的weight等等


写入丢失
==============

redis cluster中写入会从master以异步的方式备份到slave, 所以如果当master向slave发起备份之前master掉线, 在经过某个时间(cluster_node_timeout)之后

那么某个slave会进行选举称为新的master, 显然, 这些写入都会全部丢失

还有一种可能写入丢失的情况是网络分区

1. A由于网络分区是unreachable状态

2. A的slave A1也无法连接到A, 那么A1将会进行选举

3. 之后网络分区被修复, A重新能和其他instance互联

4. 一个client继续向A发起写入请求, 这是可能的, 因为这个client的映射表还没有更新, 并不知道A已经被A1给取代了, 此时向A写入的请求可能

   在A被转为slave之后丢失了.

   也就是所A重新是reachable状态之后, A自己依然觉得自己是master, 将会接收写入请求, 但是经过和集群交换ping/pong之后发现自己应该是slave

   那么将会和master进行同步, 此时写入丢失了

这个情况下也不太可能发生. 因为一旦A发现和大多数master连接超时, 这个超时时间是cluster_node_timeout, 当然可配置为连不上任一一个master, 之后, 将会拒绝写入

当网络分区被修复之后, 会强制等待一段时间和其他节点同步ping/pong中的configure信息, 这个时间内也是无法写入的. 这个时间是CLUSTER_WRITABLE_DELAY, 强制为2s

同时该master需要rejoin到进群, 同样需要等待一段时间才能接收query.

**总而言之, redis使用强制等待一段时间的方式去阻止一个状态从fail到ok的master接收写入, 从而阻止写丢失.**

master如何降级为slave在下面选举部分


.. code-block:: c

    #define CLUSTER_MAX_REJOIN_DELAY 5000  // rejoin的最小和最大等待时间  <=====================
    #define CLUSTER_MIN_REJOIN_DELAY 500
    #define CLUSTER_WRITABLE_DELAY 2000  // 2s的写延迟  <=======================
    
    void clusterUpdateState(void) {
    
        /* If this is a master node, wait some time before turning the state
         * into OK, since it is not a good idea to rejoin the cluster as a writable
         * master, after a reboot, without giving the cluster a chance to
         * reconfigure this node. Note that the delay is calculated starting from
         * the first call to this function and not since the server start, in order
         * to don't count the DB loading time. */
        if (first_call_time == 0) first_call_time = mstime();
        // 这里是阻止一个处于分区状态的master重启的时候, 读取旧的配置发现自己是fail并且是master
        // 但是我们不知道我们已经处于分区状态多久了, 此时走到下面的
        // rejoin逻辑的话, 会发现其实我们是在线状态, 就直接允许写入, 这个也是不允许的  <===========================
        // 所以这里first_call_time经过第一次修改之后就不会变了
        if (nodeIsMaster(myself) &&
            server.cluster->state == CLUSTER_FAIL &&
            mstime() - first_call_time < CLUSTER_WRITABLE_DELAY) return;


        // 记录下我们处于分区状态的时间
        // 重启就丢失, 所以需要上面的判断 <==================
        /* If we are in a minority partition, change the cluster state
         * to FAIL. */
        {
            int needed_quorum = (server.cluster->size / 2) + 1;

            if (reachable_masters < needed_quorum) {
                new_state = CLUSTER_FAIL;
                among_minority_time = mstime();
            }
        }


        // 需要rejoin
        if (new_state != server.cluster->state) {
            mstime_t rejoin_delay = server.cluster_node_timeout;
    
            // 至少等待这些多的时间
            /* If the instance is a master and was partitioned away with the
             * minority, don't let it accept queries for some time after the
             * partition heals, to make sure there is enough time to receive
             * a configuration update. */
            if (rejoin_delay > CLUSTER_MAX_REJOIN_DELAY)
                rejoin_delay = CLUSTER_MAX_REJOIN_DELAY;
            if (rejoin_delay < CLUSTER_MIN_REJOIN_DELAY)
                rejoin_delay = CLUSTER_MIN_REJOIN_DELAY;
    
            // 如果我们当前是ok, 但是等待时间没有达到rejoin_delay
            // 那么继续是fail状态, 等待同步configuration <============================
            if (new_state == CLUSTER_OK &&
                nodeIsMaster(myself) &&
                mstime() - among_minority_time < rejoin_delay)
            {
                return;
            }
    
    }


Ping/Pong和Gossip消息
============================

instance之间会发送ping给其他instance, 每个ping都会触发其他instance返回pong, 但是pong也可以主动发送, 比如configuration改变的时候

同时如果instance发现某个instance是连接不到了, 那么它会把这个状态给广播给其他instance, 这个消息称为gossip消息

**gossip消息是某个instance, 称为sender, 假设为A, 向我们, 假设为B, 发送, A自己保存的关于另外一个instance, 假设为C, 的状态的消息.**

如果一个packet的type是ping/pong/meet, 那么该packet就需要处理gossip信息

ping的策略是

1. 每秒采样5次, 选择收到该instance的pong时间最小的instance作为ping的目标

2. 同时为了保证在cluster_node_timeout/2内ping完所有的instance

   我们必须检查所有的instance, 如果我们收到该instance的pong时间和当前时间差大于cluster_node_timeout / 2的话, 重新ping该instance

.. code-block:: c

    /* This is executed 10 times every second */
    // 每秒执行10次
    void clusterCron(void) {
        iteration++; /* Number of times this function was called so far. */
        // 每秒执行10次, 每10次执行一次ping, 所以每秒ping一个instance
        if (!(iteration % 10)) {
            int j;
            /* Check a few random nodes and ping the one with the oldest
             * pong_received time. */
            // 采样5次  <======================================
            for (j = 0; j < 5; j++) {
                de = dictGetRandomKey(server.cluster->nodes);
                clusterNode *this = dictGetVal(de);
                /* Don't ping nodes disconnected or with a ping currently active. */
                if (this->link == NULL || this->ping_sent != 0) continue;
                if (this->flags & (CLUSTER_NODE_MYSELF|CLUSTER_NODE_HANDSHAKE))
                    continue;
                // 比较我们收到目标节点的时间, 取最小的  <==========================
                if (min_pong_node == NULL || min_pong > this->pong_received) {
                    min_pong_node = this;
                    min_pong = this->pong_received;
                }
            }
            // ping目标节点   <===============================
            if (min_pong_node) {
                serverLog(LL_DEBUG,"Pinging node %.40s", min_pong_node->name);
                // 发送ping  <========================
                clusterSendPing(min_pong_node->link, CLUSTERMSG_TYPE_PING);
            }
        }

        // 为了保证在cluster_node_timeout / 2时间内ping完所有的节点  <====================================
        di = dictGetSafeIterator(server.cluster->nodes);
        while((de = dictNext(di)) != NULL) {
            clusterNode *node = dictGetVal(de);
            now = mstime(); /* Use an updated time at every iteration. */

            /* If we have currently no active ping in this instance, and the
             * received PONG is older than half the cluster timeout, send
             * a new ping now, to ensure all the nodes are pinged without
             * a too big delay. */
            if (node->link &&
                node->ping_sent == 0 &&
                (now - node->pong_received) > server.cluster_node_timeout/2)
            {
                clusterSendPing(node->link, CLUSTERMSG_TYPE_PING);
                continue;
            }
        }

    }


ping网络包太多
---------------------

显然单纯按照上面的策略, ping包会很多, 在https://github.com/redis/redis/issues/3929这里作者做出了一些改进

目标是要改进第二个发送条件, 也就是如果pong包的接收时间太老(小于now-cluster_node_timeout/2), 那么直接发送ping

这里如果A向我们发送的gossip中表示C还在线, 同时C在我们B这里发现当前没有发送ping(可能上一轮没有被选中为目标去ping), 那么我们选择相信A

既然相信A, 那么我们就没必要向C发送Ping, 为了阻断上一节中的cluster_node_timeout/2的判断条件, 我们这里设置一个合适的pong_received时间

如果A收到C的pong的时间小于当前时间+500ms, 并且大于B收到C的pong时间, 那么我们B就选择A收到C的pong的时间作为B收到C的pong的时间

**简单来说就是, 我们以A收到C的pong时间为准, 如果A收到C的pong时间比B收到C的pong时间更大, 但是不能太大(当前时间+500ms)**

.. code-block:: c

    // 处理gossip消息
    void clusterProcessGossipSection(clusterMsg *hdr, clusterLink *link) {
        uint16_t count = ntohs(hdr->count);
        clusterMsgDataGossip *g = (clusterMsgDataGossip*) hdr->data.ping.gossip;
        // 这个是sender  <===========================
        clusterNode *sender = link->node ? link->node : clusterLookupNode(hdr->sender);
        while(count--) {
            // 拿到目标instance  <==========================
            uint16_t flags = ntohs(g->flags);
            clusterNode *node;
            sds ci;
            // 
            node = clusterLookupNode(g->nodename);
            if (node) {

                // 这里就是改进方案  <===========================
                /* If from our POV the node is up (no failure flags are set),
                 * we have no pending ping for the node, nor we have failure
                 * reports for this node, update the last pong time with the
                 * one we see from the other nodes. */
                // flags是从g中拿到的   <========================
                // 如果flags不是PFAIL或者FAIL, 那么显然sender说node是在线状态, 我们选择相信  <=================
                if (!(flags & (CLUSTER_NODE_FAIL|CLUSTER_NODE_PFAIL)) &&
                    node->ping_sent == 0 &&
                    clusterNodeFailureReportsCount(node) == 0)
                {
                    mstime_t pongtime = ntohl(g->pong_received);
                    pongtime *= 1000; /* Convert back to milliseconds. */

                    /* Replace the pong time with the received one only if
                     * it's greater than our view but is not in the future
                     * (with 500 milliseconds tolerance) from the POV of our
                     * clock. */
                    // 选个最大时间, 如果sender收到node的pong时间小于当前instance的时间+500ms, 并且
                    // 大于我们收到的node的pong时间  <==========================
                    if (pongtime <= (server.mstime+500) &&
                        pongtime > node->pong_received)
                    {
                        node->pong_received = pongtime;
                    }
                }
            }
        }
    }

ping/pong中的时间消息属性
---------------------------------

1. 而ping_sent在发送ping的时候设置上当前时间

2. ping_sent和pong_received在收到pong包之后会被分别设置为0和now

   所以通过ping_sent为0判断当前的ping是否收到了pong

3. data_received则是收到任何消息之后就更新该属性

.. code-block:: c

    int clusterProcessPacket(clusterLink *link) {

        if (sender) sender->data_received = now; // 不管怎么样, 更新data_received  <=====================

        /* PING, PONG, MEET: process config information. */
        if (type == CLUSTERMSG_TYPE_PING || type == CLUSTERMSG_TYPE_PONG ||
            type == CLUSTERMSG_TYPE_MEET)
        {
            if (link->node && type == CLUSTERMSG_TYPE_PONG) {
                link->node->pong_received = now;
                link->node->ping_sent = 0;
            }


            // 最后会处理gossip消息  <=======================
            if (sender) clusterProcessGossipSection(hdr,link);
        }

    }

设置ping_sent为当前时间

.. code-block:: c

    void clusterSendPing(clusterLink *link, int type) {

        if (link->node && type == CLUSTERMSG_TYPE_PING)
            link->node->ping_sent = mstime();

    }

检测instance掉线
===========================

0. 如果没有向某个节点发送ping, 也就是没有等待中的ping, 那么就不管

1. 如果我们和某个instance的ping/pong超时了, 那么对方可能掉线了, 然后设置对方节点为PFAIL，可能已经掉线

   这时在定时任务中会校验到这个instance是PFAIL状态, 那么就在发送ping的时候带上这个PFAIL信息

2 **每个instance收到PFAIL的gossip消息只会, 判断对方是否是master, 如果是master, 那么把这个PFAIL加入到本地记录中**

   比如A把C标识为PFAIL, 然后B收到了A的消息, 如果A是master, 那么把这个PFAIL加入到本地PFAIL列表

   如果我们能连接到C, 那么不管, 直接退出

3. 如果一个我们发现我们某个instance是PFAIL状态, 那么再查找是否大多数master节点都把该instance设置为PFAIL, 如果是, 那么设置该instance为FAIL状态

   同时向其他节点广播该instance已经FAIL. 这里每次收到pong的时候记录下sender关于其他节点的状态, 是否为PFAIL或者FAIL

   存储在本地, 然后查找的时候没必要向其他节点请求了只是查表

4. A如果发现B是PFAIL, 同时查找本地保存的表, 如果发现大多数节点都把B设置为PFAIL, 那么A就把C设置为FAIL状态

   A向其他instance广播C是FAIL的消息, FAIL消息会强制其他节点把C设置为FAIL状态

5. 状态转移的单向的, PFAIL => FAIL => None, 或者PFAIL => None

6. **这里注意一点是, 只有master的FPAIL是有用的, 但是发送FAIL这个强制消息是所有人都可以的**

   比如SLAVE C1发现大多数master节点把MASTER C设置为PFAIL了, 同时自己也不能连到C, 那么C1就会发送FAIL, 那么其他所有节点都会收到FAIL消息, 强制把C状态设置为FAIL

设置PFAIL状态
------------------

.. code-block:: c

    /* This is executed 10 times every second */
    void clusterCron(void) {

        di = dictGetSafeIterator(server.cluster->nodes);
        while((de = dictNext(di)) != NULL) {
            clusterNode *node = dictGetVal(de);
            /* If we are not receiving any data for more than half the cluster
             * timeout, reconnect the link: maybe there is a connection
             * issue even if the node is alive. */
            // 计算ping_delay和data_delay  <======================
            mstime_t ping_delay = now - node->ping_sent;
            mstime_t data_delay = now - node->data_received;
            // 判断是否需要重连   <===========================
            if (node->link && /* is connected */
                now - node->link->ctime >
                server.cluster_node_timeout && /* was not already reconnected */
                node->ping_sent && /* we already sent a ping */
                node->pong_received < node->ping_sent && /* still waiting pong */
                /* and we are waiting for the pong more than timeout/2 */
                ping_delay > server.cluster_node_timeout/2 &&
                /* and in such interval we are not seeing any traffic at all. */
                data_delay > server.cluster_node_timeout/2)
            {
                /* Disconnect the link, it will be reconnected automatically. */
                freeClusterLink(node->link);
            }

            // 如果没有发送ping, 那就不管它  <=============================
            /* Check only if we have an active ping for this instance. */
            if (node->ping_sent == 0) continue;
    
            /* Check if this node looks unreachable.
             * Note that if we already received the PONG, then node->ping_sent
             * is zero, so can't reach this code at all, so we don't risk of
             * checking for a PONG delay if we didn't sent the PING.
             *
             * We also consider every incoming data as proof of liveness, since
             * our cluster bus link is also used for data: under heavy data
             * load pong delays are possible. */
            // 1. 如果没有收到pong, 那么data_received为0, 那么node_delay取ping_delay
            // 2. 如果收到了pong, 那么ping_sent为0, 取data_received
            // 这里是取最小值  <======================================
            mstime_t node_delay = (ping_delay < data_delay) ? ping_delay :
                                                              data_delay;
    
            // 超时时间大于cluster_node_timeout, 设置instance为PFAIL状态  <===========================
            if (node_delay > server.cluster_node_timeout) {
                /* Timeout reached. Set the node as possibly failing if it is
                 * not already in this state. */
                if (!(node->flags & (CLUSTER_NODE_PFAIL|CLUSTER_NODE_FAIL))) {
                    serverLog(LL_DEBUG,"*** NODE %.40s possibly failing",
                        node->name);
                    node->flags |= CLUSTER_NODE_PFAIL;
                    update_state = 1;
                }
            }
    
        }
    
    }


定时任务通过ping发送PFAIL状态
-------------------------------------

注意server.cluster->stats_pfail_nodes这个属性

.. code-block:: c

    /* This is executed 10 times every second */
    void clusterCron(void) {
    
        di = dictGetSafeIterator(server.cluster->nodes);
        server.cluster->stats_pfail_nodes = 0;  // 标志为先置为0  <==============================
    
        while((de = dictNext(di)) != NULL) {
            clusterNode *node = dictGetVal(de);
    
            /* Not interested in reconnecting the link with myself or nodes
             * for which we have no address. */
            if (node->flags & (CLUSTER_NODE_MYSELF|CLUSTER_NODE_NOADDR)) continue;
    
            if (node->flags & CLUSTER_NODE_PFAIL)
                server.cluster->stats_pfail_nodes++;  // 如果某个instance是PFAIL状态, 增加计数  <==============
        }
    
    
    }


在任意一个clusterSendPing调用中会判断stats_pfail_nodes标志, 然后带上PFAIL信息

也就是说A向B发送的gossip中, 会把C的状态设置为PFAIL

.. code-block:: c

    /* Send a PING or PONG packet to the specified node, making sure to add enough
     * gossip information. */
    void clusterSendPing(clusterLink *link, int type) {
    
        int pfail_wanted = server.cluster->stats_pfail_nodes;  // 拿到计数  <=======================
        
        while(freshnodes > 0 && gossipcount < wanted && maxiterations--) {

            // 如果有instance为PFAIL状态, 那么我们要带上  <===================
            /* If there are PFAIL nodes, add them at the end. */
            if (pfail_wanted) {
                dictIterator *di;
                dictEntry *de;
        
                di = dictGetSafeIterator(server.cluster->nodes);
                while((de = dictNext(di)) != NULL && pfail_wanted > 0) {
                    clusterNode *node = dictGetVal(de);
                    if (node->flags & CLUSTER_NODE_HANDSHAKE) continue;
                    if (node->flags & CLUSTER_NODE_NOADDR) continue;
                    if (!(node->flags & CLUSTER_NODE_PFAIL)) continue;
                    clusterSetGossipEntry(hdr,gossipcount,node);  // 这里设置hdr中的flag为node中的flag, 所以把PFAIL状态给带上了  <===========================
                    freshnodes--;
                    gossipcount++;
                    /* We take the count of the slots we allocated, since the
                     * PFAIL stats may not match perfectly with the current number
                     * of PFAIL nodes. */
                    pfail_wanted--;
                }
                dictReleaseIterator(di);
            }
        
        }
    }

拷贝PFAIL状态到gossip
---------------------------

gossip是一个列表, 每个元素表示A对于某个node的状态

.. code-block:: c

    void clusterSetGossipEntry(clusterMsg *hdr, int i, clusterNode *n) {
        clusterMsgDataGossip *gossip;
        gossip = &(hdr->data.ping.gossip[i]);
        memcpy(gossip->nodename,n->name,CLUSTER_NAMELEN);
        gossip->ping_sent = htonl(n->ping_sent/1000);
        gossip->pong_received = htonl(n->pong_received/1000);
        memcpy(gossip->ip,n->ip,sizeof(n->ip));
        gossip->port = htons(n->port);
        gossip->cport = htons(n->cport);
        gossip->flags = htons(n->flags);
        gossip->notused1 = 0;
    }


PFAIL转为FAIL
--------------------

在B接收到A发送的gossip之后, 发现A看到C的状态为PFAIL, 那么B更新一下自己C的状态

这里重要的一旦是判断sender是否是master!!!!

.. code-block:: c

    void clusterProcessGossipSection(clusterMsg *hdr, clusterLink *link) {
    
        uint16_t count = ntohs(hdr->count);  // gossip列表的长度  <====================
        clusterMsgDataGossip *g = (clusterMsgDataGossip*) hdr->data.ping.gossip;
        clusterNode *sender = link->node ? link->node : clusterLookupNode(hdr->sender);
    
        while(count--) {
            uint16_t flags = ntohs(g->flags);
    
            /* Update our state accordingly to the gossip sections */
            node = clusterLookupNode(g->nodename); // 拿到instance  <===================
            if (node) {
                /* We already know this node.
                   Handle failure reports, only when the sender is a master. */
                // 如果sender是master, 我们采取行动!!! <====================================
                if (sender && nodeIsMaster(sender) && node != myself) {
                    // A说C是PFAIL状态  <=========================
                    if (flags & (CLUSTER_NODE_FAIL|CLUSTER_NODE_PFAIL)) {
                        if (clusterNodeAddFailureReport(node,sender)) {
                            serverLog(LL_VERBOSE,
                                "Node %.40s reported node %.40s as not reachable.",
                                sender->name, node->name);
                        }
                        //  更新一下自己对C的状态
                        markNodeAsFailingIfNeeded(node);
                    } else {
                        if (clusterNodeDelFailureReport(node,sender)) {
                            serverLog(LL_VERBOSE,
                                "Node %.40s reported node %.40s is back online.",
                                sender->name, node->name);
                        }
                    }
                }
            }
        }
    
    }

更新自己对目标instance的PFAIL/FAIL状态
------------------------------------------------

1. 如果B能连上C, 就不管

2. 如果如果B对C也是PFAIL, 那么查询本地保存的所有其他instance对C的状态, 如果超过一半的instance说他们对C也是PFAIL

   那么C就是FAIL

3. 调用clusterSendFail取广播C是FAIL状态这个消息, 广播不走ping/pong, 是直接广播, 加快掉线检测的速度

.. code-block:: c

    * 1) Either we reach the majority and eventually the FAIL state will propagate
    *    to all the cluster.
    * 2) Or there is no majority so no slave promotion will be authorized and the
    *    FAIL flag will be cleared after some time.
    void markNodeAsFailingIfNeeded(clusterNode *node) {
        int failures;
        int needed_quorum = (server.cluster->size / 2) + 1;

        if (!nodeTimedOut(node)) return; /* We can reach it. */ // 这里查询node->, 在clusterCron函数中更新!!!!  <=====================
        if (nodeFailed(node)) return; /* Already FAILing. */

        // 走到这里表示该节点是我们第一次不能访问, 但是还要根据大多数节点确认 <=======================
        failures = clusterNodeFailureReportsCount(node);  // 这里查询本地保存的其他node的gossip
        /* Also count myself as a voter if I'm a master. */
        if (nodeIsMaster(myself)) failures++;
        // 如果没有大多数节点说对方掉线了, 那么它还是在线的, 即使我们连接不到它  <======================
        if (failures < needed_quorum) return; /* No weak agreement from masters. */

        serverLog(LL_NOTICE,
            "Marking node %.40s as failing (quorum reached).", node->name); 

        // 走到这里表示我们既不能连接它同时大多数节点说也连接不到它, 那么它就是掉线了
        // 开始把对方标识为FAIL <=====================
        /* Mark the node as failing. */
        node->flags &= ~CLUSTER_NODE_PFAIL;
        node->flags |= CLUSTER_NODE_FAIL;
        node->fail_time = mstime();

        /* Broadcast the failing node name to everybody, forcing all the other
         * reachable nodes to flag the node as FAIL.
         * We do that even if this node is a replica and not a master: anyway
         * the failing state is triggered collecting failure reports from masters,
         * so here the replica is only helping propagating this status. */
        clusterSendFail(node->name);  // 广播, 不走ping  <===============================

    }


清除FAIL状态
===================

当某个我们能重新连接上某个node之后, 需要清除FAIL. 清除FAIL状态位的情况有3个

1. 如果node是slave, 那么直接清除

2. 如果node是master同时不负责任何slot, 直接清除

3. 如果是master同时其上次检测到掉线时间很久了, 其掉线时间大于cluster_node_timeout * CLUSTER_FAIL_UNDO_TIME_MULT

   同时该master中的slot配置还没有变化, 也就是没有其他node在其掉线之后接管其中的slot, 比如没有slave被选举为新的master


.. code-block:: c

    /* This function is called only if a node is marked as FAIL, but we are able
     * to reach it again. It checks if there are the conditions to undo the FAIL
     * state. */
    void clearNodeFailureIfNeeded(clusterNode *node) {
        mstime_t now = mstime();

        serverAssert(nodeFailed(node));

        /* For slaves we always clear the FAIL flag if we can contact the
         * node again. */
        // 如果该node是slave, 直接清除
        // 如果该node是master, 同时其不负责任何的slot, 那么也直接清除  <============================
        if (nodeIsSlave(node) || node->numslots == 0) {
            serverLog(LL_NOTICE,
                "Clear FAIL state for node %.40s: %s is reachable again.",
                    node->name,
                    nodeIsSlave(node) ? "replica" : "master without slots");
            node->flags &= ~CLUSTER_NODE_FAIL;  // 清除该node的FAIL状态位 <================================
            clusterDoBeforeSleep(CLUSTER_TODO_UPDATE_STATE|CLUSTER_TODO_SAVE_CONFIG);
        }

        /* If it is a master and...
         * 1) The FAIL state is old enough.
         * 2) It is yet serving slots from our point of view (not failed over).
         * Apparently no one is going to fix these slots, clear the FAIL flag. */
        // 这里是第3个情况  <=========================
        if (nodeIsMaster(node) && node->numslots > 0 &&
            (now - node->fail_time) >
            (server.cluster_node_timeout * CLUSTER_FAIL_UNDO_TIME_MULT))
        {
            serverLog(LL_NOTICE,
                "Clear FAIL state for node %.40s: is reachable again and nobody is serving its slots after some time.",
                    node->name);
            node->flags &= ~CLUSTER_NODE_FAIL;
            clusterDoBeforeSleep(CLUSTER_TODO_UPDATE_STATE|CLUSTER_TODO_SAVE_CONFIG);
        }
    }


epoch
================

集群的配置是和epoch相关的, epoch就是版本号(单调递增), 或者时钟, 一旦配置更新, 那么首先更新epoch.

**但每个instnace的epoch不一定要求都是相等的**, 但是更新配置的时候只能是最大的epoch来更新, 比较小的epoch的更新都被拒绝

这样就可能会出现不同的配置但是其epoch是一样的, 需要解决这个冲突

初始epoch
-----------------

在集群创建的时候, redis-cli为每个instance分配一个epoch, 第一个instance的config_epoch为1, 第二个instance的为2, 以此类推

.. code-block:: c

    static int clusterManagerCommandCreate(int argc, char **argv) {
            int config_epoch = 1;
            listRewind(cluster_manager.nodes, &li);
            // 在while中, 每次config_epoch都++
            while ((ln = listNext(&li)) != NULL) {
                clusterManagerNode *node = ln->value;
                redisReply *reply = NULL;
                reply = CLUSTER_MANAGER_COMMAND(node,
                                                "cluster set-config-epoch %d",
                                                config_epoch++);
                if (reply != NULL) freeReplyObject(reply);
            }
    
    }

instance收到set-config-epoch之后, 直接设置自己的config_epoch. 同时set-config-epoch只能在向一个新的instance发起

意味着该instance没有加入任何集群

.. code-block:: c

    void clusterCommand(client *c) {
        if (server.cluster_enabled == 0) {
            addReplyError(c,"This instance has cluster support disabled");
            return;
        }
        if(){
          // 其他命令
        } else if (!strcasecmp(c->argv[1]->ptr,"set-config-epoch") && c->argc == 3)
        {
            // 这里说明只能向一个fresh的instance发起set-config-epoch命令  <=======================
            // 一个fresh的instance是config_epoch为0, 不知道任何其他节点的(也就是没加入任何集群) <<=====================
            /* CLUSTER SET-CONFIG-EPOCH <epoch>
             *
             * The user is allowed to set the config epoch only when a node is
             * totally fresh: no config epoch, no other known node, and so forth.
             * This happens at cluster creation time to start with a cluster where
             * every node has a different node ID, without to rely on the conflicts
             * resolution system which is too slow when a big cluster is created. */
            long long epoch;
    
            if(){
              // 其他条件
            } else {
                // 设置自己的configEpoch  <==========================
                myself->configEpoch = epoch;
                serverLog(LL_WARNING,
                    "configEpoch set to %llu via CLUSTER SET-CONFIG-EPOCH",
                    (unsigned long long) myself->configEpoch);
    
                // 这里server.cluster->currentEpoch和myself->configEpoch相等
                // 都是设置的值  <=================================
                if (server.cluster->currentEpoch < (uint64_t)epoch)
                    server.cluster->currentEpoch = epoch;
                /* No need to fsync the config here since in the unlucky event
                 * of a failure to persist the config, the conflict resolution code
                 * will assign a unique config to this node. */
                clusterDoBeforeSleep(CLUSTER_TODO_UPDATE_STATE|
                                     CLUSTER_TODO_SAVE_CONFIG);
                addReply(c,shared.ok);
            }
        }
    }

在创建集群的时候, 发送set-config-epoch命令是在addslots指令之后的, 所以addslots, delslots都不会提升epoch

只有当调用setslots <slot> node <node id>的时候才会提升epoch, 使用setslots设置migrating和importing状态也都不会提升epoch

也就是只有slot迁移完成之后才会提升epoch

.. code-block:: c

    void clusterCommand(client *c) {
        if(){
          // 其他命令
        } else if ((!strcasecmp(c->argv[1]->ptr,"addslots") ||
                   !strcasecmp(c->argv[1]->ptr,"delslots")) && c->argc >= 3)
        {
            // 并不会提升epoch
            /* CLUSTER ADDSLOTS <slot> [slot] ... */
            /* CLUSTER DELSLOTS <slot> [slot] ... */
            // 很多代码被省略了
            zfree(slots);
            clusterDoBeforeSleep(CLUSTER_TODO_UPDATE_STATE|CLUSTER_TODO_SAVE_CONFIG);
            addReply(c,shared.ok);
    
        }else if (!strcasecmp(c->argv[1]->ptr,"setslot") && c->argc >= 4) {
            if(){
               // 其他条件, 比如migrating和importing
            } else if (!strcasecmp(c->argv[3]->ptr,"node") && c->argc == 5) {
                // slot已经迁移完成了!!!!!!!! <<====================
                /* CLUSTER SETSLOT <SLOT> NODE <NODE ID> */
    
                // 下面两句分别是操作server.cluster->slots
                // server.cluster->slots保存的是哪些slot由哪些master负责
                // server.cluster->slots = [master1, master1, master2, master2, ....]
                // 删除就是server.cluster->slots[i] = NULL
                // 添加就是server.cluster->slots[j] = master  <========================
                clusterDelSlot(slot);
                clusterAddSlot(n,slot);
    
                /* If this node was importing this slot, assigning the slot to
                 * itself also clears the importing status. */
                if (n == myself &&
                    server.cluster->importing_slots_from[slot])
                {
                    /* This slot was manually migrated, set this node configEpoch
                     * to a new epoch so that the new version can be propagated
                     * by the cluster.
                     *
                     * Note that if this ever results in a collision with another
                     * node getting the same configEpoch, for example because a
                     * failover happens at the same time we close the slot, the
                     * configEpoch collision resolution will fix it assigning
                     * a different epoch to each node. */
                    // 这里提升自己的epoch
                    // 如果提升?  <=================================
                    if (clusterBumpConfigEpochWithoutConsensus() == C_OK) {
                        serverLog(LL_WARNING,
                            "configEpoch updated after importing slot %d", slot);
                    }
                    server.cluster->importing_slots_from[slot] = NULL;
                    /* After importing this slot, let the other nodes know as
                     * soon as possible. */
                    clusterBroadcastPong(CLUSTER_BROADCAST_ALL);
                }
            }
        }
    }

通过cluster info能打印出自己的epoch和整个集群中的(自己看到的)epoch

.. code-block::

    allenling@allenling-ubuntu:~/redis-cluster-test$ redis-cli -c -p 7000
    127.0.0.1:7000> cluster info
    cluster_current_epoch:6
    cluster_my_epoch:1 // 这是我的epoch <==================
    127.0.0.1:7000>
    allenling@allenling-ubuntu:~/redis-cluster-test$ redis-cli -c -p 7001
    127.0.0.1:7001> CLUSTER info
    cluster_current_epoch:6
    cluster_my_epoch:2  // 我的epoch <====================
    127.0.0.1:7001>


**但是我们通过cluster info发现, slave和master的cluster_my_epoch是一样的!**

.. code-block::

    allenling@allenling-ubuntu:~/redis-cluster-test$ redis-cli -c -p 7004
    127.0.0.1:7004> CLUSTER info
    cluster_current_epoch:6
    cluster_my_epoch:1  // 这里
    127.0.0.1:7004>

这是因为slave显示的是master的信息, 但是其实slave内部的configEpoch不一样是和master相等的

.. code-block:: c

    myepoch = (nodeIsSlave(myself) && myself->slaveof) ?
              myself->slaveof->configEpoch : myself->configEpoch; // 拿到的是master的configEpoch <========
    sds info = sdscatprintf(sdsempty(),
        "cluster_state:%s\r\n"
        "cluster_slots_assigned:%d\r\n"
        "cluster_slots_ok:%d\r\n"
        "cluster_slots_pfail:%d\r\n"
        "cluster_slots_fail:%d\r\n"
        "cluster_known_nodes:%lu\r\n"
        "cluster_size:%d\r\n"
        "cluster_current_epoch:%llu\r\n"
        "cluster_my_epoch:%llu\r\n"  // 显示的是master的configEpoch <================
        , statestr[server.cluster->state],
        slots_assigned,
        slots_ok,
        slots_pfail,
        slots_fail,
        dictSize(server.cluster->nodes),
        server.cluster->size,
        (unsigned long long) server.cluster->currentEpoch,
        (unsigned long long) myepoch
    );


但是如果slave要进行选举的话, 使用的还是myself->configEpoch而不是master的configEpoch(myself->slaveof->configEpoch)


configEpoch和currentEpoch
------------------------------

configEpoch存储是节点信息上(node->configEpoch), 而currentEpoch是存储在集群信息中的(slave.cluster->currentEpoch)

所以configEpoch是每个node自己都有一份, 每个node都不一样, 同时每个node都向其他node广播自己的configEpoch(slave的话广播的是master的)

每个node本地都保存了其他node的configEpoch. 比如n1, n2, n3的configEpoch分别是1， 2， 3，那么n1, n2, n3本地都存了一份

.. code-block:: c

    {n1: {"configEpoch": 1}, n2: {"configEpoch": 2}, n3: {"configEpoch": 3}}

而currentEpoch是集群中同步的版本号, 最大的currentEpoch表示最新的配置, 所有较小的currentEpoch都会被更新, 和currentEpoch同步

虽然redis cluster spec和网上很多文章都提到currentEpoch是使用在slave选举, 而configEpoch是用在slot的变更上

但是从代码上, 除了slave发送选举的auth request是单独提升currentEpoch, 就算是slot迁移, configEpoch依然会根据currentEpoch一起变更

同时就算slave发送选举auth request是单独提升currentEpoch, 但是如果slave赢得选举的话, configEpoch最终还是会提升到currentEpoch

**所以任何比较configEpoch其实最终都是比较了currentEpoch, 同时解决configEpoch的时候, 也同步了configEpoch和currentEpoch, 所以这两者看起来就是一个东西**


1. 初始化的时候, myself->configEpoch和server.cluster->currentEpoch都设置为0

2. 然后cli发送set-config-epoch命令, 两者都设置为传入的值

   .. code-block:: c

       if (!strcasecmp(c->argv[1]->ptr,"set-config-epoch") && c->argc == 3){
           myself->configEpoch = epoch;
           serverLog(LL_WARNING,
               "configEpoch set to %llu via CLUSTER SET-CONFIG-EPOCH",
               (unsigned long long) myself->configEpoch);

           if (server.cluster->currentEpoch < (uint64_t)epoch)
               server.cluster->currentEpoch = epoch;
       }


    所以此时对于6个节点, 我们分别有
                                  n1 n2 n3 n4 n5 n6
    server.cluster->currentEpoch: 1  2  3  4  5  6
    myself->configEpoch         : 1  2  3  4  5  6

3. 在cli发送meet之后, 大家在ping中会带上自己的currentEpoch和configEpoch


   .. code-block:: c

   void clusterSendPing(clusterLink *link, int type) {
        // 构建msg
       clusterBuildMessageHdr(hdr,type);
   }

   void clusterBuildMessageHdr(clusterMsg *hdr, int type) {
       // 总是带上这两个信息
       hdr->currentEpoch = htonu64(server.cluster->currentEpoch);
       hdr->configEpoch = htonu64(master->configEpoch);
   }


4. 在instance收到任何packet(不仅仅是ping), 将会同步这两个信息

   .. code-block:: c

       int clusterProcessPacket(clusterLink *link) {
           if (sender && !nodeInHandshake(sender)) {
               /* Update our currentEpoch if we see a newer epoch in the cluster. */
               // 这里要同步了!!!!!!!!!!!!!!!!!!!
               senderCurrentEpoch = ntohu64(hdr->currentEpoch);
               senderConfigEpoch = ntohu64(hdr->configEpoch);
               // 同步currentEpoch  <=====================
               if (senderCurrentEpoch > server.cluster->currentEpoch)
                   server.cluster->currentEpoch = senderCurrentEpoch;
               /* Update the sender configEpoch if it is publishing a newer one. */
               // 同步每一个node的configEpoch <======================
               if (senderConfigEpoch > sender->configEpoch) {
                   sender->configEpoch = senderConfigEpoch;
                   clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|
                                        CLUSTER_TODO_FSYNC_CONFIG);
               }
           }
       }

    所以经过消息同步, 每个节点都保存了这样的消息

    server.cluster->currentEpoch = 6

    nodes: {"node1_ID": {"configEpoch": 1},
            "node2_ID": {"configEpoch": 2},
            "node3_ID": {"configEpoch": 3},
            "node4_ID": {"configEpoch": 4},
            "node5_ID": {"configEpoch": 5},
            "node6_ID": {"configEpoch": 6},
            }

提升epoch
----------------

除了主动发起提升epoch的命令(bumpepoch)之外, 提升epoch有3种情况

1. slave选举的时候直接提升server.cluster->currentEpoch

   .. code-block:: c

       void clusterHandleSlaveFailover(void) {
           /* Ask for votes if needed. */
           if (server.cluster->failover_auth_sent == 0) {
               server.cluster->currentEpoch++; // currentEpoch自增一下 <==================
               // 记录下发起auth request的currentEpoch   <=========================
               server.cluster->failover_auth_epoch = server.cluster->currentEpoch;
               serverLog(LL_WARNING,"Starting a failover election for epoch %llu.",
                   (unsigned long long) server.cluster->currentEpoch);
               clusterRequestFailoverAuth();
               server.cluster->failover_auth_sent = 1;
               clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|
                                    CLUSTER_TODO_UPDATE_STATE|
                                    CLUSTER_TODO_FSYNC_CONFIG);
               return; /* Wait for replies. */
           }

           /* Check if we reached the quorum. */
           if (server.cluster->failover_auth_count >= needed_quorum) {
               /* We have the quorum, we can finally failover the master. */

               serverLog(LL_WARNING,
                   "Failover election won: I'm the new master.");

               // 这里, 如果我们赢得了选举, configEpoch会和currentEpoch同步的!!!!! <<=================
               /* Update my configEpoch to the epoch of the election. */
               if (myself->configEpoch < server.cluster->failover_auth_epoch) {
                   myself->configEpoch = server.cluster->failover_auth_epoch;
                   serverLog(LL_WARNING,
                       "configEpoch set to %llu after successful failover",
                       (unsigned long long) myself->configEpoch);
               }

               // 所以这里发送的ping/pong, 我们的configEpoch和currentEpoch是相等的
               // 且都是最大的!!! <<==========================
               /* Take responsibility for the cluster slots. */
               clusterFailoverReplaceYourMaster();
           }
       }

2. 收到主动failover命令, 进行takover操作, 此时可能会同时提升configEpoch和currentEpoch

   .. code-block:: c

       void clusterCommand(client *c) {
           if(){
               // 其他命令
           }else if (!strcasecmp(c->argv[1]->ptr,"failover") && (c->argc == 2 || c->argc == 3))
           {
               if (takeover) {
                   /* A takeover does not perform any initial check. It just
                    * generates a new configuration epoch for this node without
                    * consensus, claims the master's slots, and broadcast the new
                    * configuration. */
                   serverLog(LL_WARNING,"Taking over the master (user request).");
                   // 这里提升server.cluster->currentEpoch和configEpoch!!!! <==========================
                   clusterBumpConfigEpochWithoutConsensus();
                   clusterFailoverReplaceYourMaster(); // 发送failover消息给其他instance <==============
               } 
           }
       }

3. 迁移slot完成之后, 此时可能会同时提升configEpoch和currentEpoch

.. code-block:: c

    void clusterCommand(client *c) {
        if(){
            // 其他命令
        } else if (!strcasecmp(c->argv[3]->ptr,"node") && c->argc == 5) {
            /* CLUSTER SETSLOT <SLOT> NODE <NODE ID> */

            clusterDelSlot(slot);
            clusterAddSlot(n,slot);

            /* If this node was importing this slot, assigning the slot to
             * itself also clears the importing status. */
            if (n == myself &&
                server.cluster->importing_slots_from[slot])
            {
                /* This slot was manually migrated, set this node configEpoch
                 * to a new epoch so that the new version can be propagated
                 * by the cluster.
                 *
                 * Note that if this ever results in a collision with another
                 * node getting the same configEpoch, for example because a
                 * failover happens at the same time we close the slot, the
                 * configEpoch collision resolution will fix it assigning
                 * a different epoch to each node. */
                // 这里提升server.cluster->currentEpoch和myelf->configEpoch !!!!!!!!!!!!!!!! <=======================
                if (clusterBumpConfigEpochWithoutConsensus() == C_OK) {
                    serverLog(LL_WARNING,
                        "configEpoch updated after importing slot %d", slot);
                }
                server.cluster->importing_slots_from[slot] = NULL;
                /* After importing this slot, let the other nodes know as
                 * soon as possible. */
                clusterBroadcastPong(CLUSTER_BROADCAST_ALL); // 广播配置改变的  <==========================
            }
        }

    }

4. 函数clusterBumpConfigEpochWithoutConsensus, 总是会同步configEpoch和currentEpoch

   比如n1, n2, n3的configEpoch分别是1, 2, 3, currentEpoch都是3

   第一次n1调用该函数, 那么显然n1的configEpoch和currentEpoch会变为4, 然后n2, n3的currentEpoch都变为4

   同时记录下n1的configEpoch为4

   第二次n2调用该函数, 那么n2的configEpoch和currentEpoch都会变为5, 然后以此类推

   .. code-block:: c

        int clusterBumpConfigEpochWithoutConsensus(void) {
            // 各个instance中的configEpoch最大值 <======================
            uint64_t maxEpoch = clusterGetMaxEpoch();

            if (myself->configEpoch == 0 ||
             myself->configEpoch != maxEpoch)
            {
             // 如果我们当前的configEpoch不是最大值, 那么提升currentEpoch <==============
             server.cluster->currentEpoch++;
             // 同时把myself-configEpoch也设置为最新的currentEpoch  <=======================
             myself->configEpoch = server.cluster->currentEpoch; 
             clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|
                                  CLUSTER_TODO_FSYNC_CONFIG);
             serverLog(LL_WARNING,
                 "New configEpoch set to %llu",
                 (unsigned long long) myself->configEpoch);
             return C_OK;
            } else {
             return C_ERR;
            }
        }

5. clusterGetMaxEpoch

   .. code-block:: c

       uint64_t clusterGetMaxEpoch(void) {
           uint64_t max = 0;
           dictIterator *di;
           dictEntry *de;
       
           di = dictGetSafeIterator(server.cluster->nodes);
           // 拿到最大的configEpoch  <===============================
           while((de = dictNext(di)) != NULL) {
               clusterNode *node = dictGetVal(de);
               if (node->configEpoch > max) max = node->configEpoch;
           }
           dictReleaseIterator(di);
           // 取max和currentEpoch的最大值 <============================
           if (max < server.cluster->currentEpoch) max = server.cluster->currentEpoch;
           return max;
       }



configEpoch冲突
------------------

根据上面的代码, 我们可以看到configEpoch总是在本地自增, 根据redis cluster spec, 迁移slots如果需要全部分master同意的话, 是很低效的

*Requiring an agreement to generate new configuration epochs during resharding, for each hash slot moved, is inefficient.*

在slave选举的时候, 即使两个slave同时发起选举, 同时增加了currentEpoch, 但是由于master中对于auth request

进行了时间和次数上的约束, 所以configEpoch其实没有派上用场. 同时由于currentEpoch是最大的才能被处理, 所以currentEpoch没有冲突

这个说法(即使currentEpoch冲突, 但是选举的时候如果之前已经向当前currentEpoch投过票了, 那么不会再投票的).

而configEpoch有两个地方会提升, 分别是failover指令和slots迁移完成. 这个两个事件总是本地增加 configEpoch (虽然此时configEpoch会和currentEpoch同步)

所以configEpoch会可能出现冲突. 根据redis cluster spec中提到的可能情况:

*However because of the two cases above, it is possible (though unlikely) to end with multiple nodes having the same
configuration epoch. A resharding operation performed by the system administrator,
and a failover happening at the same time (plus a lot of bad luck) could cause currentEpoch collisions
if they are not propagated fast enough.*

假设m1, m2, m3, 已经m1的slave ms1, 当你在迁移m1的slot到m2的时候, ms1此时收到failover指令, 所以最终m2和ms1中的

configEpoch对阵m1的slot的configEpoch是相等的(当然currentEpoch也是相等的)

当然configEpoch相等也可能是软件, 系统的bug导致的

*Moreover, software bugs and filesystem corruptions can also contribute to multiple nodes having the same configuration epoch.*

所以要解决configEpoch冲突. redis中处理这个冲突很简单, **冲突的两个node, 名字的字典序比较小的configEpoch(包括currentEpoch)自增1, 另外一个保持**


.. code-block:: c


    int clusterProcessPacket(clusterLink *link) {

        /* PING, PONG, MEET: process config information. */
        if (type == CLUSTERMSG_TYPE_PING || type == CLUSTERMSG_TYPE_PONG ||
            type == CLUSTERMSG_TYPE_MEET)
        {
            /* If our config epoch collides with the sender's try to fix
             * the problem. */
            // 如果发现configEpoch冲突!!!!! <<====================
            if (sender &&
                nodeIsMaster(myself) && nodeIsMaster(sender) &&
                senderConfigEpoch == myself->configEpoch)
            {
                clusterHandleConfigEpochCollision(sender);
            }

        }

    }

     /*
     * When this function gets called, what happens is that if this node
     * has the lexicographically smaller Node ID compared to the other node
     * with the conflicting epoch (the 'sender' node), it will assign itself
     * the greatest configuration epoch currently detected among nodes plus 1.
     *
     * This means that even if there are multiple nodes colliding, the node
     * with the greatest Node ID never moves forward, so eventually all the nodes
     * end with a different configuration epoch.
     */
    void clusterHandleConfigEpochCollision(clusterNode *sender) {
        /* Prerequisites: nodes have the same configEpoch and are both masters. */
        if (sender->configEpoch != myself->configEpoch ||
            !nodeIsMaster(sender) || !nodeIsMaster(myself)) return;
        /* Don't act if the colliding node has a smaller Node ID. */
        // sender的名字的字典序小于我们, 那么sender可以继续, 我们就不继续了!!!!!!!  <<=========================
        if (memcmp(sender->name,myself->name,CLUSTER_NAMELEN) <= 0) return;
        // 走到这里说明我们的名字的字典序比较小, 自增一下!!!!  <<===========================
        /* Get the next ID available at the best of this node knowledge. */
        server.cluster->currentEpoch++;
        myself->configEpoch = server.cluster->currentEpoch;
        clusterSaveConfigOrDie(1);
        serverLog(LL_VERBOSE,
            "WARNING: configEpoch collision with node %.40s."
            " configEpoch set to %llu",
            sender->name,
            (unsigned long long) myself->configEpoch);
    }



slave选举
===============


slave rank
----------------

https://tkstone.blog/2019/01/16/redis-replication-gap-tuning/

slave之间会有一个rank, 0最大, 这个rank是根据slave和master之间同步的offset来排序的

.. code-block::

    allenling@allenling-ubuntu:~/redis-cluster-test$ redis-cli -c -p 7000
    127.0.0.1:7000> info replication
    slave_repl_offset:84
    master_repl_offset:84

可以看到master_repl_offset和slave_repl_offset这个两个数据表示master和slave之间的"数据差", 可以简单看成master有84"个"数据(master_repl_offset指标)

但是有可能slave_repl_offset比84小, 这是很可能的, 因为master和slave之间的同步是异步执行的.

如果slave_repl_offset比master_repl_offset小, 证明该slave有可能因为网络等问题, 和master的数据没完全同步上.

根据上面的参考文章, 没同步上要么是master发送丢失, 要么是slave的ack丢失. 同时ack是定时执行的, 所以减少这个ack周期也有助于减少

master_repl_offset和slave_repl_offset之间的差值

正常的情况下, 每个slave的repl_offset都和master的repl_offset一样大

获取当前slave的rank, 如果没有slave的offset比我们的大, 我们就是rank0

.. code-block:: c

    int clusterGetSlaveRank(void) {
        long long myoffset;
        int j, rank = 0;  // rank默认为0  <======================
        clusterNode *master;

        serverAssert(nodeIsSlave(myself));
        master = myself->slaveof;
        if (master == NULL) return 0; /* Never called by slaves without master. */

        myoffset = replicationGetSlaveOffset();
        // 遍历所有的slaves, 找出有多少个slave的offset比我们的大  <================
        for (j = 0; j < master->numslaves; j++)
            if (master->slaves[j] != myself &&
                !nodeCantFailover(master->slaves[j]) &&
                master->slaves[j]->repl_offset > myoffset) rank++;
        return rank;  // 没有人比我们的offset更大, 那么我们就是rank0
    }


选举条件
---------------


一个slave如果要发起选举, 那么需要满足3个条件

1. master是FAIL状态

2. master负责某些slot

3. slave和master之间断线的时间不超过某个时间, 这个时间是用户配置的. 这是为了保证slave和master断线不要太久

4. 我们是rank0


选举的操作是在函数clusterHandleSlaveFailover中

.. code-block:: c

    void clusterHandleSlaveFailover(void) {

        // 这几个条件就是上面提到的  <====================================
        /* Pre conditions to run the function, that must be met both in case
         * of an automatic or manual failover:
         * 1) We are a slave.
         * 2) Our master is flagged as FAIL, or this is a manual failover.
         * 3) We don't have the no failover configuration set, and this is
         *    not a manual failover.
         * 4) It is serving slots. */
        if (nodeIsMaster(myself) ||
            myself->slaveof == NULL ||
            (!nodeFailed(myself->slaveof) && !manual_failover) ||
            (server.cluster_slave_no_failover && !manual_failover) ||
            myself->slaveof->numslots == 0)
        {
            /* There are no reasons to failover, so we set the reason why we
             * are returning without failing over to NONE. */
            server.cluster->cant_failover_reason = CLUSTER_CANT_FAILOVER_NONE;
            return;
        }

    }


该函数在clusterCron中每次执行都会调用, 意味着每秒至少(其他地方也会调用)有10次取检查是否有掉线.


.. code-block:: c

    /* This is executed 10 times every second */
    void clusterCron(void) {
    
        if (nodeIsSlave(myself)) {
            clusterHandleManualFailover();
            // 注意这里的判断条件
            // 只有设置了CLUSTER_MODULE_FLAG_NO_FAILOVER才不会去执行选举
            // 但是这个标志只有手动设置, 代码中不会设置
            // 所以正常情况下这个条件总是!0  <=============================================
            if (!(server.cluster_module_flags & CLUSTER_MODULE_FLAG_NO_FAILOVER))
                clusterHandleSlaveFailover();
        }
    
    }





选举流程
-------------

0. 设置一个delay时间, 表示只有到该时间的时候才会去执行选举

   这个delay时间也把多个slave的选举时间给错开了, 这样同一个时间中只有一个slave会发起选举, 同时同一时间master只会回复一个slave(这个也可以用epoch来保证)

   比如所有的slave的offset都一样, 那么就需要错开选举时间, 任意一个都可以选举

   同时这个delay也给我们slave时间去交流各自的rank, 那么最大的rank的slave才会去发起选举, 这样rank0的slave总是能赢得选举

1. 通过PONG去广播自己的rank以及收集其他slave的rank

2. 如果自己的rank是最大的, 可以发起请求, 否则退出

3. 发起选举的时候, 增加epoch, 向所有的节点(虽然文档是说所有的master节点)发起CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST消息表示需要选举

4. 如果收到超过半数的master返回FAILOVER_AUTH_ACK, 那么说明我们赢得了选举, 我们就是新的master

   每次收到FAILOVER_AUTH_ACK, 都会调用clusterHandleSlaveFailover去校验我们是否达到了超过半数的节点返回ack

   等到FAILOVER_AUTH_ACK的超时是server.cluster_node_timeout*2(如果乘以2只会小于2秒, 那么至少2秒)

   同时超过只会会等待server.cluster_node_timeout*4(这里同样, 乘以4只会如果小于4秒, 取4秒), 再次去发起新的选举


.. code-block:: c

    void clusterHandleSlaveFailover(void) {
    
        // 我们之前发送auth request的时间就是failover_auth_time
        // 那么auth_age就是我们等待ack等待了多久  <==============================
        mstime_t auth_age = mstime() - server.cluster->failover_auth_time;
        int needed_quorum = (server.cluster->size / 2) + 1;  // 注意这里是大多数master而不是大多数节点
    
        // 这里auth_timeout就是等待FAILOVER_AUTH_ACK的超时
        // auth_retry_time就是重新发起选举的间隔  <===========================
        auth_timeout = server.cluster_node_timeout*2;
        if (auth_timeout < 2000) auth_timeout = 2000;
        auth_retry_time = auth_timeout*2;
    
    
        if (auth_age > auth_retry_time) {
           // 等了至少4s了, 重新发送auth request吧
           // 再次设置延迟时间!!!!!! <=======================
            server.cluster->failover_auth_time = mstime() +
                500 + /* Fixed delay of 500 milliseconds, let FAIL msg propagate. */
                random() % 500; /* Random delay between 0 and 500 milliseconds. */
            server.cluster->failover_auth_count = 0;
            // 设置failover_auth_sent为0表示需要发送auth request!!! <=======================
            server.cluster->failover_auth_sent = 0;
            server.cluster->failover_auth_rank = clusterGetSlaveRank();
            /* We add another delay that is proportional to the slave rank.
             * Specifically 1 second * rank. This way slaves that have a probably
             * less updated replication offset, are penalized. */
            server.cluster->failover_auth_time +=
                server.cluster->failover_auth_rank * 1000;
        }
    
        if (auth_age > auth_timeout) {
            // 如果我们等待ack超过了2s了, 那么记录一下, 退出
            // 下次进来的时候, 就走重新发送auth request逻辑  <======================
            clusterLogCantFailover(CLUSTER_CANT_FAILOVER_EXPIRED);
            return;
        }
    
    
        // 这里我们没有发送auth request, 并且
        // 不是手动failover的话, 我们需要检查一下我们是否收到
        // 了更大的rank, 也就是别人的rank更大
        // 那么就等待更久, 为什么?
        // 因为比我们大的rank可能也会掉线, 所以我们需要随时准备好了 <=============================
        /* It is possible that we received more updated offsets from other
         * slaves for the same master since we computed our election delay.
         * Update the delay if our rank changed.
         *
         * Not performed if this is a manual failover. */
        if (server.cluster->failover_auth_sent == 0 &&
            server.cluster->mf_end == 0)
        {
            int newrank = clusterGetSlaveRank();
            if (newrank > server.cluster->failover_auth_rank) {
                long long added_delay =
                    (newrank - server.cluster->failover_auth_rank) * 1000;
                server.cluster->failover_auth_time += added_delay;
                server.cluster->failover_auth_rank = newrank;
                serverLog(LL_WARNING,
                    "Replica rank updated to #%d, added %lld milliseconds of delay.",
                    newrank, added_delay);
            }
        }
    
        // 经过那么判断只会, 那么我们需要发送auth request了 <======================
        /* Ask for votes if needed. */
        if (server.cluster->failover_auth_sent == 0) {
            server.cluster->currentEpoch++;
            server.cluster->failover_auth_epoch = server.cluster->currentEpoch;
            serverLog(LL_WARNING,"Starting a failover election for epoch %llu.",
                (unsigned long long) server.cluster->currentEpoch);
            clusterRequestFailoverAuth();
            // 设置failover_auth_sent标志位为1
            server.cluster->failover_auth_sent = 1;
            clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|
                                 CLUSTER_TODO_UPDATE_STATE|
                                 CLUSTER_TODO_FSYNC_CONFIG);
            return; /* Wait for replies. */
        }

        // 最后最后!!!!! <=======================
        // 这里说明我们收到了auth ack, 所以我们需要判断一下是否达到了大多数master <=========================
        /* Check if we reached the quorum. */
        if (server.cluster->failover_auth_count >= needed_quorum) {
            /* We have the quorum, we can finally failover the master. */

            serverLog(LL_WARNING,
                "Failover election won: I'm the new master.");

            /* Update my configEpoch to the epoch of the election. */
            if (myself->configEpoch < server.cluster->failover_auth_epoch) {
                myself->configEpoch = server.cluster->failover_auth_epoch;
                serverLog(LL_WARNING,
                    "configEpoch set to %llu after successful failover",
                    (unsigned long long) myself->configEpoch);
            }

            /* Take responsibility for the cluster slots. */
            clusterFailoverReplaceYourMaster(); // 广播我是master这个消息  <========================
        } else {
            clusterLogCantFailover(CLUSTER_CANT_FAILOVER_WAITING_VOTES);
        }
    }

slave广播自己提升为master
-------------------------------------

.. code-block:: c

    void clusterFailoverReplaceYourMaster(void) {
        int j;
        clusterNode *oldmaster = myself->slaveof;
    
        if (nodeIsMaster(myself) || oldmaster == NULL) return;
    
        /* 1) Turn this node into a master. */
        clusterSetNodeAsMaster(myself);
        replicationUnsetMaster();
    
        /* 2) Claim all the slots assigned to our master. */
        // 这里把所有oldmaster负责的slot转移到自己来 <===========================
        for (j = 0; j < CLUSTER_SLOTS; j++) {
            if (clusterNodeGetSlotBit(oldmaster,j)) {
                clusterDelSlot(j);
                clusterAddSlot(myself,j);
            }
        }
    
        /* 3) Update state and save config. */
        clusterUpdateState();
        clusterSaveConfigOrDie(1);
    
        /* 4) Pong all the other nodes so that they can update the state
         *    accordingly and detect that we switched to master role. */
        clusterBroadcastPong(CLUSTER_BROADCAST_ALL); // 广播PONG信息给所有的节点 <==========================
    
        /* 5) If there was a manual failover in progress, clear the state. */
        resetManualFailover();
    }


更新两个slots结构
------------------

一个slot是cluster_state中的, 一个数组, 每个元素存储的是clusterNode指针

clusterDelSlot, clusterAddSlot更新的是这个slots数组


.. code-block:: c

    typedef struct clusterState {
        clusterNode *slots[CLUSTER_SLOTS];
    
    }

但是我们发送的pong不太可能带上这个数组, 因为这个数组太大了, 所以我们发送的另外一个slots

这个slot是保存在clusterNode中的, 是一个bitmap, 每个bit表示一个slot, 那么一个字节是8个bit, 所以这个数组长度就是CLUSTER_SLOTS/8

.. code-block:: c

    typedef struct clusterNode {
        unsigned char slots[CLUSTER_SLOTS/8]; /* slots handled by this node */
    }

所以在clusterFailoverReplaceYourMaster中调用的clusterAddSlot和clusterDelSlot分别去更新了上述两个slot结构

**同时把node中的numslots个数分别加1和减1**

而在调用clusterSendPing中就带上了第二个slots结构, 发送给其他的instance

.. code-block:: c

    void clusterSendPing(clusterLink *link, int type) {
    
        clusterBuildMessageHdr(hdr,type);
    
    }
    
    void clusterBuildMessageHdr(clusterMsg *hdr, int type) {
    
        clusterNode *master;
    
        // 如果我们不是master, 那么我们发送的其实是master的信息
        master = (nodeIsSlave(myself) && myself->slaveof) ?
                  myself->slaveof : myself;
    
    
        // 拷贝clusterNode中的slots信息
        memcpy(hdr->myslots,master->slots,sizeof(hdr->myslots));
    
    }




master发送auth ack
--------------------------


.. code-block:: c

    /* Vote for the node asking for our vote if there are the conditions. */
    // 这里node是gossip的sender, 也就是slave
    void clusterSendFailoverAuthIfNeeded(clusterNode *node, clusterMsg *request) {
        clusterNode *master = node->slaveof;  // 拿到master  <========================
        uint64_t requestCurrentEpoch = ntohu64(request->currentEpoch);
        uint64_t requestConfigEpoch = ntohu64(request->configEpoch);
        unsigned char *claimed_slots = request->myslots;
        int force_ack = request->mflags[0] & CLUSTERMSG_FLAG0_FORCEACK;
        int j;
    
        /* IF we are not a master serving at least 1 slot, we don't have the
         * right to vote, as the cluster size in Redis Cluster is the number
         * of masters serving at least one slot, and quorum is the cluster
         * size + 1 */
        // 如果我不是master或者我不负责任何slot, 不管  <===========================
        if (nodeIsSlave(myself) || myself->numslots == 0) return;
    
        /* Request epoch must be >= our currentEpoch.
         * Note that it is impossible for it to actually be greater since
         * our currentEpoch was updated as a side effect of receiving this
         * request, if the request epoch was greater. */
        // 判断request的epoch和自己的epoch
        // 如果前者小于后者, 拒绝  <===========================
        if (requestCurrentEpoch < server.cluster->currentEpoch) {
            serverLog(LL_WARNING,
                "Failover auth denied to %.40s: reqEpoch (%llu) < curEpoch(%llu)",
                node->name,
                (unsigned long long) requestCurrentEpoch,
                (unsigned long long) server.cluster->currentEpoch);
            return;
        }
    
        // lastVoteEpoch表示我们最近投票的epoch
        // 如果lastVoteEpoch和我们当前看到的currentEpoch相等, 相当于我们已经投过票了并且配置已经改变过了
        // 不能再投票了, 这里可能是某个slave的auth request因为网络问题(比如网络分区)很久才到达
        // 到达的时候另外一个slave已经发起并完成了另一轮auth request  <======================
        /* I already voted for this epoch? Return ASAP. */
        if (server.cluster->lastVoteEpoch == server.cluster->currentEpoch) {
            serverLog(LL_WARNING,
                    "Failover auth denied to %.40s: already voted for epoch %llu",
                    node->name,
                    (unsigned long long) server.cluster->currentEpoch);
            return;
        }
    
        // 判断该slave的master是否是FAIL状态 <===================
        /* Node must be a slave and its master down.
         * The master can be non failing if the request is flagged
         * with CLUSTERMSG_FLAG0_FORCEACK (manual failover). */
        if (nodeIsMaster(node) || master == NULL ||
            (!nodeFailed(master) && !force_ack))
        {
            if (nodeIsMaster(node)) {
                serverLog(LL_WARNING,
                        "Failover auth denied to %.40s: it is a master node",
                        node->name);
            } else if (master == NULL) {
                serverLog(LL_WARNING,
                        "Failover auth denied to %.40s: I don't know its master",
                        node->name);
            } else if (!nodeFailed(master)) {
                serverLog(LL_WARNING,
                        "Failover auth denied to %.40s: its master is up",
                        node->name);
            }
            return;
        }
    
        // 如果我们之前已经为这个master的选举通过票了
        // 即使这个request的epoch更大
        // 我们也不能在cluster_node_timeout * 2的时间内对同一个master的选举投两次票  <=========================
        /* We did not voted for a slave about this master for two
         * times the node timeout. This is not strictly needed for correctness
         * of the algorithm but makes the base case more linear. */
        if (mstime() - node->slaveof->voted_time < server.cluster_node_timeout * 2)
        {
            serverLog(LL_WARNING,
                    "Failover auth denied to %.40s: "
                    "can't vote about this master before %lld milliseconds",
                    node->name,
                    (long long) ((server.cluster_node_timeout*2)-
                                 (mstime() - node->slaveof->voted_time)));
            return;
        }

        // slave发送的configEpoch小于master的configEpoch, 也不会发送auth ok <======================
        /* The slave requesting the vote must have a configEpoch for the claimed
         * slots that is >= the one of the masters currently serving the same
         * slots in the current configuration. */
        for (j = 0; j < CLUSTER_SLOTS; j++) {
            if (bitmapTestBit(claimed_slots, j) == 0) continue;
            if (server.cluster->slots[j] == NULL ||
                server.cluster->slots[j]->configEpoch <= requestConfigEpoch)
            {
                continue;
            }
            /* If we reached this point we found a slot that in our current slots
             * is served by a master with a greater configEpoch than the one claimed
             * by the slave requesting our vote. Refuse to vote for this slave. */
            serverLog(LL_WARNING,
                    "Failover auth denied to %.40s: "
                    "slot %d epoch (%llu) > reqEpoch (%llu)",
                    node->name, j,
                    (unsigned long long) server.cluster->slots[j]->configEpoch,
                    (unsigned long long) requestConfigEpoch);
            return;
        }
    
        // 否则发送auth ok, 然后记录下时间
        /* We can vote for this slave. */
        server.cluster->lastVoteEpoch = server.cluster->currentEpoch;
        node->slaveof->voted_time = mstime();
        clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|CLUSTER_TODO_FSYNC_CONFIG);
        clusterSendFailoverAuth(node);
        serverLog(LL_WARNING, "Failover auth granted to %.40s for epoch %llu",
            node->name, (unsigned long long) server.cluster->currentEpoch);
    }


master得知其他slave被提升为master了
---------------------------------------

修改本地的信息表

.. code-block:: c

    int clusterProcessPacket(clusterLink *link) {
    
        /* PING, PONG, MEET: process config information. */
        if (type == CLUSTERMSG_TYPE_PING || type == CLUSTERMSG_TYPE_PONG ||
            type == CLUSTERMSG_TYPE_MEET)
        {

            /* Check for role switch: slave -> master or master -> slave. */
            if (sender) {
                // 如果该instance的slaveof为空, 那么这个instance就是master
                if (!memcmp(hdr->slaveof,CLUSTER_NODE_NULL_NAME,
                    sizeof(hdr->slaveof)))
                {
                    /* Node is a master. */
                    // 我们得知该instance是master
                    // 更新本地信息中这个node的flag  <==================
                    clusterSetNodeAsMaster(sender);
                } else {
                    /* Node is a slave. */
                }
            }

            // sender是master, 同时dirty_slots为1表示slots的归属发生了变化 <=======================
            clusterNode *sender_master = NULL; /* Sender or its master if slave. */
            int dirty_slots = 0; /* Sender claimed slots don't match my view? */

            if (sender) {
                sender_master = nodeIsMaster(sender) ? sender : sender->slaveof;
                // 比较slots这个bitmap, 如果不一样, dirty_slots就为1 <=====================
                if (sender_master) {
                    dirty_slots = memcmp(sender_master->slots,
                            hdr->myslots,sizeof(hdr->myslots)) != 0;
                }
            }

            /* 1) If the sender of the message is a master, and we detected that
             *    the set of slots it claims changed, scan the slots to see if we
             *    need to update our configuration. */
            // slots发生了变化, 更新slots信息!!!!!!!!!!!!!! <============================
            if (sender && nodeIsMaster(sender) && dirty_slots)
                clusterUpdateSlotsConfigWith(sender,senderConfigEpoch,hdr->myslots);

        }
    
    }


    /* Reconfigure the specified node 'n' as a master. This function is called when
     * a node that we believed to be a slave is now acting as master in order to
     * update the state of the node. */
    // 这个函数只是单纯的把sender在本地的角色设置为master
    // 并没有判断sender是否是myself的slave
    // 降级处理是在下面  <========================
    void clusterSetNodeAsMaster(clusterNode *n) {
        // 如果这个node本来就是master, 那么退出 <===================
        if (nodeIsMaster(n)) return;
    
        if (n->slaveof) {
            clusterNodeRemoveSlave(n->slaveof,n);
            if (n != myself) n->flags |= CLUSTER_NODE_MIGRATE_TO;
        }
        // 更新各种flag <===================
        n->flags &= ~CLUSTER_NODE_SLAVE;
        n->flags |= CLUSTER_NODE_MASTER;
        n->slaveof = NULL;
    
        // 保存配置
        /* Update config and state. */
        clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|
                             CLUSTER_TODO_UPDATE_STATE);
    }


master可能被降级为slave
-----------------------------

1. sender的slot和我们没关系, 说明这个sender的master不是我们自己, 更新一下信息slots信息

2. sender的slot和我们有关系, 但是我们不是master而是slave, 所以我们要重配置, 更新一下自己的master信息和slot信息

3. sender的slot和我们有关系, 同时我们是master. 说明我们之前是master但是被其他slave给抢占了master角色, 所以我们需要自己降级为slave

   降级其实就是设置自己为slave, 进行重配置, 也就是把自己的master设置为sender. 2和3的处理一样的

4. 这里要注意一点就是下面的dirty_slot_count和dirty_slots这个两个是用来表示

   如果把一个master A的slot被移动到另外一个master B中, 同时A中这些slot中还有key

   那么我们不需要把A设置为slave, 而只是需要把这些dirty的信息给清除掉

   所以在下面最后的if else中, 如果curmaster->numslots为0, 表示A中所有的slot都被移动到其他节点X

   那么A是掉线的master, 此时被降级为slave, 所以走重配置master逻辑

   如果curmaster->numslots不为0, 显然是部分迁移A的slot到另外一个master, 那么显然不走重配置而是把dirty_slots中的key给删除掉而已!!!!

   **注意的是这个情况虽然是master到master的slot迁移, 但是不是走importing和migrating状态的, 不一样**

5. 这里有个前提是slave A1中的slots的归属节点是指向A1的master A, 而不是A1

   所以A的slave A2, A3都不会设置dirty_slots和dirty_slots_count的, 直接重新配置master



.. code-block:: c

    // 更新slots信息
    // 这里有包含了2中情况
    void clusterUpdateSlotsConfigWith(clusterNode *sender, uint64_t senderConfigEpoch, unsigned char *slots) {
        int j;
        clusterNode *curmaster, *newmaster = NULL;
        /* The dirty slots list is a list of slots for which we lose the ownership
         * while having still keys inside. This usually happens after a failover
         * or after a manual cluster reconfiguration operated by the admin.
         *
         * If the update message is not able to demote a master to slave (in this
         * case we'll resync with the master updating the whole key space), we
         * need to delete all the keys in the slots we lost ownership. */
        uint16_t dirty_slots[CLUSTER_SLOTS];
        int dirty_slots_count = 0;
    
        /* Here we set curmaster to this node or the node this node
         * replicates to if it's a slave. In the for loop we are
         * interested to check if slots are taken away from curmaster. */
        // 如果我们是master, 那么这个curmaster就是我们自己  <=======================
        // 否则这个curmaster就是我们的master
        // 而newmaster则会在下面赋值为sender的!!!! <===========================
        curmaster = nodeIsMaster(myself) ? myself : myself->slaveof;
    
        // 如果sender是自己, 那么就不需要更新了!!!!!!!!!!!!!
        if (sender == myself) {
            serverLog(LL_WARNING,"Discarding UPDATE message about myself.");
            return;
        }
    
        for (j = 0; j < CLUSTER_SLOTS; j++) {
            if (bitmapTestBit(slots,j)) {
                /* The slot is already bound to the sender of this message. */
                if (server.cluster->slots[j] == sender) continue;
    
                /* The slot is in importing state, it should be modified only
                 * manually via redis-trib (example: a resharding is in progress
                 * and the migrating side slot was already closed and is advertising
                 * a new config. We still want the slot to be closed manually). */
                // 一个状态为importing的slot会在最后更新的, 所以这里没必要更新 <======================
                if (server.cluster->importing_slots_from[j]) continue;
    
                /* We rebind the slot to the new node claiming it if:
                 * 1) The slot was unassigned or the new node claims it with a
                 *    greater configEpoch.
                 * 2) We are not currently importing the slot. */
                // 如果这个slot的configEpoch小于sender的configEpoch
                // 说明slave提升了epoch, 那么可以修改配置
                // 如果该slot没有人负责, 那么直接可以赋值  <===================================
                if (server.cluster->slots[j] == NULL ||
                    server.cluster->slots[j]->configEpoch < senderConfigEpoch)
                {
                    /* Was this slot mine, and still contains keys? Mark it as
                     * a dirty slot. */
                    // 本来这个slot是我们(我们是master)负责, 同时sender不是我们自己
                    // sender要抢走这个slot!!!!!!!!!!!!!! <=======================
                    // 设置dirty_slot_count和dirty_slots数组
                    // 如果我们是slave, 那么不会设置dirty_slots_count和dirty_slots的
                    // 所以意味着是把我们这个master负责的slot移动另外一个
                    // dirty_slots_count和dirty_slots是保存那些是本来我们负责的slot同时这些slot
                    // 还有key, 我们需要删除这些key!!!!!!!!!!!!!! <==========================
                    if (server.cluster->slots[j] == myself &&
                        countKeysInSlot(j) &&
                        sender != myself)
                    {
                        dirty_slots[dirty_slots_count] = j;
                        dirty_slots_count++;
                    }
    
                    if (server.cluster->slots[j] == curmaster)
                        // 把newmaster赋值为sender
                        newmaster = sender;
                    // 修改两个slots数组 <===============================
                    // 同时减少curmaster->numslots的计数和增加sender->numslots的计数!!!!!!!!!!!!!!!!!!!!!!
                    //
                    clusterDelSlot(j);
                    clusterAddSlot(sender,j);
                    clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|
                                         CLUSTER_TODO_UPDATE_STATE|
                                         CLUSTER_TODO_FSYNC_CONFIG);
                }
            }
        }
    
        /* If at least one slot was reassigned from a node to another node
         * with a greater configEpoch, it is possible that:
         * 1) We are a master left without slots. This means that we were
         *    failed over and we should turn into a replica of the new
         *    master.
         * 2) We are a slave and our master is left without slots. We need
         *    to replicate to the new slots owner. */
        // 如果curmaster->numslots为0, 表示没有需要清除的slot, 那么设置设置master
        // 因为我们在调用clusterDelSlot和和clusterAddSlot的时候已经操作过
        // 注意如果是一个master变为slave, 即使我们的dirty_slots_count不为0
        // 但是其实curmaster->numslots为0， 我们也不需要删除
        // 因为我们会变为slave, 和新的master同步slot!!!
        if (newmaster && curmaster->numslots == 0) {
            serverLog(LL_WARNING,
                "Configuration change detected. Reconfiguring myself "
                "as a replica of %.40s", sender->name);
            clusterSetMaster(sender);
            clusterDoBeforeSleep(CLUSTER_TODO_SAVE_CONFIG|
                                 CLUSTER_TODO_UPDATE_STATE|
                                 CLUSTER_TODO_FSYNC_CONFIG);
        } else if (dirty_slots_count) {
            // 我们是master, 但是我们只需要移动一部分slot到其他的master中
            // 所以我们不需要降级, 只是需要
            // 清除一下dirty_slos中的key!!!!!!! <==========================
            /* If we are here, we received an update message which removed
             * ownership for certain slots we still have keys about, but still
             * we are serving some slots, so this master node was not demoted to
             * a slave.
             *
             * In order to maintain a consistent state between keys and slots
             * we need to remove all the keys from the slots we lost. */
            for (j = 0; j < dirty_slots_count; j++)
                delKeysInSlot(dirty_slots[j]);
        }
    }



网络分区和可用性
===================

https://stackoverflow.com/questions/29615478/resharding-keys-when-a-node-goes-down-in-redis-cluster

https://stackoverflow.com/questions/63242537/difference-between-cluster-allow-reads-when-down-and-cluster-require-full-cov

redis cluster并不是那种每个instance都包含整个数据备份的模式(比如Elastic), 而是每个instance只负责整个数据的某一个部分, 所以一旦某个master group(master和其slave)掉线

那么显然整个集群都是不可用状态

当把7000和其slave都下线之后, 任何读写都失败

.. code-block::

    allenling@allenling-ubuntu:~/redis-cluster-test$ redis-cli -c -p 7001
    127.0.0.1:7001>
    allenling@allenling-ubuntu:~/redis-cluster-test$ redis-cli -c -p 7001
    127.0.0.1:7001> get hello
    (error) CLUSTERDOWN The cluster is down
    127.0.0.1:7001>

如果想要集群在某个master group掉线只会能继续服务, 那么有redis有两个参数可以设置

1. cluster-allow-reads-when-down, 默认是no

   这个参数会允许读操作, 不允许写操作. 下面7000和其slave掉线, 但是我们依然能读取在7001/7002上的key, 但是无论写入哪个master都不允许

   .. code-block::

       allenling@allenling-ubuntu:~/redis-cluster-test$ redis-cli -c -p 7001
       127.0.0.1:7001> get hello
       -> Redirected to slot [866] located at 127.0.0.1:7000
       Could not connect to Redis at 127.0.0.1:7000: Connection refused
       Could not connect to Redis at 127.0.0.1:7000: Connection refused
       not connected> get hello
       allenling@allenling-ubuntu:~/redis-cluster-test$ redis-cli -c -p 7001
       127.0.0.1:7001> get foo1
       -> Redirected to slot [13431] located at 127.0.0.1:7002
       "1"
       127.0.0.1:7002> set foo2 2
       (error) CLUSTERDOWN The cluster is down and only accepts read commands
       127.0.0.1:7002> set hello 1
       (error) CLUSTERDOWN The cluster is down and only accepts read commands


2. cluster-require-full-coverage, 默认是yes

   该参数如果是yes, 表示所有的master都在线才会可读写, 如果设置为no, 那么只需要超过一半的master在线就可读写!

   但是这里注意, redis cluster中每个节点都会负责一部分数据, 所以可写肯定是写当前可达的instance, 数据不一致也仅仅是在master-slave之间

   .. code-block:: c

       void clusterUpdateState(void) {
           int j, new_state;
           int reachable_masters = 0;
           static mstime_t among_minority_time;
           static mstime_t first_call_time = 0;
           new_state = CLUSTER_OK; // 默认是OK
           /* Check if all the slots are covered. */
           // 这里如果cluster-require-full-coverage是yes的话, 逐个校验slot是否在线  <======================
           if (server.cluster_require_full_coverage) {
               for (j = 0; j < CLUSTER_SLOTS; j++) {
                   if (server.cluster->slots[j] == NULL ||
                       server.cluster->slots[j]->flags & (CLUSTER_NODE_FAIL))
                   {
                       new_state = CLUSTER_FAIL;
                       break;
                   }
               }
           }
           /* If we are in a minority partition, change the cluster state
            * to FAIL. */
           // 否则只需要保证大多数master可达就可读可写了 <=============================
           {
               int needed_quorum = (server.cluster->size / 2) + 1;
               if (reachable_masters < needed_quorum) {
                   new_state = CLUSTER_FAIL;
                   among_minority_time = mstime();
               }
           }
       }

所以在redis specificiation中提到必须大部分master可达并且每个master至少有一个slave存活

*In the majority side of the partition assuming that there are at least the majority of masters and a slave for every unreachable master* 不准确, 至少在代码中并不是这样的

所以如果一个网络有3个master+3个slave, 我只需要2个master在线就可以读写(当然是针对存活着的master)



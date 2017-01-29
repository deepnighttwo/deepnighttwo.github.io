---
layout: post
title: "HBase vs Aerospike 第一篇"
keywords: ["aerospike","hbase","nosql"]
description: "hbase vs aerospike"
category: default
tags:     [dataprocess]
---

# HBase vs Aerospike

简单介绍一下HBase和Aerospike。目标是对NoSQL选型时，初步了解HBase和Aerospike的特点和长短处。首先抛出一个简单的浓缩的结论，HBase=Distributed SortedMap。 Aerospike=Distributed HashMap

本文假设读者已经知道NoSQL的基本概念。比如KV Store，和传统关系型数据库的区别等。


# HBase

HBase是hadoop生态系统里的NoSQL。简单来说就是一个巨大的、分布式的SortedMap。

## 存储

HBase本身依托底层的HDFS作为存储。所以它自己是不处理存储的。所以HBase的存储也就天然拥有HDFS的那些优点，比如一份数据是会存储到三个物理机的，所以，如果有一个存储的物理机宕机，理论上来说另外一个服务器可以透明的继续提供服务，而作为应用的HBase本身对这种事情可以说无感。

## 分区 （Region）

作为一个分布式的存储系统，HBase将一个表（HTable）分为一个或者多个、key的区间首尾相连的Region。每个Region的最重要的两个属性就是start key和end key。分别代表这个region中开始的key和结束的key，因为是排序的，所以也是第一个key和最后一个key。

用SortedMap打个比方，比如说有这么一个SortedMap，存放的是全国的学生。Key就是地区+学校名字+年级班级+学号。那么当这个table占用的内存已经大到无法在单台机体上存放的时候，我们很自然的会想到把它一分为二：所有比山东-济宁-一中-00级00班-00号大的key，扔到Host1，其余的扔到host2。这样，两个host上分别有一个SortedMap，两者有明确的分界线（山东-济宁-一中-00级00班-00号），当要存储/查找一个key的时候，通过一个比较算法，可以明确的知道去哪个机器找。那么再接下来，两台机器如果都不够用了怎么办？可以按照key排序的顺序，继续分：

- 所有大于0，小于A省B市-XX级YY班级-MM号的，在Host1；
- 所有大于等于A省B市-XX级YY班级-MM号的，小于C省D市-XX级YY班级-MM号的，在Host2
- ......
- 所有大于等于X省Y市-XX级YY班级-MM号的，小于无穷大的，在HostX

这里一定要强调的一点是顺序。也就是Host2里的都大于Host1的。Host内部本身是排序的。到这里，就有点像分布式SortedMap的样子了。

当然HBase并非是这样实现的。首先HBase是持久化的，会把数据写到HDFS，而不是仅仅放在自己的内存。其次是，在HBase里，Region和提供服务的机器（HBase叫做RegionServer）是通过配置（meta表）关联的，是可以动态变化的，一个Region可以任意“漂流”到任何一个RegionServer上，都可以提供正常的服务。一个RegionServer上可以有任意多个Region。


HBase在读写操作的时候，HBaseClient会缓存每个Region的start key 和 end key，同时也会缓存Region和RegionServer的对应关系。这样，在Client端读写时，就可以通过key算出Region，通过Region找到RegionServer，直接向这个RegionServer发起请求。当然这个对应关系是动态变化的，当Client发现server返回一个“我这儿没有这个key”的错误时，client就知道该去meta表刷新一下自己缓存的对应关系了。


## Key的设计和scan

HBase的key是一个byte[]，无论业务上想用什么做为key，最终都要转换为byte[]数组才能写入HBase。读取的时候也是一样的。

在实际使用HBase时，Key的设计是至关重要的。这和HBase的scan操作有关。scan操作接受一个start key，一个end key，然后根据情况可以附加不同的参数（比如正则表达式过滤）等。有点点类似 select * from table where xxxxx. 里的where。但是实际上比这个功能要弱。因为所有的条件都必须在key里，而key只能是byte数组。

比如最简单的情况，只是使用start key和end key作为scan条件，可以做到找出“山东-济宁-一中-03级02班”到“山东-济宁-一中-03级03班-00号”的所有学生。翻译成人话就是找出03级02班的所有学生（假设一个班级不会有00号的学生）。

即使可以通过正则表达式过滤，意思还是一样的，start key和end key指定了扫描的区间。而这个区间越小，scan的速度就越快。

如果要找到所有A省内学号为1号的学生，那么start key和end key就只能设置为[“A省”，“B省”]，也就是说，所有A省的数据都要扫描一遍。

如果要找到全国学号为1的学生，那就只能全表扫描了。其效率可以说非常低。

从这个例子可以简单看出key对HBase scan的影响。简单来说，Key就是HBase的表索引。数据库的表的索引如果设计不好，会让应用很难办。同样的，如果HBase的Key设计的不能满足业务的需求，可能会导致表的数据实际不可用。因为如果对于个大的表做全表扫描，可以说肯定会超时的。

所以在设计key的时候，查询是越是必须、必要的字段，就越要放在前面。这是设计HBase key的时候的准则。当然如果没有scan的需求，可以忽视key的设计，只要能唯一的id一条记录就可以。

## 多索引和热点

既然把Key和索引关联了起来，多索引也就是顺理成章的事情了。HBase本身是不支持多索引的。有些厂商自己扩展了多索引的支持，但是现在好像声音也不大，估计是有不少问题。

如果自己做多索引，一个简单直接的方法是创建n个索引表，一个实体表。简单的设计如下：对于实体表来说，key是一个UUID或者是一个hash值（后文通称为hashkey），value当然是真正的payload。对于每个索引表来说，可以自由根据业务组织自己的key，value则是前面实体表的key，也就是hashkey。这样，应对每种业务需求，可以设计不同的索引表。通过索引表的get/scan，可以得到实体表的keys，进而去实体表get到真正的值。虽然多了一个步骤，但是比起没有索引表时，需要扫描大量数据来说，还是有优势的。

这种情况下，一个附带的好处是将实体表的写请求均匀分散到了不同的Region上。因为是使用HashKey作为key，所以实体表的key实际上是没有业务意义的，也就不会因为业务的某个爆发点儿导致某个region的读写量暴增。


## CAP

HBase对CAP的取舍是：
 - Consistency： Yes。因为HBase是支持WAL的，而且每个region是delegate给一个RegionServer的，所以consistency没问题。
 - Availability：No。因为同一个时间只有一个RegionServer提供服务，所以HBase的Availability不是满分，如果一个RegionServer宕机了，就要有别的RegionServer(s)把这台宕机的server上的Region一个个launch起来，然后更新meta表等等。
 - Partition Tolerance：Yes。因为一个时间只有一个RegionServer提供服务，所以，即使子网之间段时间的不通，也不影响集群提供服务。


## Co-processor

HBase的另一个杀手锏功能就是协处理器，co-processor。简单来说，就是支持把计算挪到服务器端，而不是在客户端算好，再写到服务器端。

举个例子，假设每次有一个订单，订单系统都会将user id，total amount作为消息发送出去。一个业务系统要接受消息，同时把以user id做key的一条记录的某个value累加上total amount。

通常的做法是收到一个消息，读取user id，然后去HBase里读取需要累加的那一列的值x，然后计算x+total amount作为新的值，然后写回HBase。

这样的一个坏处是如果业务系统有多台机器，同时收到了两条user id相同的消息，就可能会写冲突/写覆盖。还有就是这样会读一次，写一次。如果要解决冲突，就更麻烦了，可能要读至少两次。

而co-processor则是支持用户把这个计算x+total amount的过程注册在HBase server端。通过调用，只要传递user id和total amount就可以完成计算，仅一次调用，而且可以避免写冲突。

其缺点：

 - 会占用HBase服务器资源。但是因为其本身是IO密集型的服务，所以如果计算量不大，可以考虑用co-processor。
 - 需要把自定义的Jar包发布到HBase集群
 - 如果自定义的Jar包有问题，可能会影响整个集群的稳定性。

## Tips & Best Practice

 - HBase不支持表的重命名。因为HBase用表明作为目录名，为每个表创建了单独的目录。
 - 为了避免读写热点，可以采取实体表和索引表分开的策略。
 - HBase支持自己根据表大小做split，也根据预先做好region的分配，既pre split。建议根据实际集群情况做pre split。HBase的Split是不稳定之源。
 - HBase是基于HDFS的，也就是说，HBase如果想玩起来，得先有一个hadoop集群。
 - HBase支持bloom filter。
 - HBase支持server端的Atomic Compare and Set / Atomic Increment / Atomic Append操作


## HBase总结

HBase把数据排序，分为不同的Region（这个过程是动态的，开始可能只有一个Region），每个Region中的数据都是排序好的，每个Region对应着HDFS中的一个文件夹，文件夹中有这个Region相关的数据。任何一个RegionServer都可以读取这个文件夹的数据，然后就可以对外提供这个Region的服务（数据读写scan等）。

HBase的两大法宝：scan和co-processor。

HBase的key设计很重要，要先把最重要的key放在前面。

HBase可以支持非常大的数据量，几十几百T。


# Aerospike

Aerospike是一个纯粹的NoSQL服务。它有社区版和商业版，主要是管理工具和监控支持上的不同。它不依赖于其它生态系统，即装即用，可以集群，也可以单台。下文简称Aerospike为AS。

## 存储的基本概念

使用AS首先要创建至少一个Namespaece。一个namespace可以指定其rep count，使用的内存大小，存储方式是仅仅内存还是内存磁盘混合，TTL等。

一个Namepsace会分为1024个partition。类似于HBase中的pre split。

一个Namespace下可以有一个或多个Set。如果和数据库类比的话，Namespace类似于DB，Set类似于Table。

要定位一条记录，需要namespace，set以及记录的key。


## 关于Key的那些“奇葩”

### Hash Key? Hash Key Only!

Aerospike的一个特点就是使用Hash Key做为存储和查询的Key。和Java中的HashMap不一样的是，HashMap是会解决hash冲突的，而Aerospike不会。它假设不会冲突。具体可以参见我当时提的这个问题https://discuss.aerospike.com/t/what-will-aerospike-do-if-there-is-a-hash-collision-two-records-have-the-same-key/779

简言之，AS会把Set和Key的名字作为输入，通过RIPEMD-160 hash算法得到一个20B的digest作为key。这个算法可以保证冲突的可能极为低。一个long是8个B，20个B已经是两个long外加4个B。可以相信如果hash算法足够好，确实可以有效避免冲突。

还有，缺省AS也不会存储原始的key。

### 所有的Key都在内存里

是的，你没有看错。AS在启动的时候，会把key全部读到内存里，无论是不是用到。这个过程不是lazy loading。这带来的一个限制就是，一台机器／一个集群能存储多少条记录，可以根据内存大小算出来。因为一个key占用20 B，假设磁盘存储不是问题，总内存大小／20B就是这个集群／机器能容纳的总的记录数。

## UDF 

HBase有co-processor，可以把计算挪到服务器端。相应地AS也有UDF（User Defined Function），可以通过注册Lua脚本来实现server side computing。相比HBase，注入Lua脚本在实际操作上更简单灵活。

## 集群

AS的集群规模支持的数量不会太大。它有两种组集群的方式，一种最多支持一百多台，另一种稍多，但是也是百台规模。

因为每个namespace都被分成了1024个partition，所以在集群状态下，这些partition被分配到不同的机器上。又因为AS的key是被hash过的，所以记录被均匀的分配到这些partition上。这样，AS巧妙的让整个集群每个机器的负载都很均匀。

## CAP

和HBase一样，AS对CAP的取舍也是CP。当一台机器宕机的时候，AS也需要一段时间才能恢复那台机器上的partition。尤其是AS底层没有hdfs这种网络存储，所以在一台机器宕机后，为了保证rep count，会有大量的数据拷贝。

## 文档

Aerospike的支持做的还是不错的，有问题直接去论坛问，不出一两天就会有专人回答。而且他们在stackoverflow上好像也有人值守，在上面提问，只要注明aerospike这个tag，也会有专人回答的。

Aerospike的文档做的也不错，而且组织的很好，属于从入门到上手的。

如果对AS有兴趣，在看过官方文档之后，可以看一遍我当时提的问题https://discuss.aerospike.com/users/deepnighttwo/activity/topics。


## 总结

AS在设计的理念上就是保持简单有效。相比之下，HBase在设计和概念上要比AS复杂的多。结果就是，HBase远不如AS稳定。AS把自己的优势做到最好，然后把不尝试用一些复杂／蹩脚／不稳定的设计去弥补这些不足。这样带来的好处就是，让AS成为一个很有针对性很有特点的NoSQL。这些特点里最重要的一个就是稳定。


{% include references.md %}

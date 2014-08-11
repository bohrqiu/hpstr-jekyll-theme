---
layout: post
title: "zookeeper 乱七糟八"
image:
  feature: abstract-10.jpg

tags: [sample post, images, test]
comments: true
share: true
---
##原理相关:

1. Three ZooKeeper servers is the minimum recommended size for an ensemble, and we also recommend that they run on separate machines. Because Zookeeper requires a majority, it is best to use an odd number of machines.To create a deployment that can tolerate the failure of F machines, you should count on deploying 2xF+1 machines. 
<!--more-->
2. The replicated database is an in-memory database containing the entire data tree. Updates are logged to disk for recoverability, and writes are serialized to disk before they are applied to the in-memory database.
3. Read requests are serviced from the local replica of each server database. Requests that change the state of the service, write requests, are processed by an agreement protocol.As part of the agreement protocol all write requests from clients are forwarded to a single server, called the leader. The rest of the ZooKeeper servers, called followers, receive message proposals from the leader and agree upon message delivery. The messaging layer takes care of replacing leaders on failures and syncing followers with leaders.(读请求直接读本地节点，写请求写到leader节点，其他follower节点从leader同步)

4. zookeeper对于修改操作会优先顺序写入事务日志，保证可用性，所以不能使用很繁忙的设备写事务日志。

5. dataDir store the in-memory database snapshots 

6. 使用 daemontools or SMF来保证zookeeper进程down后自动重启

7. If you only have one storage device, put trace files on NFS and increase the snapshotCount; it doesn't eliminate the problem, but it should mitigate it.

8. 可以在zookeeper集群前使用L4 LB,但是要注意LB failure detect机制，保证能快速的发现后端失败的服务器.不然的话，请求有可能被发送到down掉的server上。（[Can I run an ensemble cluster behind a load balancer?](https://cwiki.apache.org/confluence/display/ZOOKEEPER/FAQ )）

9. 在每个ZK机器上，我们需要在数据目录（数据目录就是dataDir参数指定的那个目录）下创建一个myid文件，myid中就是这个Server ID数字。

10. zookeeper监控

		http://jm.taobao.org/2012/01/12/zookeeper%E7%9B%91%E6%8E%A7/ 

11. java client

		https://github.com/Netflix/curator

	应该是最优秀的zookeeper client了，避免原生client的一些问题，并提供一些常用的工具

12. zookeeper eclipse插件

 		http://www.massedynamic.org/mediawiki/index.php?title=Eclipse_Plug-in_for_ZooKeeper

13. 跨机房使用Observer

	Observer不参与选举,永远是follower.跨机房时,读请求直接在Observer上搞定,写操作转到leader
14. 其他资料：


		http://rdc.taobao.com/team/jm/archives/2318
		https://cwiki.apache.org/confluence/display/ZOOKEEPER/Index
		https://cwiki.apache.org/confluence/display/CURATOR/Third+Party+Articles
		http://wiki.apache.org/hadoop/ZooKeeper/FAQ
		
-------------

##开发相关:
###1.怎么处理CONNECTION_LOSS?
	
CONNECTION_LOSS意味着客户端和服务器端的连接坏了(坏了并不一定就是完全断开了,可能是连接不稳定).此异常并不意味着发送给zookeeper服务端的请求处理失败.当create请求发送到zk服务器并处理后,在响应时,网络坏了.这种情况下,需要用户判断是否处理成功或者重试.
	
zookeeper官方提供的Locker通过在EPHEMERAL_SEQUENTIAL中加入sessionId来判断当前session是否已经创建node.
###2.怎么处理SESSION_EXPIRED?
	
SESSION_EXPIRED说明客户端和zk服务端已经出现了网络分区,并且分区间隔时间超过了session timeout,zk服务器认为此client已经挂掉了.
	
zk集群来管理会话过期.zk client和服务端建立连接时,会设置一个timeout值,集群通过此值来确定session过期.当session过期后,集群会删除此会话的所有的ephemeral节点,并通知watcher.**在这个时候,会话过期的节点和集群的连接仍然是断开的,当连接被重新建立后,会被通知session expiration.(客户端收到session expiration是当客户端又和zookeeper服务端重连后,client在网络异常的情况下,不知道何时session expiration)**
	
	状态流转的例子如下:
	
		1. 'connected' : session is established and client is communicating with cluster (client/server communication is operating properly)
		2. .... client is partitioned from the cluster
		3.'disconnected' : client has lost connectivity with the cluster
		4. .... time elapses, after 'timeout' period the cluster expires the session, nothing is seen by client as it is disconnected from cluster
		5. .... time elapses, the client regains network level connectivity with the cluster
		6. 'expired' : eventually the client reconnects to the cluster, it is then notified of the expiration

	
	
###3.InterruptedException
	
ZooKeeper遵循java的线程中断机制,也通过此方式来取消用户操作.可以参照下面的代码,让InterruptedException中断操作,或者把此异常抛出去,让使用方来处理.
	
		@Override
		public void process(WatchedEvent event) {
			if (event.getType() == EventType.NodeDataChanged) { 
				try {
					displayConfig();
				} catch (InterruptedException e) {
					System.err.println("Interrupted. Exiting."); 
					Thread.currentThread().interrupt();
				} catch (KeeperException e) { 
					System.err.printf("KeeperException: %s. Exiting.\n", e);
				} 
			}
		}


###4.time
	
A low session timeout leads to faster detection of machine failure. 

Applications that create more complex ephemeral state should favor longer session timeouts, as the cost of reconstruction is higher. In some cases, it is possible to design the application so it can restart within the session timeout period and avoid session expiry. (This might be desirable to perform maintenance or upgrades). **Every session is given a unique identity and password by the server, and if these are passed to ZooKeeper while a connection is being made, it is possible to recover a session (as long as it hasn’t expired)**. An application can therefore arrange a graceful shutdown, whereby it stores the session identity and password to stable storage before restarting the pro- cess, retrieving the stored session identity and password and recovering the session.

###5.The single event thread
	
每个zookeeper对象有一个线程来分发事件给watchers.如果你在watcher里面处理很耗时的操作,那么其他watchers会等这个watcher内处理完.

###6.官方demo分析

基本原理:

在`dir`(简单理解为lock string)下面创建sequential ephemeral node(后面简称Node),如果创建的node序列号最小,则表明当前连接就是锁的持有者.

这里需要注意几个地方:

1. 在创建Node过程中网络分区,导致ConnectionLossException
   
   比如在zookeeper服务端节点已经创建成功了,但是在返回结果时,网络出问题了.这个时候我们唯一能做的只是重试.但是重试创建节点操作又会导致新的Node被创建,最后出现没人管的孤儿节点,死锁奇迹般的发生了.
   
   出现ConnectionLossException异常后,节点通过重试,SeesionId不会改变.官方的例子通过`"x-" + sessionId + "-"`作为`prefix`来检查这种情况.
   
   在排序Node时,先比较`prefix`,再比较`sequence`.

2. 羊群效应
   
   如果在getChildren时加上watcher会导致每次child被删除时,所有child都会收到通知.如果在集群规模很大时,就很悲剧了.这里可以只监听比当前序号小的Node,来减少网络压力.比如Node-27监听Node-26,Node-26监听Node25

3. 其他异常处理

   对于不可恢复的异常,比如`SessionExpiredException`(The session timeout may not be less than 2 ticks or more than 20),这个异常需要抛给用户来处理,可以通过回调的方式让用户加入异常处理逻辑.
      
   `InterruptedException`异常要么抛出去,要么调用`Thread.currentThread().interrupt()`,别抓住了自己吃掉.
   
   
官方的lock实现还算不错,只是太低层次了点(重试带着简单粗暴的美感,但是对于非幂等性操作谨慎).我们还需要通过`LockListener`结合`CountDownLatch`(当lockAcquired时countDown),来提供可阻塞的API.当然最好是我们弄个`java.util.concurrent.locks.Lock`的分布式锁实现.

参考:

	http://wiki.apache.org/hadoop/ZooKeeper/Troubleshooting
	https://www.inkling.com/read/hadoop-definitive-guide-tom-white-3rd/chapter-14/building-applications-with
	http://wiki.apache.org/hadoop/ZooKeeper/ErrorHandling



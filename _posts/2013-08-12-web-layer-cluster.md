---
layout: post
title: "web服务器集群技术"
image:
  feature: abstract-10.jpg

tags: [sample post, images, test]
comments: true
share: true
---

web服务器集群技术包括web负载均衡和http session 失效转移
<!--more-->
##1.负载均衡
负载均衡我们主要关注以下四点：

###1.1 实现负载均衡的算法

实现的算法很多，可以参考[此文章](http://www.cnblogs.com/shanyou/archive/2012/11/09/2763272.html)。最好是选用通过检测后端服务器状态来实现最优的负载均衡。

###1.2 健康检查

当一台服务器失效了，负载均衡器应当检测出失效并不再将请求分发到这台服务器上。同样，它也要检测服务器是否恢复正常，并恢复分发请求。

健康检测要关注检测状态所消耗的时间，比如haproxy如下的配置：

		check inter 2000 rise 2 fall 3

检测周期为2s，连续成功2次认为节点恢复，连续失败3次认为节点不能提供服务。这里就存在负载均衡和后端服务状态不一致的时间窗口6s，我们需要通过一些机制或者手段去掉这6s对用户的影响或者尽量减少对用户的影响。

通过haproxy的redispatch机制，我们可以减少对用户的影响。通过在负载均衡上主动关闭流量，我们几乎可以做到完全屏蔽用户的影响(关闭流量会引导新请求到其他节点，对于正在处理的请求，我们最好是等一段时间，让他处理完，等待时间和影响用户感知，这个由业务来权衡)。

###1.3 会话粘滞

机制很多，我觉得比较好的方式是在第一次http请求时向cookie写入节点信息(减少代理上网造成的不均衡)，后续的请求都转发到此节点。

我们需要根据不同的应用选择是否启用会话粘滞。如果是接口调用，我们没有必要支持会话粘滞；如果是web页面，我们需要启用此特性。还需要注意的一点是，负载均衡上的会话超时时间设置应该大于或者等于web容器的会话超时时间设置。

###1.4 其他

其他功能和负载均衡关系不是很大，但是可以放在负载均衡设备上来做。比如ssl卸载、gzip压缩、内容缓存。在我们访问量比较少的情况下，这些操作还是放在web容器来做吧，省钱。

##2. Session失效转移
session失效转移是说在用户访问的某节点挂掉后，用户还能够正常的获取session做操作，这里面通常会涉及到三个问题：

###2.1 全局http session id

如果session不能唯一，这肯定要天下大乱，后面我会谈到在memcached-session-manager中怎么保证session id不重复

###2.2 如何备份会话状态

常见的机制有：数据库备份、广播复制(所有集群内的web容器都保存所有的会话)、对等复制(每台服务器任意选择一台服务器备份)、中心状态服务器复制(session保存到中心服务器)、分布式缓存(现在大多数互联网企业选择的方案)

###2.3 备份的频率和粒度

备份频率和粒度很影响性能和可靠性

####2.3.1 备份频率：

 - 在web请求处理结束后备份 
 - 固定时间间隔备份
	
memcached-session-manager灵活的使用了web请求结束后备份和固定时间间隔机制检查，来提高性能。

####2.3.2 备份粒度：

- 整个会话

	每次都备份整个会话，这样可以带来易用性，但是性能不佳
- 修改过的会话

	仅当会话修改后才备份会话。当“session.setAttribute()”或 “session.removeAttribute()”被调用后，则认为会话被修改过。所以这种方式，我们在修改会话内的对象时，必须主动调一次set/removeAttribute,让它知道这个会话已经被修改。我们可以在序列化对象时做压缩尽量减少网络的开销。
- 修改过的属性

	这种方式带来最小的网络开销，可能会遇到一些问题。比如后端缓存服务器是否支持，属性之间的交叉引用如何识别等。属性交叉引用可以通过计算所有属性的hash值来判断某属性的修改是否会影响到其他属性，我们需要权衡网络开销和cpu消耗

##3. 我们的选择
###3.1 负载均衡

负载均衡可以用硬件或者软件来实现。如果从成本的角度考虑，现阶段用软负载可能更好。我们需要从性能、稳定性、负载的产品重要性几个方面来考虑

###3.2 session失效转移

我们采用缓存集群来保存session，数据的可用性、一致性交给缓存集群。备份粒度和频率我们通过组件来实现。

####3.2.1 分布式缓存产品的选择

备份的粒度是决定我们选择分布式缓存产品的一个重要因素。备份整个会话或者修改过的会话，我们可以选择key-value类型的nosql缓存组件。如果我们要支持`备份修改过的属性`,我们需要选择支持更多数据(比如支持内置的命名空间，MAP)结构的nosql缓存组件。

memcached、mongodb、redis、tair、mongodb、voldemort...有很多很多nosql产品我们可以选择。选择机会多了，选择也就越难了。

根据CAP理论，我们只能在一致性、可用性、分区容错性上取舍，根据不同的应用场景来选择不同的处理方式。没有绝对的最优，只有不断的根据我们不同阶段的特点选择不同的产品。

对于缓存来说最好的选择方案是选择支持灵活的路由机制(服务端路由或者client路由保证AP)，支持丰富的数据结构(从易用性和性能考虑)，支持数据持久化(保证A，最好是有这个特性，当然缓存嘛，只是来加快应用的，不应该把数据只存在缓存中，不能保证高可用)，支持多版本控制(尽量保证C)。

下图是voldemort(有幸参与过此产品的应用开发)的物理架构图。大多数的nosql产品都面临这下面三种物理架构的抉择。

![](http://www.project-voldemort.com/voldemort/images/physical_arch.png)

memcached、redis只支持客户端路由，严格意义上讲，它算不上分布式缓存组件。如果要达到高可用性和分区容错性，我们需要自己来存多份([NRW](http://www.project-voldemort.com/voldemort/design.html)Routing Parameters部分)

选择mongodb、mongodb、tair、voldemort算是比较好的方案，鉴于我们的运维能力，可能hold不住，暂时只能呵呵了。tair相对来说，很适合我们的应用场景，支持比纯KV更丰富的数据结构，支持服务端路由，支持服务端NRW。

目前我们选择memcached作为缓存组件。


####3.2.2 session备份组件的选择

我们可以通过filter来备份session。但是对于后端缓存组件选用memcached来说，这样会存在一个问题。存储的key为sessionid，通过hash或者[一致性hash](http://blog.csdn.net/sparkliang/article/details/5279393)来实现路由，这样在memcached集群拓扑变动时，会造成路由的迁移(拓扑变动造成路由到不同的memcached服务器)。对于应用来说，缓存丢失了。

如果我们能把第一次选择memcached节点写入到sessionid里面，后续的请求都根据sessionid中的node信息选择memcached，这样在节点动态调整时，不会造成缓存丢失。但是我们在filter中不能改变sessionid的值，所以我们选择了[memcached-session-manager](https://code.google.com/p/memcached-session-manager/)。

##4.memcached-session-manager

###4.1 主要特性如下：


- Supports Tomcat 6 and Tomcat 7

- Handles sticky or non-sticky sessions

	启用session sticky时，memcached作为二级缓存，tomcat不挂掉时，不会从memcached取数据

- No Single Point of Failure

- Handles tomcat failover

	tomcat挂掉时从memcached读取session

- Handles memcached failover

	non-sticky模式下，由于jvm没有缓存session，它会把session存到两台memcached，保证可用性。
	在sticky模式下，jvm缓存着session，一台memcached也会存session，也保证了可用性。

	当然，这个只是相对的保证了可用性，不能完全保证可用性。

- Comes with pluggable session serialization

	我们可以选择kryo作为序列化组件

- Allows asynchronous session storage for faster response times
	
	在请求响应之前，异步写入session到memcached

- Sessions are only sent to memcached if they're actually modified

	仅当session被修改时，才存储session

- JMX management & monitoring
	
	提供JMX管理监控功能

###4.2 代码分析
考虑到性能，我们只采用sticky模式(jvm和一个memcached中存session)，主要的功能实现如下：

- session创建

	a.`request.getSession()`调用`MemcachedSessionService#createSession`,创建session

	b.用`org.apache.catalina.util.SessionIdGenerator`生成sessionid，

	c.在sessionid中加入memcached节点信息(`MemcachedSessionService.newSessionId`，通过`NodeIdService.getMemcachedNodeId`随机选择节点)
	
	注意：SessionIdGenerator只能保证jvm内的不重复，多个jvm下需要另外的id生成机制，如果加上jvmRoute可以避规这个问题。


- session恢复

	a.首先在本地session缓存中找session，如果有此session。就用此session

	b.如果本地缓存没有session，则MemcachedSessionService.findSession通过用户请求传来的sessionid从memcached服务器找session


- session存储

	在请求结束后(de.javakaffee.web.msm.RequestTrackingHostValve:173)，会检测session是否有修改(调用session.setAttribute标记此session被修改)。如果修改，(de.javakaffee.web.msm.BackupSessionService:205)创建一个BackupSessionTask，在检查到session内容改变后异步(通过序列化后的的byte数组做hash比较)写入memcached。


- session过期策略
	
	如果仅仅使用session.maxInactiveInterval，在session初始化时设置此key的过期时间。这需要在每次session被访问时都修改memcached中的getLastAccessedTime，这样做效率不是太好。所以在memcached-session-manager通过容器来提供周期性的回调，检查需要过期的时间。

	通过ContainerBackgroundProcessor线程来周期性的回调MemcachedBackupSessionManager.backgroundProcess()方法。过期时间为：session.maxInactiveInterval - timeIdle

	注意：需要保证memcached和应用服务器时间一致

- session销毁

	MemcachedBackupSessionManager.removeInternal，会把memcached和jvm中的session清理掉

	注意：在销毁session时如果memcached挂掉，会出现不一致的情况。

- 可靠性

	选择采用sticky模式时，没有多份复制数据。如果很不幸，tomcat和memcached都挂了，session就丢失了。见官方[maillist](https://groups.google.com/forum/#!topic/memcached-session-manager/W6z6eSuhAJ0)。

	在非sticky模式下，session会保存到两个memcached(MemcachedSessionService:1079)。但是tomcat本地没有存储session，会影响所有的请求性能。


- memcached状态检测

	在创建、查找session、恢复session时，都会通过NodeAvailabilityCache检测memcached状态，检测后端memcached状态间隔50ms(MemcachedNodesManager.NODE_AVAILABILITY_CACHE_TTL)

###4.3 最佳实践

根据上面的代码分析，以下的最佳实践适合我们。

对于我们的开发同学，session里面的对象修改后，需要setAttribute下。

对于运维同学：

- 设置tomcat jvmRoute，避免sessionid重复

- MemcachedSessionService.setMemcachedProtocol设置二进制协议

- MemcachedSessionService.setSessionBackupTimeout 默认异步操作100ms超时，在网络不好的情况下会出现大量的异常，设置长点。

- MemcachedSessionService.setOperationTimeout是memcached客户端和服务端通信时的超时时间,不能设置太短


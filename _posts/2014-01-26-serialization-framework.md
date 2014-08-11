---
layout: post
title: "序列化框架选型"
image:
  feature: abstract-10.jpg
tags: [sample post, images, test]
comments: true
share: true
---

###序列化框架选型
分布式应用中,序列化很关键,选择一个合适的序列化框架能给分布式应用带来性能红利,还能减少不必要的麻烦.本文仅仅从我遇到的一些实际问题来说明序列化的选型.更深入部分也可以参考[java序列化](http://bohr.me/2012/04/02/java-serialization.html)
<!--more-->
序列化框架的性能可以参考[jvm-serializers](https://github.com/eishay/jvm-serializers/wiki),`Kryo`的性能还是很牛叉的.

除了性能,还要考虑兼容性/序列化后的大小.如果仅仅考虑性能,我们选择`Kryo`就足够了.但是Kryo有些用着不爽的地方,比如不是线程安全的/兼容性.
####1.兼容性
字段增删不兼容,这个问题有时候很麻烦.比如用memache做缓存,把对象序列化后存入memcache,如果出现字段增删的情况,必须在服务重启的时候把缓存清空,不然就会导致[灰常严重的BUG](http://bohr.me/2013/11/03/kryo-oom.html).但是如果应用服务器有多台,这个问题还是避免不了.总会有个时间窗口会出现不同服务器上的同一个应用有不同的类版本,仍然可能会出现灰常严重的BUG.

现在的`Kryo`提供了兼容性的支持,使用`CompatibleFieldSerializer.class`,在`kryo.writeClassAndObject`写入的信息如下:

	class name|field lenght|field1 name|field2 name|field1 value| filed2 value

读入`kryo.readClassAndObject`时,会先读入`field names`.然后匹配当前反序列化类的field和顺序,构造结果.

子类和父类中有同名的字段时，kryo反序列化会丢失字段值,出现问题的原因和[hessian](http://bohr.me/2013/11/29/hessian-java-serialization.html)出问题一样.


给kryo提交了一个[improvement](https://github.com/EsotericSoftware/kryo/pull/187),在初始化类型信息时,去掉父类中重复名称的field.
####2.线程安全
非线程安全也很好处理,每次都new对象出来,当然这样不是最佳的使用方式.通过线程变量来解决会比较合理,保证了性能还能提供很方便使用的工具类.

####3.性能
下面测试了java下的各种序列化实现方式的性能

	0 Serialisation    write=4,206ns read=16,945ns total=21,151ns
 	1 Serialisation    write=3,626ns read=18,205ns total=21,831ns
	0 MemoryByteBuffer write=270ns read=324ns total=594ns
 	1 MemoryByteBuffer write=270ns read=330ns total=600ns
 	0 MemoryByteBufferWithNativeOrder  write=357ns read=360ns total=717ns
 	1 MemoryByteBufferWithNativeOrder  write=323ns read=359ns total=682ns
 	0 DirectByteBuffer write=236ns read=325ns total=561ns
 	1 DirectByteBuffer write=231ns read=301ns total=532ns
 	0 DirectByteBufferWithNativeOrder  write=261ns read=310ns total=571ns
 	1 DirectByteBufferWithNativeOrder  write=243ns read=290ns total=533ns
 	0 UnsafeMemory write=28ns read=82ns total=110ns
 	1 UnsafeMemory write=24ns read=75ns total=99ns
 	0 kryo write=373ns read=348ns total=721ns
 	1 kryo write=390ns read=386ns total=776ns
 	0 kryoWithCompatibleFields write=1,037ns read=1,625ns total=2,662ns
 	1 kryoWithCompatibleFields write=1,038ns read=1,657ns total=2,695ns
 	0 kryoWithCompatibleFieldsAndDuplicateFieldAccept  write=1,077ns read=1,560ns total=2,637ns
 	1 kryoWithCompatibleFieldsAndDuplicateFieldAccept  write=1,064ns read=1,583ns total=2,647ns
 	0 kryoWithUnsafe   write=164ns read=204ns total=368ns
 	1 kryoWithUnsafe   write=168ns read=210ns total=378ns
 	0 fastjson write=1,942ns read=5,834ns total=7,776ns
 	1 fastjson write=1,873ns read=5,879ns total=7,752ns
 	
每种序列化执行1000000次,并且有预热. 各组数据相对比较,可以得出一些结论:

* 直接调用unsafe,最快,但是最麻烦
* java自带的序列化很慢,最好不要用
* kryo2.22提供的unsafe支持,性能非常卓越
* kryo兼容性序列化器,开销挺大.写需要写入字段名,读的时候还需要做匹配撮合,读比写慢
* fastjson也挺快的,兼容性\跨语言互操性俱佳.

序列化后的字节大小可以参考[jvm-serializers](https://github.com/eishay/jvm-serializers/wiki)

---
layout: post
title: "java序列化"
image:
  feature: abstract-10.jpg
tags: [sample post, images, test]
comments: true
share: true
---

##什么是序列化

serialization is the process of converting a data structure or object state into a format that can be stored (for example, in a file or memory buffer, or transmitted across a network connection link) and "resurrected" later in the same or another computer environment.
<!--more-->
##JAVA内置序列化介绍

###基本介绍

由于Java提供了良好的默认支持，实现基本的对象序列化是件比较简单的事。待序列化的Java类只需要实现Serializable接口即可。Serializable仅是一个标记接口，并不包含任何需要实现的具体方法。实现该接口只是为了声明该Java类的对象是可以被序列化的。实际的序列化和反序列化工作是通过ObjectOuputStream和ObjectInputStream来完成的。ObjectOutputStream的writeObject方法可以把一个Java对象写入到流中，ObjectInputStream的readObject方法可以从流中读取一个Java对象。

在写入和读取的时候，虽然用的参数或返回值是单个对象，但实际上操纵的是一个对象图，包括该对象所引用的其它对象，以及这些对象所引用的另外的对象。Java会自动帮你遍历对象图并逐个序列化。除了对象之外，Java中的基本类型和数组也是可以通过 ObjectOutputStream和ObjectInputStream来序列化的。

序列化:

	try {
	    User user = new User("Bohr", "QIU");
	    ObjectOutputStream output = new ObjectOutputStream(new FileOutputStream("user.bin"));
	    output.writeObject(user);
	    output.close();
	} catch (IOException e) {
	    e.printStackTrace();
	}

反序列化:
 
	try {
	    ObjectInputStream input = new ObjectInputStream(new FileInputStream("user.bin"));
	    User user = (User) input.readObject();
	    System.out.println(user);	
	} catch (Exception e) {	
	    e.printStackTrace();	
	}

 User类实现了Serializable接口。

###FAQ

####哪些东东会被序列化

序列化只保存状态，不保存行为。在默认的序列化实现中，Java对象中的非静态和非瞬时域都会被序列化。

####如何控制序列化字段

1、把域声明为瞬时的，即使用transient关键词。

2、添加一个serialPersistentFields域来声明序列化时要包含的域。


	private static final ObjectStreamField[] serialPersistentFields = { 
	    new ObjectStreamField("firstName", String.class) 
	};  
上面的代码给出了 serialPersistentFields的声明示例，即只有firstName这个域是要被序列化的。

####为什么反序列化时不成功

虚拟机是否允许反序列化，不仅取决于类路径和功能代码是否一致，一个非常重要的一点是两个类的序列化 ID 是否一致（就是 private static final long serialVersionUID = 1L）。

如果类没有serialVersionUID静态字段，jvm在反序列化时会根据field情况生成serialVersionUID，类字段增减时，就会出现版本不一致的问题，最好添加一个默认的版本，当然也可以通过这种方式来控制类的不同版本。

####实现了Serializable接口为什么还是序列化不成功？

如果类中的域包含不可序列化的对象，序列化就会失败，抛出NotSerializableException。

####父类实现了Serializable接口，子类不想序列化怎么办？

子类可以重载writeObject () 或者 readObject ()方法，在方法体中throw NotSerializableException 

####什么是compatible changes and incompatible changes

增减字段或者方法属于compatible changes，只要serialVersionUID相同，还可以成功的反序列化。但是如果修改类层次，或者不实现Serializable接口，就属于incompatible changes。参考java 序列化规范

####反序列化后的对象和new 的对象有什么区别

在通过ObjectInputStream的readObject方法读取到一个对象之后，这个对象是一个新的实例，但是其构造方法是没有被调用的，其中的域的初始化代码也没有被执行。对于那些没有被序列化的域，在新创建出来的对象中的值都是默认的。也就是说，这个对象从某种角度上来说是不完备的。这有可能会造成一些隐含的错误。调用者并不知道对象是通过一般的new操作符来创建的，还是通过反序列化所得到的。解决的办法就是在类的readObject方法里面，再执行所需的对象初始化逻辑。对于一般的Java类来说，构造方法中包含了初始化的逻辑。可以把这些逻辑提取到一个方法中，在readObject方法中调用此方法。

####序列化安全性

Java对象序列化之后的内容格式是公开的。所以可以很容易的从中提取出各种信息。从实现的角度来说，可以从不同的层次来加强序列化的安全性。

对序列化之后的流进行加密。这可以通过CipherOutputStream来实现。
实现自己的writeObject和readObject方法，在调用defaultWriteObject之前，先对要序列化的域的值进行加密处理。
使用一个SignedObject或SealedObject来封装当前对象，用SignedObject或SealedObject进行序列化。
在从流中进行反序列化的时候，可以通过ObjectInputStream的registerValidation方法添加ObjectInputValidation接口的实现，用来验证反序列化之后得到的对象是否合法。
 

####自定义对象序列化
#####Serializable接口

`Serializable`接口是一个标记接口，接口内没有任何方法，但是在实际的序列化和反序列化过程中，jvm通过一些契约规则来满足用户自定义需求。

基本的对象序列化机制让开发人员可以在包含哪些域上进行定制。如果想对序列化的过程进行更加细粒度的控制，就需要在类中添加writeObject和对应的 readObject方法。这两个方法属于前面提到的序列化机制的隐含契约的一部分。在通过ObjectOutputStream的 writeObject方法写入对象的时候，如果这个对象的类中定义了writeObject方法，就会调用该方法，并把当前 ObjectOutputStream对象作为参数传递进去。writeObject方法中一般会包含自定义的序列化逻辑，比如在写入之前修改域的值，或是写入额外的数据等。对于writeObject中添加的逻辑，在对应的readObject中都需要反转过来，与之对应。

在添加自己的逻辑之前，推荐的做法是先调用Java的默认实现。在writeObject方法中通过ObjectOutputStream的defaultWriteObject来完成，在readObject方法则通过ObjectInputStream的defaultReadObject来实现。下面的代码在对象的序列化流中写入了一个额外的字符串。

	
	private void writeObject(ObjectOutputStream output) throws IOException {
	    output.defaultWriteObject();
	    output.writeUTF("Hello World");
	}
	
	private void readObject(ObjectInputStream input) throws IOException, ClassNotFoundException {
	    input.defaultReadObject();
	    String value = input.readUTF();	
	    System.out.println(value);
	
	} 
 
#####Externalizable接口

`Externalizable` 是一个包含方法签名的接口,包括writeExternal和readExternal: 

	 public void readExternal(ObjectInput arg0) throws IOException,  
	            ClassNotFoundException {  
	        Object obj = arg0.readObject();       
	 }  
	
	 public void writeExternal(ObjectOutput arg0) throws IOException {   
	        arg0.writeObject("Hello world");  
	 }  


参考:
[http://www.infoq.com/cn/articles/cf-java-object-serialization-rmi](http://www.infoq.com/cn/articles/cf-java-object-serialization-rmi)
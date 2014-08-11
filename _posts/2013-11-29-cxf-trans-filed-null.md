---
layout: post
title: "cxf如何传字段为null"
image:
  feature: abstract-10.jpg

tags: [sample post, images, test]
comments: true
share: true
---


传输对象如下：

	public class EmailSenderOrder {
	private String clientName="test";
	//clientName默认值为test
	}

应用场景需要传输的`clientName`为null，我们在使用client时，会有如下代码：

	EmailSenderOrder emailSenderOrder = new EmailSenderOrder(); 
	emailSenderOrder.setClientName(null); 

服务端在接受到对象后，clientName结果等于了test。
<!--more-->
默认情况下，如果field为null，cxf client生成的报文中没有这个field的报文，这就导致在反序列化时，初始化对象时用了默认值。

可以如下解决：

	//通过字段序列化
	@XmlAccessorType(XmlAccessType.FIELD) 

	public class EmailSenderOrder {

	//如果clientName为null，生成报文<clientName xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:nil="true"/>
	@XmlElement(name = "clientName", nillable = true) 
	private String clientName="test";
	
	}
---
layout: post
title: "java对象属性复制"
image:
  feature: abstract-10.jpg
tags: [sample post, images, test]
comments: true
share: true
---
java对象属性复制的工具类很多,常用的有`org.springframework.beans.BeanUtils#copyProperties`,`org.apache.commons.beanutils.BeanUtils#copyProperties`,这两个都是通过反射实现的,性能嘛,你关心TA就在那里,你不关心TA还是在那里.
<!--more-->
高级点的有`net.sf.cglib.beans.BeanCopier`,通过生成源代码实现属性复制,但是他的api很难使用.而且不支持基本类型和包装器类型的转换(java的boxing和unboxing只是语法糖而已).

so,重新造了个轮子,采用javassit来生成源代码,并且提供方便使用的api.

使用就像下面这样:

	TestBean target = new TestBean();
	Copier.copy(TestBean1.createTest(), target);
	
TA生成的源代码如下:

	public class CopierImpl1002
 	 implements Copier.Copy
	{
	  public void copy(Object paramObject1, Object paramObject2)
	  {
	    TestBean localTestBean = (TestBean)paramObject1;
    	TestBean1 localTestBean1 = (TestBean1)paramObject2;
    	if (localTestBean.getA1() != null)
      		localTestBean1.setA1(localTestBean.getA1().longValue());
    	localTestBean1.setA10(localTestBean.getA10());
 	   	localTestBean1.setA2(localTestBean.isA2());
 	  	localTestBean1.setA3(Integer.valueOf(localTestBean.getA3()));
   		localTestBean1.setB8(Short.valueOf(localTestBean.getB8()));
  	  	localTestBean1.setList(localTestBean.getList());
 	 }
	}
	
源代码见:[https://gist.github.com/bohrqiu/5046a2a7d983996f0e5a](https://gist.github.com/bohrqiu/5046a2a7d983996f0e5a)
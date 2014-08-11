---
layout: post
title: "hessian反序列化field值丢失bug"
image:
  feature: abstract-10.jpg

tags: [sample post, images, test]
comments: true
share: true
---

###bug说明
---
子类和父类中有同名的字段时，hessian反序列化会丢失字段值。
<!--more-->
父类`Person`：

	public class Person implements Serializable {
	    private static final long serialVersionUID = 1L;
	    private String name;
	    private int age;
	    public String getName() {
	        return name;
	    }
	    public void setName(String name) {
	        this.name = name;
	    }
	    public int getAge() {
	        return age;
	    }
	    public void setAge(int age) {
	        this.age = age;
	    }
	    @Override
	    public String toString() {
	        final StringBuilder sb = new StringBuilder("Person{");
	        sb.append("name='").append(name).append('\'');
	        sb.append(", age=").append(age);
	        sb.append('}');
	        return sb.toString();
	    }
	}

子类`PersonEx`：

	public class PersonEx  extends  Person{
	    private static final long	serialVersionUID	= 1L;
	    private String	name;
	    public String getName() {
	        return name;
	    }
	    public void setName(String name) {
	        this.name = name;
	    }
	    @Override
	    public String toString() {
	        final StringBuilder sb = new StringBuilder("PersonEx{");
	        sb.append("name='").append(name).append('\'');
	        sb.append('}');
	        sb.append(super.toString());
	        return sb.toString();
	    }
	}

当我们序列化如下对象后

	  PersonEx p = new PersonEx();
      p.setName("XXX");
      p.setAge(28);
后，反序列化回来的`PersonEx`对象`getName()==null`.

###原因分析
---

造成这个问题的原因如下：
hessian序列化机制，他通过反射把所有字段名和值(整个传输对象图)读取出来,见`JavaSerializer#writeObject`:

	   	if (ref == -1) {
		//写入需要反射的字段名
	      writeDefinition20(out);
	      out.writeObjectBegin(cl.getName());
	   	}
		//写入值
		writeInstance(obj, out);

按照此规则，现在序列化结构如下(忽略其他映射关系)：
	
	name|age|name|Person|XXX|28|null

反序列化时，读取字段名，在set字段值：

1. 读取到name字段，调用`setName("XXX")`
2. 再次读取name字段，调用`setName(null)`

结果就囧了。。。。

###解决办法
--- 

####1.修改hessian代码

修改了hessian的序列化机制，当类的父对象出现相同的字段名时，跳过对此字段的处理。

	 //添加成员变量propertiesNameSet	 
   	 private HashSet<String> propertiesNameSet = Sets.newHashSet();
	 //在JavaSerializer构造器里，遍历字段时，添加检查
	 if (propertiesNameSet.contains(field.getName())) {
           continue;
     } else {
          propertiesNameSet.add(field.getName());
     }
	

hessian在反序列化时有同名字段的判断`com.caucho.hessian.io.JavaDeserializer#getFieldMap`保证`fieldMap`中不出现同名的字段。

####2.修改同名字段问题

出现了同名字段，也可以认为是我们代码设计出问题了，没有理解清楚继承结构。在子类中删除同名的字段，就避免这个bug。

###小结
现在的序列化框架基本上都会遇到此问题,`Kryo`中的`com.esotericsoftware.kryo.serializers.CompatibleFieldSerializer`也没有对此问题进行处理.作为框架开发者,要尽量减少框架使用者犯错误的机会,也就是所谓的防御性编程吧.

给`Kryo` [pull request](https://github.com/EsotericSoftware/kryo/pull/187) ,防止因为子类和父类中有同名field name而导致的问题.
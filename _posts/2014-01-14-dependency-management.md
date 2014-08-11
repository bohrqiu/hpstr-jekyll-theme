---
layout: post
title: "依赖管理"
image:
  feature: abstract-10.jpg
tags: [sample post, images, test]
comments: true
share: true
---

#依赖管理
项目依赖问题很让人头痛，也出了很多事故，线上线下都很闹心。本着让大家都能开心的原则😄，在**@培根**童鞋的要求下，刚好我对maven很熟，刚好我对公司遇到的依赖问题也很了解，写了此文档。
<!--more-->
请您带着一个大大的问号**?**阅读此文档.您可以把您遇到的依赖问题(此文档没有包含的)发给我,丰富案例库.您可以补充依赖检查中的不足之处,毕竟我个人的能力有限.您可以分享一些灰常有用的插件,方便大家.您可以谈谈您对依赖风险控制的想法.**Help Me Help You!**
##一.依赖管理目标
1. 此规范更新后,使用方不需要修改代码
3. 规范开源jar包版本
4. 检查传递依赖
5. 检查项目classpath中是否有类名相同的依赖jar包
6. 检查已知不能使用的jar包
7. 定义常用插件
8. 优化yjf-common-util依赖,不是每个项目都使用的包定义为provided,由项目自己引入

##二.如何实现目标
###1.创建公共父pom

* 定义所有易极付项目都依赖的父pom `com.yiji.yiji-parent`
* 此pom为SNAPSHOT
* 此pom在`dependencyManagement`中定义常用jar包的版本
* 此pom显示依赖`yjf-common-util` `guava` `log`
* 此pom使用`maven-enforcer-plugin`来规范传递依赖,要求当前依赖的版本和传递依赖版本一样或者比传递依赖版本高(比如A->Cv1 ,A->D->Cv2,如果`v1<v2`,则打包失败)
* 此pom使用`maven-enforcer-plugin`来规范引入会导致已知问题的包(比如我们的项目都使用`slf4j`和`logback`,那我们的依赖中不能出现`org.slf4j:slf4j-log4j12`)
* 此pom使用`com.yiji.maven.yiji-maven-plugin`来检查classpath中是否有类名相同的jar包出现.如果有,在`console`中也会提示警告,在执行打包命令的目录生成`dependency-check.log`文件,此文件中会记录检查了哪些包.同时,以后我们也可以通过此日志文件来了解我们项目间的依赖情况.
* 此pom包含常用maven插件`maven-compiler-plugin` `maven-source-plugin` `maven-eclipse-plugin` `findbugs-maven-plugin` `maven-pmd-plugin`方便大家日常使用
* 此pom中的开源依赖,我会定期check是否有更新,是否有bug修复

###2.开发`com.yiji.maven.yiji-maven-plugin`
此插件已开发完毕,代码也很简单,可以发现一些类加载顺序不一致导致的潜在的问题.

目前此插件只检查不同的jar包中是否有相同的类名.还可以增加对资源文件的检查,文件名相同还可以增加对内容的检查.这些需求如有必要,以后在加上.此插件也是SNAPSHOT的,以后我升级了,大家不用改动任何代码.
###3.定制`settings.xml`

* 此文件定义`snapshot`依赖为每次打包检查
* 其他大多人不需要关心的东东


##三.如何实施
目前已完成cs项目的改造,使用上面的东东,cs的pom文件还廋身不少.

由于传递依赖,不敢贸然大规模推广,先选择被依赖较少的项目使用.和**@培根**商量,先选择**boss项目\易融通项目\易房保项目**使用,使用过程中出现任何问题,请联系我(也可以顺带请我喝茶)

如果这几个项目把雷踩完了,需要找一个统一的时间点,大家一起修改\测试\上线
##四.如何搞
###1.替换`setting.xml`
下载`svn://192.168.45.206/common/yiji-parent/settings.xml`,替换maven安装目录中的`setting.xml`
###2.配置项目父pom
拿cs为例，在cs的父pom中加入

	<parent>
        <groupId>com.yiji</groupId>
        <artifactId>yiji-parent</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>
去掉dependencyManagement中的开源jar依赖(公司内部的依赖不要去掉， `com.yiji.yiji-parent`中没有定义这些东东)。检查项目中的开源依赖是否有版本号，如果有并且IDE提示重复的定义，去掉此版本号；如果没有提示，应该是在 `com.yiji.yiji-parent`里没有加入此依赖，请联系我。
###3.测试打包
执行`mvnp`试试，如果有传递依赖问题，打包会失败，请先联系我。如果打包成功，请检查`dependency-check.log`文件中有没有警告
###4.测试项目
运行单元测试用例，看会不会出现问题，最好找**@翼德**同学全量回归下。

##五.FAQ
###1.为什么不提供方便发布的东东

#####原因有：


- 容易出错，命令很简单，容易把发往生产的包发到测试环境，除非你很了解
- setting.xml里不支持定义`distributionManagement`，只能在pom里面定义。因为我们有多套nexus，需要通过profile来定义不同环境对应不同的nexus，但是profile不能继承(我测试是这样的)
- 我们大多数项目，只能把facade发布到nexus，即便profile支持继承，我也不敢把这玩意儿加到`com.yiji.yiji-parent`中。万一某童鞋在项目根目录执行了mvn deploy，**@培根**会不开心的...😠

#####如何解决：

可以参考`com.yiji.yiji-parent`中的profiles部分，请在facade中定义这个玩意儿，然后执行命令

###2.有哪些常用的mvn命令，可以方便大家使用

非window用户，请在~/.bash_profile中加入

		alias mvni='mvn -T 1C clean install -Dmaven.test.skip=true'
		alias mvnp='mvn -T 1C clean package -Dmaven.test.skip=true'
		alias mvnv='mvn versions:set -DgenerateBackupPoms=false'
		alias mvnd='mvn -T 1C clean deploy -Dmaven.test.skip=true'
		alias mvndd='mvn -T 1C clean deploy -P dev  -Dmaven.test.skip=true'
		alias mvndo='mvn -T 1C clean deploy -P online -Dmaven.test.skip=true'
		alias mvnc='mvn -T 1C clean eclipse:clean idea:clean'
		alias mvne='mvn -T 1C clean eclipse:clean eclipse:eclipse  -DdownloadSources=true'
不知道各命令啥意思的童鞋请google.

window用户请把`svn://192.168.45.206/common/yiji-parent/script`中的脚本添加到PATH中，have a try.

###3.为毛不加入自动生成doc文档插件

`yjf-common-util`里面用`maven-javadoc-plugin`来生成doc文档并在发布时上传到nexus。不过这很费时，而且我们以前使用的`codetemplates`里面有很多javadoc不认识的东东，警告一大堆，看着惨不忍睹。

###4.IDE里面安装maven插件有什么好处
好处很多，它可以检查一些pom编写错误，也可以方便看依赖树。eclipse对maven支持很牛，依赖树看着会很爽，简单的依赖问题，用它就可以搞定。IDEA的智能提示很牛，添加依赖快捷键就可以搞定。

以后有依赖问题的童鞋，先用IDE提供的依赖树功能发现问题。找我也可以，但别让我给你安装maven插件，生命是短暂的啊。
###5.maven我不熟怎么办
肯定不是凉拌，可以先看看`http://www.infoq.com/cn/minibooks/maven-in-action` `http://www.juvenxu.com/category/maven/`
后面我会给大家分享maven一些基本的东东。
###6.依赖问题除了maven导致的,还有其他导致的,如何解决?
主要是把场景找出来,然后分析这些问题,我们自己来添加些防御手段.比如今天**@周洋**同学遇到的两个spring bean id配置一样了,导致本地开发测试ok,199启动确不正常.我们可以给spring添加点东东来检查重复id的问题.
###7.我依赖的某jar版本就是不一样
有这样的的需求,现在问题比较突出的应该是金融这块.金融依赖很多银行提供的jar包,这些包可能会冲突.比如金融某项目,既依赖`httpclient3`又依赖`httpclient4`.

这种情况只能在项目里面指定版本了,使用`com.yiji.maven.yiji-maven-plugin`里提供的版本不是强制约束.但是建议大家都别这样做,除非没办法.
###8.spring4.0都发布了,我们是不是该升级了.
我们现在用的是spring 3.1,可能有项目用的spring3.2.

[spring3.2](http://docs.spring.io/spring/docs/3.2.6.RELEASE/spring-framework-reference/htmlsingle/#new-in-3.2) 有很多新特性,比如test framework,此次升级也把spring升级到3.2.6.

[spring 4.0](http://docs.spring.io/spring/docs/4.0.0.RELEASE/spring-framework-reference/htmlsingle/#spring-whats-new)改动太大,暂时不考虑

###9.在公共父pom中升级一个版本,风险怎么把控
以后大家都继承此父pom,升级一个版本意味着大家都升级了.风险确实很大.

首先,我们会去分析此依赖的`release notes`,评估升级的必要性和影响面.

然后,我们会找bops这样的大杂烩项目来做测试,测试相关特性是否受影响.  
###10.为什么不把`setting.xml`的配置移到pom中
这样做的目的是为了做到环境感知,不同环境的maven `setting.xml`会不一样,这是信息中心和我需要做的事情.对于大家,只需要使用svn repos中的`setting.xml`.放到pom中,这个pom需要定义不同的profile,还需要修改我们现有的各个环境的打包脚本.

###11.常见踩雷问题
####11.1 java.lang.NoClassDefFoundError: org/springframework/ui/velocity/VelocityEngineFactory
这个原因是spring把相关的类放到了spring-context-support里面.如果你用spring的声明式cache,也会遇到找不到类,都加入下面的依赖.

	<dependency>
		<groupId>org.springframework</groupId>
		<artifactId>spring-context-support</artifactId>
	</dependency>
####11.2 velocity报错

	java.lang.IllegalStateException: Cannot convert value of type 	[org.springframework.web.servlet.view.velocity.VelocityConfigurer] to required type [org.apache.velocity.app.VelocityEngine] for property 'velocityEngine': no matching editors or conversion strategy found
	at org.springframework.beans.TypeConverterDelegate.convertIfNecessary(TypeConverterDelegate.java:267)

报错是因为配置的bean `org.springframework.web.servlet.view.velocity.VelocityConfigurer` id为`velocityEngine`.此id覆盖了默认的velocityEngine,把这个id改为`velocityConfigurer`就ok
####11.3  net.sf.ehcache.util.UpdateChecker - Update check failed: 
关闭ehcache启动时检查版本,在ehcache配置根元素上添加属性`updateCheck="false"` 
####11.4 `BeanCopier`报错
'net.sf.cglib.core.TypeUtils.parseType(Ljava/lang/String;)Lorg/objectweb/asm/Type;'
这是以为spring 3.2对asm有改动[Migrating to Spring Framework 3.2](http://docs.spring.io/spring/docs/3.2.6.RELEASE/spring-framework-reference/htmlsingle/#migration-3.2)(D.3和D.4讲了这些东东),咱也得跟着改动,遇到这个错误,去掉cglib-nodep的依赖就ok

加入下面的依赖:
	
	<dependency>
        <groupId>cglib</groupId>
        <artifactId>cglib</artifactId>
    </dependency>
    <dependency>
        <groupId>org.ow2.asm</groupId>
        <artifactId>asm-util</artifactId>
    </dependency>

####11.5 `Spring MVC controller` 处理ajax请求报错 `406 Not Acceptable`
`spring 3.2`引入了内容协商的概念,此概念很REST,资源输出格式由客户端来定义.目前支持三种:

* 请求后缀名
比如getUser.html getUser.xml  getUser.json 分别代表请求输出为html/xml/json
* 参数
比如请求为getUser?type=xml getUser?type=json
* http header 
在http请求中设置`Accept` header,由客户端编程定义接收什么格式的返回.

这三种方式,我个人觉得第三种最优雅,很适合编程实现对资源的访问.第一种很直观,第二种有点破坏REST的味道了.参考[Content Negotiation using Spring MVC](http://spring.io/blog/2013/05/11/content-negotiation-using-spring-mvc)

遇到这个错误,很可能是因为我们在`Controller`中定义`@RequestMapping`的`value`带有html后缀,但是我们在方法上也加上了`@ResponseBody`,这让spring很困惑,你请求为html,返回输出又要去解析为json.

**配置:**
1.引入正确的schema
把schemaLocation中spring mvc schema版本去掉

		http://www.springframework.org/schema/mvc
	      http://www.springframework.org/schema/mvc/spring-mvc.xsd

2.配置`contentNegotiationManager`
	
	<bean id="contentNegotiationManager"
          class="org.springframework.web.accept.ContentNegotiationManagerFactoryBean">
        <property name="favorPathExtension" value="false"/>
        <property name="favorParameter" value="false"/>
        <property name="ignoreAcceptHeader" value="true"/>
        <property name="useJaf" value="false"/>
        <property name="defaultContentType" value="application/json"/>
    </bean>
    
3.配置json转换器

	<mvc:annotation-driven
             content-negotiation-manager="contentNegotiationManager">
        <mvc:message-converters register-defaults="true">
            <bean id="fastJsonHttpMessageConverter"
                  class="com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter">
                <property name="supportedMediaTypes">
                    <list>
                        <value>application/json;charset=UTF-8</value>
                    </list>
                </property>
                <property name="features" value="WriteDateUseDateFormat"/>
            </bean>
        </mvc:message-converters>
    </mvc:annotation-driven>
    
我们很多童鞋spring mvc用得很不地道,建议看看官方demo [mvc-showcase-screencast](http://s3.springsource.org/MVC/mvc-showcase-screencast.mov)

####11.6 使用`com.yjf.common.web.CrossScriptingFilter`报找不到ESAPI
添加依赖.

	<dependency>
    	<groupId>org.owasp.esapi</groupId>
    	<artifactId>esapi</artifactId>
		<exclusions>
			<exclusion>
            	<artifactId>log4j</artifactId>
            	<groupId>log4j</groupId>
        	</exclusion>
        	<exclusion>
        		<groupId>xerces</groupId>
            	<artifactId>xercesImpl</artifactId>
        	</exclusion>
         </exclusions>
    </dependency>
    <dependency>
    	<groupId>xerces</groupId>
        <artifactId>xercesImpl</artifactId>
    </dependency>


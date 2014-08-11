---
layout: post
title: "使用jdk7新增异常构造器提高异常性能"
image:
  feature: abstract-10.jpg

tags: [sample post, images, test]
comments: true
share: true
---

JDK7在Throwable对象中增加了一个新的构造器

	protected Throwable(String message, Throwable cause,
                        boolean enableSuppression,
                        boolean writableStackTrace)
第三个参数表示是否启用suppressedExceptions(try代码快中抛出异常，在finally中又抛出异常，导致try中的异常丢失)
。第四个参数表示是否填充异常栈，如果为false，异常在初始化的时候不会调用本地方法fillInStackTrace。
<!--more-->
在业务开发中，我们会使用很多异常来表示不同的业务限制，比如用户余额不足、用户权限不够、参数不合法，这样的异常是不需要填充栈信息的，所以我们可以使用如下的代码来提高异常生成性能：

	package com.yjf.common.exception;	
	public class AppExcetpion extends RuntimeException {
		private static final long serialVersionUID = 1L;		
		public AppExcetpion() {
			this(null, null);
		}
		public AppExcetpion(String message, Throwable cause) {
			super(message, cause, true, false);
		}
		
		public AppExcetpion(String message) {
			this(message, null);
		}
		
		public AppExcetpion(Throwable cause) {
			this(null, cause);
		}
	
	}

使用此异常会比以前我们使用的异常性能提高10多倍。

	2013-05-09 14:58:39 [main] ERROR com.yjf.common.exception.JAVA7ExceptionTest.testJava7:23 - 运行:2000000/2010000 耗时:457ms
	2013-05-09 14:58:45 [main] ERROR com.yjf.common.exception.JAVA7ExceptionTest.testOri:40 - 运行:2000000/2010000 耗时:6442ms
	
ps:没用使用jdk7或者某些场景下不能使用新特性的,可以考虑重写`fillInStackTrace`方法:

	public class OrderCheckException extends IllegalArgumentException {
	
		private static final long serialVersionUID = 1L;
	
		private Map<String, String> errorMap = new HashMap<>();
	
		private String msg;
	
		public OrderCheckException() {
			super();
		}
	
		public OrderCheckException(Throwable cause) {
			super(cause);
		}
	
		public Map<String, String> getErrorMap() {
			return errorMap;
		}
	
		/**
		 * 增加参数错误信息
		 * 
		 * @param parameter 校验失败参数
		 * @param msg 参数信息
		 */
		public void addError(String parameter, String msg) {
			this.errorMap.put(parameter, msg);
			this.msg = null;
		}
	
		@Override
		public String getMessage() {
			if (msg == null) {
				if (errorMap.isEmpty()) {
					msg = "";
				} else {
					StringBuilder sb = new StringBuilder();
					for (Map.Entry entry : errorMap.entrySet()) {
						sb.append(entry.getKey()).append(SplitConstants.SEPARATOR_CHAR_COLON)
						.append(entry.getValue()).append(SplitConstants.SEPARATOR_CHAR_COMMA);
					}
					msg = sb.toString();
				}
			}
			return msg;
		}
	
		@Override
		public synchronized Throwable fillInStackTrace() {
			return this;
		}
  	}
  	
 上面这个`OrderCheckException`,它继承了`IllegalArgumentException`,但是此异常没有重写`java.lang.RuntimeException#RuntimeException(java.lang.String, java.lang.Throwable, boolean, boolean)`,只有通过重写`fillInStackTrace`来达到效果.
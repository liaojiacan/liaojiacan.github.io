title: javaagent 的使用
author: Jiacan Liao
tags:
  - java
  - java逆向
categories:
  - 逆向
date: 2017-10-10 11:40:00
---

>  javaagent 是类似一个JVM的插件，利用JVM提供的Instrumentation API实现获取或者修改加载到JVM中的类字节码。

编写一个javagent的jar的方式如下：

1.实现一个ClassFileTransformer

```
public class SimpleTransformer implements ClassFileTransformer {


	@Override
	public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
		System.out.println(className);
		System.out.println(protectionDomain.toString());
		return new byte[0];
	}
}
```

2.实现一个Premain-Class

```

public class Main {
	public static void premain(String agentOps, Instrumentation inst) {
		inst.addTransformer(new SimpleTransformer());
	}

	public static void main(String[] args) {
		System.out.println("This is a javaagent!");
	}
}

```

3.MANIFEST.MF配置

```
Manifest-Version: 1.0
Premain-Class: com.github.liaojiacan.Main
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Can-Set-Native-Method-Prefix: true
```

4.运行命令

```
java -javaagent:agent.jar -jar app.jar
```

代码和assembly的打包配置可以参考，[github](https://github.com/liaojiacan/code-snippets/tree/master/javaagent)
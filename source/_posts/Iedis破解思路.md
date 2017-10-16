title: Iedis破解思路
author: Jiacan Liao
tags:
  - java逆向
categories:
  - 逆向
date: 2017-10-12 15:01:00
---

> Iedis是IDEA上的一个收费redis插件，java编写的，既然是java写的，收费这些自然是很容易绕过的。由于Java太容易被反编译，作者还是做了些代码混淆和字符串加密。
> 

### 利用JD-GUI反编译出来的代码片段

可以看出类名和字符串都被混淆和加密了，从代码上很难去定位和分析他的注册流程。
```
package com.seventh7.widget.iedis.config;

import com.intellij.icons.AllIcons.General;
import com.intellij.openapi.actionSystem.AnActionEvent;
import com.intellij.openapi.ui.Messages;

class i
  extends e
{
  private static final String[] ib;
  private static final String[] jb;
  
  i(n paramn)
  {
    super(a(20539, 52876), a(20539, 52876), AllIcons.General.Remove, a(20537, 55341), paramn);
  }
  
  void a(AnActionEvent paramAnActionEvent, P paramP)
  {
    String str1 = String.format(a(20538, 25199), new Object[] { paramP.f() });
    String str2 = a(20536, 20012);
    try
    {
      if (Messages.showOkCancelDialog(a(), str1, str2, Messages.getQuestionIcon()) == 0) {
        com.seventh7.widget.iedis.d.e.a().a(a(), paramP.b());
      }
    }
    catch (RuntimeException localRuntimeException)
    {
      throw d(localRuntimeException);
    }
  }
  
```
### 破解的2个思路
- 还原代码中的所有加密字符串，根据字符串的内定位到相关的代码，利用javassist修改class文件，将文件替换掉原来的文件
- 逆向出他的认证算法，然后做个注册机之类的。iedis是采用服务器认证的，每次启动都要去服务器查询激活，所以注册机不适合。但是我们可以本地架设一个认证服务。

> 架设认证服务器还是比较简单的，下面还是主要研究一下第一种思路。

### 还原字符串
从那些混淆的代码去定位软件的运行逻辑很难下手，但是我们可以换个思路，将软件运行过程中字符串都打印出来，这样我们基本上就可以得到一份软件的运行日志，对java程序进行运行时插入语句看似很麻烦，其实JVM默认就支持javaagent，写个javaagent即可达到效果，javaagent的使用可以参考[《javaagent-的使用》](http://liaojiacan.me/2017/10/10/javaagent-%E7%9A%84%E4%BD%BF%E7%94%A8/)

1. 编写javaagent程序
   
```
/**
 * 给iedis的加密字符串函数 插入打印代码
 */
public class IedisTransformer implements ClassFileTransformer {

	private final static String IDEA_LIB="/Applications/IntelliJ IDEA.app/Contents/lib/*";
	private final static String IDEIS_LIB="/Users/liaojiacan/Library/Application Support/IntelliJIdea2017.2/Iedis/lib/*";

	public IedisTransformer() {
		try {
			ClassPool.getDefault().appendClassPath(IDEA_LIB);
			ClassPool.getDefault().appendClassPath(IDEIS_LIB);
		} catch (NotFoundException e) {
			e.printStackTrace();
		}
	}

	@Override
	public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {

		if(className.startsWith("com/seventh7/widget/iedis")){
			try {
				CtClass clazz = ClassPool.getDefault().makeClass(new ByteArrayInputStream(classfileBuffer));
				CtMethod[] methods = clazz.getDeclaredMethods();
				CtClass string = ClassPool.getDefault().getCtClass(String.class.getName());
				for(CtMethod method :methods){

					if(method.getLongName().startsWith("com.seventh7.widget.iedis.a.p.f")){
						System.out.println("Inject :: SUCCESS!");
						method.insertBefore("if(true){return true;} ");
						continue;
					}

					if(method.getReturnType().equals(string)){
						String name = method.getLongName();
						System.out.println("transform the iedis method:"+name);
						method.insertAfter("System.out.println(\"--------------------\");" +
									" System.out.println(\""+name+"\"); " +
									" System.out.println(java.util.Arrays.toString($args)); " +
									" System.out.println(\"return:\"+$_);");
					}
				}

				return clazz.toBytecode();
			} catch (IOException e) {
				e.printStackTrace();
			} catch (NotFoundException e) {
				e.printStackTrace();
			} catch (CannotCompileException e) {
				e.printStackTrace();
			}
		}

		return null;
	}
}
```
2. 修改System.out，把所有的print打印到我们指定的文件中 /tmp/system.out

```
public class Main {
	public static void premain(String agentOps, Instrumentation inst) {
		PrintStream out = null;
		try {
			out = new PrintStream("/tmp/system.out");
		} catch (FileNotFoundException e) {
			e.printStackTrace();
		}
		System.setOut(out);
		System.setErr(out);

		if ("iedis".equals(agentOps)){
			inst.addTransformer(new IedisTransformer());
		}else if("injectPrint".equals(agentOps)) {
			inst.addTransformer(new InjectPrintTransformer());
		}else {
			inst.addTransformer(new SimpleTransformer());
		}
	}

	public static void main(String[] args) {
		System.out.println(helloWorld());
	}

	public static String helloWorld(){
		return "This is a javaagent!";
	}
}

```


3. 配置idea启动配置，加入我们的javaagent

```
#修改idea.vmoptions文件加入下面一行配置
-javaagent:/Users/liaojiacan/Workspace/tools/decomplie/javaagent/javaagent-1.0-SNAPSHOT.jar=iedis
-Xms128m
-Xmx750m
-XX:ReservedCodeCacheSize=240m
-XX:+UseCompressedOops
-Dfile.encoding=UTF-8
-XX:+UseConcMarkSweepGC
-XX:SoftRefLRUPolicyMSPerMB=50
-ea
-Dsun.io.useCanonCaches=false
-Djava.net.preferIPv4Stack=true
-XX:+HeapDumpOnOutOfMemoryError
-XX:-OmitStackTraceInFastThrow
-Xverify:none

-XX:ErrorFile=$USER_HOME/java_error_in_idea_%p.log
-XX:HeapDumpPath=$USER_HOME/java_error_in_idea.hprof
-Xbootclasspath/a:../lib/boot.jar
```
启动Idea 后我们可以在/tmp/system.out中可以看到这些关键的日志
```
--------------------
com.seventh7.widget.iedis.L.a(java.lang.String)
return:{"trailing":true,"daysLeft":9,"popup":true,"activated":false}
--------------------
com.seventh7.widget.iedis.B.a()
return:186b474e0ffffffb70ffffff96680ffffffc0240ffffff89456b0fffffffa320ffffffa70ffffff92
--------------------
com.seventh7.widget.iedis.x.a(byte[])
return:MTg2YjQ3NGUwZmZmZmZmYjcwZmZmZmZmOTY2ODBmZmZmZmZjMDI0MGZmZmZmZjg5NDU2YjBmZmZm
ZmZmYTMyMGZmZmZmZmE3MGZmZmZmZjkyOjI=
--------------------
com.seventh7.widget.iedis.L.a(java.lang.String)
return:{"trailing":true,"daysLeft":9,"popup":true,"activated":false}
--------------------

com.seventh7.widget.iedis.L.a(java.lang.String)
[https://www.codesmagic.com/q2?t=MTg2YjQ3NGUwZmZmZmZmYjcwZmZmZmZmOTY2ODBmZmZmZmZjMDI0MGZmZmZmZjg5NDU2YjBmZmZmZmZmYTMyMGZmZmZmZmE3MGZmZmZmZjkyOjI=]
return:{"trailing":true,"daysLeft":9,"popup":true,"activated":false}
--------------------
com.seventh7.widget.iedis.a.p.b(int,int)
[-13938, -6118]
return:trailing
--------------------
com.seventh7.widget.iedis.a.p.b(int,int)
[-13937, -25088]
return:daysLeft
--------------------
com.seventh7.widget.iedis.a.p.b(int,int)
[-13939, 7216]
return:popup


```
从上面的日志可以看出一些关键点：

- https://www.codesmagic.com/q2?t= 是注册的服务器

- com.seventh7.widget.iedis.a.o 这个类是很关键的类

- 认证服务器返回的认证结果为
 {"trailing":true,"daysLeft":9,"popup":true,"activated":false}


> 查看反编译的代码，可以看出这个类是一个抽象类，他的唯一子类是com.seventh7.widget.iedis.a.p，根据外面获取到的运行日志，大概可以推断出 f这个方法是认证的方法。


```
package com.seventh7.widget.iedis.a;

import com.seventh7.widget.iedis.b.d.a;
import java.util.Map;
import java.io.IOException;
import com.seventh7.widget.iedis.L;

class p extends o
{
    private static final String[] kb;
    private static final String[] lb;
    
    //基本上可以推断出 这个就是认证的方法，最直接的方法就是直接return true
    @Override
    protected boolean f() throws IOException {
        //this.d() 是调用https://www.codesmagic.com/q2去注册的
        //Map的返回值{"trailing":true,"daysLeft":9,"popup":true,"activated":false}
        final Map d = this.d();
        //trailing
        final boolean booleanValue = L.a(d, b(-13938, -6118));
        //this.d()执行后的异常信息。
        final a[] b = av.b();
        //daysLeft
        final int b2 = L.b(d, b(-13937, -25088));
        //popup
        final boolean booleanValue2 = L.a(d, b(-13939, 7216));
        boolean booleanValue3 = false;
        Label_0104: {
            Label_0074: {
                boolean b3;
                try {
                    b3 = (booleanValue3 = booleanValue);
                    if (b != null) {
                        break Label_0104;
                    }
                    if (b3) {
                        break Label_0074;
                    }
                    break Label_0074;
                }
                catch (IOException ex) {
                    throw b(ex);
                }
                try {
                    if (b3) {
                        this.a(b2, booleanValue2);
                        return false;
                    }
                }
                catch (IOException ex2) {
                    throw b(ex2);
                }
            }
            //actived
            booleanValue3 = L.a(d, b(-13940, 8507));
        }
        final boolean b4 = booleanValue3;
        
        //如果已经过了试用，就检测激活
        Label_0122: {
            boolean b5;
            try {
                final boolean b6;
                b5 = (b6 = b4);
                // b = av.b()
                if (b != null) {
                    return b6;
                }
                if (!b5) {
                    break Label_0122;
                }
                return true;
            }
            catch (IOException ex3) {
                throw b(ex3);
            }
            try {
                if (!b5) {
                    this.c();
                    return false;
                }
            }
            catch (IOException ex4) {
                throw b(ex4);
            }
        }
        return true;
    }
    
    private static IOException b(final IOException ex) {
        return ex;
    }

  
}

```

从上面的分享结果可以看出，有两种破解思路

- 方法一 修改 com.seventh7.widget.iedis.a.p.f 永远return true

- 方法二 搭建一个认证服务器，本地替换host，认证服务器返回的结果为


```
{ "trailing": false, "popup": true, "activated": true, "daysLeft": 0 }
```

方法一的实现

```
public class IedisCracker {

	private final static String IDEA_LIB="/Applications/IntelliJ IDEA.app/Contents/lib/*";
	private final static String IDEIS_LIB="/Users/liaojiacan/Library/Application Support/IntelliJIdea2017.2/Iedis/lib/*";

	public static void main(String[] args) {
		try {
			ClassPool.getDefault().appendClassPath(IDEA_LIB);
			ClassPool.getDefault().appendClassPath(IDEIS_LIB);

			CtClass clazz = ClassPool.getDefault().getCtClass("com.seventh7.widget.iedis.a.p");

			CtMethod[] mds = clazz.getDeclaredMethods();
			for(CtMethod method : mds){
				if(method.getLongName().startsWith("com.seventh7.widget.iedis.a.p.f")){
					System.out.println("Inject :: SUCCESS!");
					try {
						method.insertBefore("if(true){return true;} ");
					} catch (CannotCompileException e) {
						e.printStackTrace();
					}
					continue;
				}
			}

			clazz.writeFile("/tmp/p.class");

		} catch (NotFoundException e) {
			e.printStackTrace();
		} catch (CannotCompileException e) {
			e.printStackTrace();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
}

```

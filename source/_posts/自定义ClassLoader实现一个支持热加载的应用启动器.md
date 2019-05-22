title: 自定义ClassLoader实现一个支持热加载的应用启动器
author: Jiacan Liao
tags:
  - ClassLoader
  - 热加载
categories:
  - JDK
date: 2019-03-17 16:46:00
---
JVM 默认是不支持Class的热加载的，也就是说我们的代码有变动，就要重启JVM来达到加载新的Class目的，但是很多容器如Tomcat、Jetty等都可以支持热加载，其底层的原理就是自定义ClassLoader。OSGI更是将类加载器玩到极至。我们来看看怎么实现一个简单的支持热加载的应用启动器。

### 一、实现的目标
- 支持热加载
- 可配置的启动类


### 二、实现

#### 1. 支持热加载
关于类的加载，必然要说一下ClassLoader，JDK中存在这几个ClassLoader：
- BootstrapClassLoader 加载基础类
- ExtClassLoader 加载拓展类，父加载器是BootstrapClassLoader
- AppClassLoader 加载应用程序类 ，父加载器是ExtClassLoader

**双亲委派：**
官方建议开发者，实现类加载器时遵循双亲委派规则，就是加载一个类时，先交给父加载器加载，如果父加载器无法加载，再由当前类加载器加载，从代码上来说，AppClassLoader已经写好了这个模版类，我们只需要覆盖findClass的逻辑即可。
> 实现热加载需要违背双亲委派规则吗？

由于ClassLoader中的defineClass方法会对已加载的类进行校验，所以我们无法对一个类进行重复加载，要实现热加载只能创建一个新的ClassLoader，假如我们采用双亲委派规则，那么我们需要加载的类会先被父加载器（AppClassLoader）给加载缓存起来，之后我们无论怎么创建一个新的加载器也无法达到热加载的目的。

```
public class HotSwapClassLoader extends ClassLoader {

	/**
	 * 指定目录下的类可以热加载
	 */
	private String basePath;

	public HotSwapClassLoader(String basePath) {
		this.basePath = basePath;
	}

	@Override
	public Class<?> loadClass(String name) throws ClassNotFoundException {
		Class<?> c = findLoadedClass(name);
		// 加载指定目录下的class
		if (c == null) {
			try {
				c = findClass(name);
				if (c != null) {
					return c;
				}
			} catch (ClassNotFoundException e) {
				return super.loadClass(name);
			}

		}
		return super.loadClass(name);
	}

	@Override
	protected Class<?> findClass(String name) throws ClassNotFoundException {

		String classResourcePath = this.basePath + "/" + name.replaceAll("\\.", "/");

		try {
			FileInputStream fileInputStream = new FileInputStream(new File(classResourcePath));
			ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
			int len;
			byte[] buffer = new byte[1024];
			while ((len = fileInputStream.read(buffer)) > 0) {
				byteArrayOutputStream.write(buffer, 0, len);
			}
			byte[] bytes = byteArrayOutputStream.toByteArray();
			return defineClass(name, bytes, 0, bytes.length);
		} catch (IOException e) {
			throw new ClassNotFoundException(name);
		}

	}
}
```

#### 2. 启动器

上面我们已经实现了一个可以随时替换的ClassLoader，我们还需要一个引导类去维护我们的ClassLoader 还有我们的应用启动入口，管理启动和关闭的时机，就好比Tomcat的Catalina一样，或者说我们的任何类的Main函数。

```
public class Bootstrap {

	private String basePath;
	private Object application;
	private String applicationClassName;
	private volatile ClassLoader applicationClassLoader;


	public Bootstrap(String basePath, String applicationClassName) {
		this.basePath = basePath;
		this.applicationClassLoader = new HotSwapClassLoader(this.basePath);
		this.applicationClassName = applicationClassName;
		try {
			this.application = getApplication();
		} catch (Exception e) {
			e.printStackTrace();
		}

	}
    
}
```
有了上面的那些成员，我们就可以利用Java的反射来实现自定义的Application类的启动（这个类可以方在任意位置，就好比我们的war包可以方在任意位置，只要在tomcat的server.xml中配置好baseApps的路径就好了）
```
	private void startApplication() throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
		this.application.getClass().getDeclaredMethod("start", null).invoke(this.application, new Object[0]);
	}

	private void stopApplication() throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
		this.application.getClass().getDeclaredMethod("stop", null).invoke(this.application, new Object[0]);
	}
```
最后，剩下最后一个问题就是，我们怎么知道我们的类需要加载呢？有2种方式就是主动刷新，还有一种就是程序监听文件夹的文件变动。我们可以利用jdk7之后提供的WatchService来监控文件或者目录的变动情况，一发生变动，则先注销之前的Application 然后再创建一个新的HotSwapClassLoader来启动新的Application。

```
private void registerResourceWatcher() {
		try {
			WatchService watchService = FileSystems.getDefault().newWatchService();
			Path p = Paths.get(basePath);
			p.register(watchService, new WatchEvent.Kind[]{ENTRY_MODIFY, ENTRY_CREATE, ENTRY_DELETE});
			while (true) {
				WatchKey k = watchService.take();
				for (WatchEvent<?> e : k.pollEvents()) {
					reloadApplication();
					break;
				}
				k.reset();
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

```

### 三、测试
```
public class Application {

	public Integer version = 46;

	/**
	 * 应用的启动入口
	 */
	public void start() {
		System.out.println("Start... version=" + version);
	}

	/**
	 * 应用的停止入口
	 */
	public void stop() {
		System.out.println("Stop... version=" + version);
	}
}

```

```
public class Main {

	public static void main(String[] args) throws IOException, InterruptedException {

		new Bootstrap("/Users/liaojiacan/Workspace/java/personal/code-snippets/java-language/target/classes"
				,"com.github.liaojiacan.classloader.app.Application").boot();
	}
}
```
启动后，我们修改Application的 version=47，然后rebuild project，这个时候这个文件就会发生改变,输出如下：

```
Start... version=46
Stop... version=46
Start... version=47
```
完整代码见Github :[https://github.com/liaojiacan/code-snippets/tree/master/java-language/src/main/java/com/github/liaojiacan/classloader](https://github.com/liaojiacan/code-snippets/tree/master/java-language/src/main/java/com/github/liaojiacan/classloader)

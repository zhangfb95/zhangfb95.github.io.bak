---
layout: post
title:  深入理解Tomcat（五）类加载机制
date:   2018-11-16 09:51:00 +0800
categories: 深入理解Tomcat
tag: 深入理解Tomcat
---

* content
{:toc}

## 前言

我们知道，Java默认的类加载机制是通过`双亲委派模型`来实现的。而Tomcat实现的方式又和`双亲委派模型`有所区别。原因在于一个Tomcat容器允许同时运行多个Web程序，每个Web程序依赖的类又必须是相互隔离的。因此，如果Tomcat使用`双亲委派模式`来加载类的话，将导致Web程序依赖的类变为共享的。

举个例子，假如我们有两个Web程序，一个依赖A库的1.0版本，另一个依赖A库的2.0版本，他们都使用了类`xxx.xx.Clazz`，其实现的逻辑因类库版本的不同而结构完全不同。那么这两个Web程序的其中一个必然因为加载的Clazz不是所使用的Clazz而出现问题！而这对于开发来说是非常致命的！

怎么解决呢？这将是这篇文章所要探讨的问题！

## 双亲委派模式

Java是一门`面向对象`的语言，而对象又必然依托于`类`。`类`要运行，必须首先被加载到内存。我们可以简单地把类分为几类：

1. Java自带的核心类
2. Java支持的可扩展类
3. 我们自己编写的类

如果所有的类都使用一个类加载器来加载，会出现什么问题呢？

假如我们自己编写一个类`java.util.Object`，它的实现可能有一定的危险性或者隐藏的bug。而我们知道Java自带的核心类里面也有`java.util.Object`，如果JVM启动的时候先行加载的是我们自己编写的`java.util.Object`，那么就有可能出现安全问题！

所以，Sun（后被Oracle收购）采用了另外一种方式来保证最基本的、也是最核心的功能不会被破坏。你猜的没错，那就是`双亲委派模式`！

`双亲委派模式`对类加载器定义了层级，每个类加载器都有一个父类加载器。在一个类需要加载的时候，首先委派给父类加载器来加载，而父类加载器又委派给祖父类加载器来加载，以此类推。如果父类及上面的类加载器都加载不了，那么由当前类加载器来加载，并将被加载的类缓存起来。

我们画一个图来辅助我们理解！

![类加载模型](https://upload-images.jianshu.io/upload_images/845143-25b1b9cf4ccb1095.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. Java自带的核心类 -- `由启动类加载器加载`
2. Java支持的可扩展类 -- `由扩展类加载器加载`
3. 我们自己编写的类 -- 默认`由应用程序类加载器或其子类加载`

双亲委派模型解决了类错乱加载的问题，也设计得非常精妙。但它也不是万能的，在有些场景也会遇到它解决不了的问题。哪些场景呢？我们举一个例子来看看。

在Java核心类里面有SPI（Service Provider Interface），它由Sun编写规范，第三方来负责实现。SPI需要用到第三方实现类。如果使用双亲委派模型，那么第三方实现类也需要放在Java核心类里面才可以，不然的话第三方实现类将不能被加载使用。但是这显然是不合理的！怎么办呢？`ContextClassLoader`（上下文类加载器）就来解围了。

在`java.lang.Thread`里面有两个方法，get/set上下文类加载器

1. `public void setContextClassLoader(ClassLoader cl)`
2. `public ClassLoader getContextClassLoader()`

我们可以通过在SPI类里面调用`getContextClassLoader`来获取第三方实现类的类加载器。由第三方实现类通过调用`setContextClassLoader`来传入自己实现的类加载器。这样就变相地解决了`双亲委派模式`遇到的问题。但是很显然，这种机制破坏了`双亲委派模式`。

## Tomcat类加载机制

既然Tomcat的类加载机器不同于双亲委派模式，那么它又是一种怎样的模式呢？[官网链接-ClassLoading](https://tomcat.apache.org/tomcat-8.5-doc/class-loader-howto.html)有对此进行描述。

另外，我们还是先来看看Tomcat整体的类加载图是怎样的吧~

![Tomcat类加载图](https://upload-images.jianshu.io/upload_images/845143-537b37785d5d72b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们在这张图中看到很多类加载器，除了Jdk自带的类加载器，我们尤其关心Tomcat自身持有的类加载器。仔细一点我们很容易发现：Catalina类加载器和Shared类加载器，他们并不是父子关系，而是兄弟关系。为啥这样设计，我们得分析一下每个类加载器的用途，才能知晓。

1. Common类加载器，负责加载Tomcat和Web应用都复用的类
2. Catalina类加载器，负责加载Tomcat专用的类，而这些被加载的类在Web应用中将不可见
3. Shared类加载器，负责加载Tomcat下所有的Web应用程序都复用的类，而这些被加载的类在Tomcat中将不可见
4. WebApp类加载器，负责加载具体的某个Web应用程序所使用到的类，而这些被加载的类在Tomcat和其他的Web应用程序都将不可见
5. Jsp类加载器，每个jsp页面一个类加载器，不同的jsp页面有不同的类加载器，方便实现jsp页面的热插拔

## CATALINA_HOME和CATALINA_BASE

在tomcat官网上有对他们进行描述，[官网链接-CATALINA_HOME and CATALINA_BASE](https://tomcat.apache.org/tomcat-8.5-doc/introduction.html#CATALINA_HOME_and_CATALINA_BASE)

![官网说明](https://upload-images.jianshu.io/upload_images/845143-03c76e69aa948717.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

简单地说，CATALINA_HOME指的是安装目录，CATALINA_BASE指的是工作目录。在一台物理机上面，可以只有一个安装目录，但是允许有多个工作目录。

## File的路径方法

`java.io.File`有3个方法可以获取路径，为了避免在阅读Tomcat源码的时候出现理解问题，这儿写出demo来说明其含义。

1. getPath()
2. getAbsolutePath()
3. getCanonicalPath()

```java
public class Main {
    public static void main(String[] args) throws Exception {
        System.out.println(System.getProperty("user.dir"));

        System.out.println("-----默认相对路径：取得路径不同------");
        File file1 = new File("../src/test1.txt");
        System.out.println(file1.getPath());
        System.out.println(file1.getAbsolutePath());
        System.out.println(file1.getCanonicalPath());

        System.out.println("-----默认相对路径：取得路径不同------");
        File file = new File("./test1.txt");
        System.out.println(file.getPath());
        System.out.println(file.getAbsolutePath());
        System.out.println(file.getCanonicalPath());

        System.out.println("-----默认绝对路径:取得路径相同------");
        File file2 = new File("/Users/pro/ws/learn/learn-javaagent/test1.txt");
        System.out.println(file2.getPath());
        System.out.println(file2.getAbsolutePath());
        System.out.println(file2.getCanonicalPath());
    }
}
```

输出结果如下：

```
-----默认相对路径：取得路径不同------
../src/test1.txt
/Users/pro/ws/learn/learn-javaagent/../src/test1.txt
/Users/pro/ws/learn/src/test1.txt
-----默认相对路径：取得路径不同------
./test1.txt
/Users/pro/ws/learn/learn-javaagent/./test1.txt
/Users/pro/ws/learn/learn-javaagent/test1.txt
-----默认绝对路径:取得路径相同------
/Users/pro/ws/learn/learn-javaagent/test1.txt
/Users/pro/ws/learn/learn-javaagent/test1.txt
/Users/pro/ws/learn/learn-javaagent/test1.txt
```

## 源码阅读

Tomcat启动的入口在`Bootstrap的main()方法`。`main()`方法执行前，必然先执行其`static{}`块。所以我们首先分析`static{}`块，然后分析`main()`方法

### Bootstrap.static{}

```java
static {
    // 获取用户目录
    // Will always be non-null
    String userDir = System.getProperty("user.dir");

    // 第一步，从环境变量中获取catalina.home，在没有获取到的时候将执行后面的获取操作
    // Home first
    String home = System.getProperty(Globals.CATALINA_HOME_PROP);
    File homeFile = null;

    if (home != null) {
        File f = new File(home);
        try {
            homeFile = f.getCanonicalFile();
        } catch (IOException ioe) {
            homeFile = f.getAbsoluteFile();
        }
    }

    // 第二步，在第一步没获取的时候，从bootstrap.jar所在目录的上一级目录获取
    if (homeFile == null) {
        // First fall-back. See if current directory is a bin directory
        // in a normal Tomcat install
        File bootstrapJar = new File(userDir, "bootstrap.jar");

        if (bootstrapJar.exists()) {
            File f = new File(userDir, "..");
            try {
                homeFile = f.getCanonicalFile();
            } catch (IOException ioe) {
                homeFile = f.getAbsoluteFile();
            }
        }
    }

    // 第三步，第二步中的bootstrap.jar可能不存在，这时我们直接把user.dir作为我们的home目录
    if (homeFile == null) {
        // Second fall-back. Use current directory
        File f = new File(userDir);
        try {
            homeFile = f.getCanonicalFile();
        } catch (IOException ioe) {
            homeFile = f.getAbsoluteFile();
        }
    }

    // 重新设置catalinaHome属性
    catalinaHomeFile = homeFile;
    System.setProperty(
            Globals.CATALINA_HOME_PROP, catalinaHomeFile.getPath());

    // 接下来获取CATALINA_BASE（从系统变量中获取），若不存在，则将CATALINA_BASE保持和CATALINA_HOME相同
    // Then base
    String base = System.getProperty(Globals.CATALINA_BASE_PROP);
    if (base == null) {
        catalinaBaseFile = catalinaHomeFile;
    } else {
        File baseFile = new File(base);
        try {
            baseFile = baseFile.getCanonicalFile();
        } catch (IOException ioe) {
            baseFile = baseFile.getAbsoluteFile();
        }
        catalinaBaseFile = baseFile;
    }
   // 重新设置catalinaBase属性
    System.setProperty(
            Globals.CATALINA_BASE_PROP, catalinaBaseFile.getPath());
}
```

我们把代码中的注释搬下来总结一下：

1. 获取用户目录
2. 第一步，从环境变量中获取catalina.home，在没有获取到的时候将执行后面的获取操作
3. 第二步，在第一步没获取的时候，从bootstrap.jar所在目录的上一级目录获取
4. 第三步，第二步中的bootstrap.jar可能不存在，这时我们直接把user.dir作为我们的home目录
5. 重新设置catalinaHome属性
6. 接下来获取CATALINA_BASE（从系统变量中获取），若不存在，则将CATALINA_BASE保持和CATALINA_HOME相同
7. 重新设置catalinaBase属性

简单总结一下，就是加载并设置catalinaHome和catalinaBase相关的信息，以备后续使用。

### main()

main方法大体分成两块，一块为init，另一块为load+start。

```java
public static void main(String args[]) {
    // 第一块，main方法第一次执行的时候，daemon肯定为null，所以直接new了一个Bootstrap对象，然后执行其init()方法
    if (daemon == null) {
        // Don't set daemon until init() has completed
        Bootstrap bootstrap = new Bootstrap();
        try {
            bootstrap.init();
        } catch (Throwable t) {
            handleThrowable(t);
            t.printStackTrace();
            return;
        }
        // daemon守护对象设置为bootstrap
        daemon = bootstrap;
    } else {
        // When running as a service the call to stop will be on a new
        // thread so make sure the correct class loader is used to prevent
        // a range of class not found exceptions.
        Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
    }

    // 第二块，执行守护对象的load方法和start方法
    try {
        String command = "start";
        if (args.length > 0) {
            command = args[args.length - 1];
        }

        if (command.equals("startd")) {
            args[args.length - 1] = "start";
            daemon.load(args);
            daemon.start();
        } else if (command.equals("stopd")) {
            args[args.length - 1] = "stop";
            daemon.stop();
        } else if (command.equals("start")) {
            daemon.setAwait(true);
            daemon.load(args);
            daemon.start();
            if (null == daemon.getServer()) {
                System.exit(1);
            }
        } else if (command.equals("stop")) {
            daemon.stopServer(args);
        } else if (command.equals("configtest")) {
            daemon.load(args);
            if (null == daemon.getServer()) {
                System.exit(1);
            }
            System.exit(0);
        } else {
            log.warn("Bootstrap: command \"" + command + "\" does not exist.");
        }
    } catch (Throwable t) {
        // Unwrap the Exception for clearer error reporting
        if (t instanceof InvocationTargetException &&
                t.getCause() != null) {
            t = t.getCause();
        }
        handleThrowable(t);
        t.printStackTrace();
        System.exit(1);
    }
}
```

我们点到`init()`里面去看看~

```java
public void init() throws Exception {
    // 非常关键的地方，初始化类加载器s，后面我们会详细具体地分析这个方法
    initClassLoaders();

    // 设置上下文类加载器为catalinaLoader，这个类加载器负责加载Tomcat专用的类
    Thread.currentThread().setContextClassLoader(catalinaLoader);
    // 暂时略过，后面会讲
    SecurityClassLoad.securityClassLoad(catalinaLoader);

    // 使用catalinaLoader加载我们的Catalina类
    // Load our startup class and call its process() method
    if (log.isDebugEnabled())
        log.debug("Loading startup class");
    Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
    Object startupInstance = startupClass.getConstructor().newInstance();

    // 设置Catalina类的parentClassLoader属性为sharedLoader
    // Set the shared extensions class loader
    if (log.isDebugEnabled())
        log.debug("Setting startup class properties");
    String methodName = "setParentClassLoader";
    Class<?> paramTypes[] = new Class[1];
    paramTypes[0] = Class.forName("java.lang.ClassLoader");
    Object paramValues[] = new Object[1];
    paramValues[0] = sharedLoader;
    Method method =
        startupInstance.getClass().getMethod(methodName, paramTypes);
    method.invoke(startupInstance, paramValues);

    // catalina守护对象为刚才使用catalinaLoader加载类、并初始化出来的Catalina对象
    catalinaDaemon = startupInstance;
}
```

关键的方法`initClassLoaders`，这个方法负责初始化Tomcat的类加载器。通过这个方法，我们很容易验证我们上一小节提到的Tomcat类加载图。

```java
private void initClassLoaders() {
    try {
        // 创建commonLoader，如果未创建成果的话，则使用应用程序类加载器作为commonLoader
        commonLoader = createClassLoader("common", null);
        if( commonLoader == null ) {
            // no config file, default to this loader - we might be in a 'single' env.
            commonLoader=this.getClass().getClassLoader();
        }
        // 创建catalinaLoader，父类加载器为commonLoader
        catalinaLoader = createClassLoader("server", commonLoader);
        // 创建sharedLoader，父类加载器为commonLoader
        sharedLoader = createClassLoader("shared", commonLoader);
    } catch (Throwable t) {
        // 如果创建的过程中出现异常了，日志记录完成之后直接系统退出
        handleThrowable(t);
        log.error("Class loader creation threw exception", t);
        System.exit(1);
    }
}
```

所有的类加载器的创建都使用到了方法`createClassLoader`，所以，我们进一步分析一下这个方法。`createClassLoader`用到了CatalinaProperties.getProperty("xxx")方法，这个方法用于从`conf/catalina.properties`文件获取属性值。

```java
private ClassLoader createClassLoader(String name, ClassLoader parent)
    throws Exception {
    // 获取类加载器待加载的位置，如果为空，则不需要加载特定的位置，使用父类加载返回回去。
    String value = CatalinaProperties.getProperty(name + ".loader");
    if ((value == null) || (value.equals("")))
        return parent;
    // 替换属性变量，比如：${catalina.base}、${catalina.home}
    value = replace(value);

    List<Repository> repositories = new ArrayList<>();

   // 解析属性路径变量为仓库路径数组
    String[] repositoryPaths = getPaths(value);

    // 对每个仓库路径进行repositories设置。我们可以把repositories看成一个个待加载的位置对象，可以是一个classes目录，一个jar文件目录等等
    for (String repository : repositoryPaths) {
        // Check for a JAR URL repository
        try {
            @SuppressWarnings("unused")
            URL url = new URL(repository);
            repositories.add(
                    new Repository(repository, RepositoryType.URL));
            continue;
        } catch (MalformedURLException e) {
            // Ignore
        }

        // Local repository
        if (repository.endsWith("*.jar")) {
            repository = repository.substring
                (0, repository.length() - "*.jar".length());
            repositories.add(
                    new Repository(repository, RepositoryType.GLOB));
        } else if (repository.endsWith(".jar")) {
            repositories.add(
                    new Repository(repository, RepositoryType.JAR));
        } else {
            repositories.add(
                    new Repository(repository, RepositoryType.DIR));
        }
    }
    // 使用类加载器工厂创建一个类加载器
    return ClassLoaderFactory.createClassLoader(repositories, parent);
}
```

`createClassLoader`方法里面有几个关键的信息，一个为`Repository`，另一个为`ClassLoaderFactory.createClassLoader`。我们逐一来分析一下。

首先来看Repository，非常简单，就是一个POJO，里面有一个location表示位置，type表示类型。type的枚举有4种：目录、jar集合、jar和url路径。

```java
public enum RepositoryType {
        DIR,
        GLOB,
        JAR,
        URL
    }

    public static class Repository {
        private final String location;
        private final RepositoryType type;

        public Repository(String location, RepositoryType type) {
            this.location = location;
            this.type = type;
        }

        public String getLocation() {
            return location;
        }

        public RepositoryType getType() {
            return type;
        }
    }
```

其次，我们来分析一下`ClassLoaderFactory.createClassLoader`--类加载器工厂创建类加载器。这个方法可谓是非常的长！前方高能~

```java
public static ClassLoader createClassLoader(List<Repository> repositories,
                                            final ClassLoader parent)
    throws Exception {

    if (log.isDebugEnabled())
        log.debug("Creating new class loader");

    // Construct the "class path" for this class loader
    Set<URL> set = new LinkedHashSet<>();
    // 遍历repositories，对每个repository进行类型判断，并生成URL，每个URL我们都要校验其有效性，有效的URL我们会放到URL集合中
    if (repositories != null) {
        for (Repository repository : repositories)  {
            if (repository.getType() == RepositoryType.URL) {
                URL url = buildClassLoaderUrl(repository.getLocation());
                if (log.isDebugEnabled())
                    log.debug("  Including URL " + url);
                set.add(url);
            } else if (repository.getType() == RepositoryType.DIR) {
                File directory = new File(repository.getLocation());
                directory = directory.getCanonicalFile();
                if (!validateFile(directory, RepositoryType.DIR)) {
                    continue;
                }
                URL url = buildClassLoaderUrl(directory);
                if (log.isDebugEnabled())
                    log.debug("  Including directory " + url);
                set.add(url);
            } else if (repository.getType() == RepositoryType.JAR) {
                File file=new File(repository.getLocation());
                file = file.getCanonicalFile();
                if (!validateFile(file, RepositoryType.JAR)) {
                    continue;
                }
                URL url = buildClassLoaderUrl(file);
                if (log.isDebugEnabled())
                    log.debug("  Including jar file " + url);
                set.add(url);
            } else if (repository.getType() == RepositoryType.GLOB) {
                File directory=new File(repository.getLocation());
                directory = directory.getCanonicalFile();
                if (!validateFile(directory, RepositoryType.GLOB)) {
                    continue;
                }
                if (log.isDebugEnabled())
                    log.debug("  Including directory glob "
                        + directory.getAbsolutePath());
                String filenames[] = directory.list();
                if (filenames == null) {
                    continue;
                }
                for (int j = 0; j < filenames.length; j++) {
                    String filename = filenames[j].toLowerCase(Locale.ENGLISH);
                    if (!filename.endsWith(".jar"))
                        continue;
                    File file = new File(directory, filenames[j]);
                    file = file.getCanonicalFile();
                    if (!validateFile(file, RepositoryType.JAR)) {
                        continue;
                    }
                    if (log.isDebugEnabled())
                        log.debug("    Including glob jar file "
                            + file.getAbsolutePath());
                    URL url = buildClassLoaderUrl(file);
                    set.add(url);
                }
            }
        }
    }

    // Construct the class loader itself
    final URL[] array = set.toArray(new URL[set.size()]);
    if (log.isDebugEnabled())
        for (int i = 0; i < array.length; i++) {
            log.debug("  location " + i + " is " + array[i]);
        }

    // 从这儿看，最终所有的类加载器都是URLClassLoader的对象~~
    return AccessController.doPrivileged(
            new PrivilegedAction<URLClassLoader>() {
                @Override
                public URLClassLoader run() {
                    if (parent == null)
                        return new URLClassLoader(array);
                    else
                        return new URLClassLoader(array, parent);
                }
            });
}
```

类加载器工厂方法中有用到`validateFile`方法，这个方法是干嘛的呢？我们分析一下。

```java
private static boolean validateFile(File file,
        RepositoryType type) throws IOException {
    // 对于目录类型或者jar集合目录类型，我们会校验是否为目录和是否可读
    if (RepositoryType.DIR == type || RepositoryType.GLOB == type) {
        if (!file.isDirectory() || !file.canRead()) {
            String msg = "Problem with directory [" + file +
                    "], exists: [" + file.exists() +
                    "], isDirectory: [" + file.isDirectory() +
                    "], canRead: [" + file.canRead() + "]";

            File home = new File (Bootstrap.getCatalinaHome());
            home = home.getCanonicalFile();
            File base = new File (Bootstrap.getCatalinaBase());
            base = base.getCanonicalFile();
            File defaultValue = new File(base, "lib");

            // Existence of ${catalina.base}/lib directory is optional.
            // Hide the warning if Tomcat runs with separate catalina.home
            // and catalina.base and that directory is absent.
            if (!home.getPath().equals(base.getPath())
                    && file.getPath().equals(defaultValue.getPath())
                    && !file.exists()) {
                log.debug(msg);
            } else {
                log.warn(msg);
            }
            return false;
        }
    }
    // 对于JAR，我们会校验文件是否可读
    else if (RepositoryType.JAR == type) {
        if (!file.canRead()) {
            log.warn("Problem with JAR file [" + file +
                    "], exists: [" + file.exists() +
                    "], canRead: [" + file.canRead() + "]");
            return false;
        }
    }
    return true;
}
```

1. 对于目录类型或者jar集合目录类型，我们会校验是否为目录和是否可读
2. 对于JAR，我们会校验文件是否可读

我们已经对initClassLoaders分析完了，接下来分析`SecurityClassLoad.securityClassLoad`，我们看看里面做了什么事情

```java
public static void securityClassLoad(ClassLoader loader) throws Exception {
    securityClassLoad(loader, true);
}

static void securityClassLoad(ClassLoader loader, boolean requireSecurityManager) throws Exception {

    if (requireSecurityManager && System.getSecurityManager() == null) {
        return;
    }

    loadCorePackage(loader);
    loadCoyotePackage(loader);
    loadLoaderPackage(loader);
    loadRealmPackage(loader);
    loadServletsPackage(loader);
    loadSessionPackage(loader);
    loadUtilPackage(loader);
    loadValvesPackage(loader);
    loadJavaxPackage(loader);
    loadConnectorPackage(loader);
    loadTomcatPackage(loader);
}

 private static final void loadCorePackage(ClassLoader loader) throws Exception {
    final String basePackage = "org.apache.catalina.core.";
    loader.loadClass(basePackage + "AccessLogAdapter");
    loader.loadClass(basePackage + "ApplicationContextFacade$PrivilegedExecuteMethod");
    loader.loadClass(basePackage + "ApplicationDispatcher$PrivilegedForward");
    loader.loadClass(basePackage + "ApplicationDispatcher$PrivilegedInclude");
    loader.loadClass(basePackage + "ApplicationPushBuilder");
    loader.loadClass(basePackage + "AsyncContextImpl");
    loader.loadClass(basePackage + "AsyncContextImpl$AsyncRunnable");
    loader.loadClass(basePackage + "AsyncContextImpl$DebugException");
    loader.loadClass(basePackage + "AsyncListenerWrapper");
    loader.loadClass(basePackage + "ContainerBase$PrivilegedAddChild");
    loadAnonymousInnerClasses(loader, basePackage + "DefaultInstanceManager");
    loader.loadClass(basePackage + "DefaultInstanceManager$AnnotationCacheEntry");
    loader.loadClass(basePackage + "DefaultInstanceManager$AnnotationCacheEntryType");
    loader.loadClass(basePackage + "ApplicationHttpRequest$AttributeNamesEnumerator");
}
```

这儿其实就是使用catalinaLoader加载tomcat源代码里面的各个专用类。我们大致罗列一下待加载的类所在的package：

1. org.apache.catalina.core.*
2. org.apache.coyote.*
3. org.apache.catalina.loader.*
4. org.apache.catalina.realm.*
5. org.apache.catalina.servlets.*
6. org.apache.catalina.session.*
7. org.apache.catalina.util.*
8. org.apache.catalina.valves.*
9. javax.servlet.http.Cookie
10. org.apache.catalina.connector.*
11. org.apache.tomcat.*

好了，至此我们已经分析完了init里面涉及到的几个关键方法，真不容易呀~

### load()和start()

我们上面提到main()方法除了初始化方法init，还有load和start。下面我们逐一来分析他们。

```java
private void load(String[] arguments)
    throws Exception {

    // Call the load() method
    String methodName = "load";
    Object param[];
    Class<?> paramTypes[];
    if (arguments==null || arguments.length==0) {
        paramTypes = null;
        param = null;
    } else {
        paramTypes = new Class[1];
        paramTypes[0] = arguments.getClass();
        param = new Object[1];
        param[0] = arguments;
    }
    Method method =
        catalinaDaemon.getClass().getMethod(methodName, paramTypes);
    if (log.isDebugEnabled())
        log.debug("Calling startup class " + method);
    method.invoke(catalinaDaemon, param);
}
```

load()方法只是调用了Catalina的load()方法，只是会根据main方法传入的参数来判断是调用`public void load()`还是`public void load(String args[])`。

```java
public void start() throws Exception {
    if( catalinaDaemon==null ) init();

    Method method = catalinaDaemon.getClass().getMethod("start", (Class [] )null);
    method.invoke(catalinaDaemon, (Object [])null);
}
```

如果Catalina对象不为null，则调用其`init()方法`来初始化，初始化完成之后，调用Catalina对象的`start方法`。

### WebApp类加载器

到这儿，我们隐隐感觉到少分析了点什么！没错，就是WebApp类加载器。整个启动过程分析下来，我们仍然没有看到这个类加载器。它又是在哪儿出现的呢？

我们知道WebApp类加载器是Web应用私有的，而每个Web应用其实算是一个Context，那么我们通过Context的实现类应该可以发现。在Tomcat中，Context的默认实现为`StandardContext`，我们看看这个类的`startInternal()方法`，在这儿我们发现了我们感兴趣的WebApp类加载器。

```java
protected synchronized void startInternal() throws LifecycleException {
    if (getLoader() == null) {
        WebappLoader webappLoader = new WebappLoader(getParentClassLoader());
        webappLoader.setDelegate(getDelegate());
        setLoader(webappLoader);
    }
}
```

入口代码非常简单，就是webappLoader不存在的时候创建一个，并调用`setLoader`方法。我们接着分析`setLoader`

```java
public void setLoader(Loader loader) {

    Lock writeLock = loaderLock.writeLock();
    writeLock.lock();
    Loader oldLoader = null;
    try {
        // Change components if necessary
        oldLoader = this.loader;
        if (oldLoader == loader)
            return;
        this.loader = loader;

        // Stop the old component if necessary
        if (getState().isAvailable() && (oldLoader != null) &&
            (oldLoader instanceof Lifecycle)) {
            try {
                ((Lifecycle) oldLoader).stop();
            } catch (LifecycleException e) {
                log.error("StandardContext.setLoader: stop: ", e);
            }
        }

        // Start the new component if necessary
        if (loader != null)
            loader.setContext(this);
        if (getState().isAvailable() && (loader != null) &&
            (loader instanceof Lifecycle)) {
            try {
                ((Lifecycle) loader).start();
            } catch (LifecycleException e) {
                log.error("StandardContext.setLoader: start: ", e);
            }
        }
    } finally {
        writeLock.unlock();
    }

    // Report this property change to interested listeners
    support.firePropertyChange("loader", oldLoader, loader);
}
```

这儿，我们感兴趣的就两行代码：

1. `((Lifecycle) oldLoader).stop(); // 旧的加载器停止`
2. `((Lifecycle) loader).start(); // 新的加载器启动`

## 总结

我们终于完整地分析完了Tomcat的整个启动过程+类加载过程。也了解并学习了Tomcat不同的类加载机制是为什么要这样设计，带来的附加作用又是怎样的。

从最后分析的load和start来看，第二入口在Catalina这个类，其对应的方法为`load()`和`start()`。后续分析源码，我们会从这两个方法入手。

除了Catalina之后，我们还要逐一分析巨多的tomcat组件，希望每个组件的分析都能带给我们不一样的阅读体验和设计方式~
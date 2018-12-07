---
layout: post
title:  深入理解feign（02）-源码解析
date:   2018-12-07 18:10:00 +0800
categories: SpringCloud
tag: java spring SpringCloud
---

* content
{:toc}

## 前言

通过上一篇文章[深入理解feign（01）-使用入门](https://www.jianshu.com/p/e3f31133db85)的介绍，我们了解了`Feign`如何使用。本节我们将深入`Feign`的源码，通过`启动`和`运行`两个阶段来分析源代码。

在进入正文之前，我们再来复习一下`Feign`的使用。

```java
public interface UserService {
    @RequestLine("GET /user/get?id={id}")
    User get(@Param("id") Long id);
}

public class User {
    Long id;
    String name;
}

public class Main {
    public static void main(String[] args) {
        UserService userService = Feign.builder()
            .options(new Request.Options(1000, 3500))
            .retryer(new Retryer.Default(5000, 5000, 3))
            .target(UserService.class, "http://api.server.com");
        System.out.println("user: " + userService.get(1L));
    }
}
```

整个流程大概是这样的：先定义一个接口，然后在接口的`类型声明`、`方法声明`和`方法参数`上面定义一些注解，然后通过`Feign.builder().{xx...}.target()`方法生成一个接口代理对象，最后对代理对象实施方法调用。

## 整体结构图

分析之前，先给出整体结构图（该图片借用了[Spring Cloud Feign设计原理](https://www.jianshu.com/p/8c7b92b4396c)），以便我们对`Feign`有一个整体的、清晰的认识。

![Feign整体结构图](https://upload-images.jianshu.io/upload_images/845143-eb0edb6dd921e977.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 启动阶段

启动阶段在于`Feign.Builder`的调用。包括两个部分，第一部分使用了`构建者模式`+`链式编程`来填充对象属性，第二部分使用了`target()`方法来生成代理对象。

### 1. Feign.Builder - 填充对象属性

```java
public abstract class Feign {

  public static Builder builder() {
    return new Builder();
  }

  public static class Builder {

    private final List<RequestInterceptor> requestInterceptors =
        new ArrayList<RequestInterceptor>();
    private Logger.Level logLevel = Logger.Level.NONE;
    private Contract contract = new Contract.Default();
    private Client client = new Client.Default(null, null);
    private Retryer retryer = new Retryer.Default();
    private Logger logger = new NoOpLogger();
    private Encoder encoder = new Encoder.Default();
    private Decoder decoder = new Decoder.Default();
    private ErrorDecoder errorDecoder = new ErrorDecoder.Default();
    private Options options = new Options();
    private InvocationHandlerFactory invocationHandlerFactory =
        new InvocationHandlerFactory.Default();
    private boolean decode404;

    public Builder logLevel(Logger.Level logLevel) {
      this.logLevel = logLevel;
      return this;
    }

    public Builder contract(Contract contract) {
      this.contract = contract;
      return this;
    }

    public Builder client(Client client) {
      this.client = client;
      return this;
    }

    public Builder options(Options options) {
      this.options = options;
      return this;
    }
    // 省略掉其他类似client()、options()的方法...
  }
}
```

### 2. Feign.Builder - target()方法

```java
public static class Builder {
  public <T> T target(Class<T> apiType, String url) {
    return target(new HardCodedTarget<T>(apiType, url));
  }

  public <T> T target(Target<T> target) {
    return build().newInstance(target);
  }

  public Feign build() {
    SynchronousMethodHandler.Factory synchronousMethodHandlerFactory =
        new SynchronousMethodHandler.Factory(client, retryer, requestInterceptors, logger,
                                             logLevel, decode404);
    ParseHandlersByName handlersByName =
        new ParseHandlersByName(contract, options, encoder, decoder,
                                errorDecoder, synchronousMethodHandlerFactory);
    return new ReflectiveFeign(handlersByName, invocationHandlerFactory);
  }
}
```

1. 为什么要有两个`target()`方法呢？原因在于通过`<T> T target(Target<T> target)`方法，我们可以提供更灵活的定制功能
2. `target()`最终会调用`new ReflectiveFeign(...)`来生成`Feign`实例
3. 关键类`SynchronousMethodHandler.Factory`用于创建一个`SynchronousMethodHandler`对象
4. 关键类`ParseHandlersByName`将`Target`的所有接口方法转换为`Map<String, MethodHandler>`对象
5. 关键类`ReflectiveFeign`是`Feign`的实现，`Feign`有一个模板方法`public abstract <T> T newInstance(Target<T> target);`用于生成代理对象

### 3. ReflectiveFeign.newInstance()方法

```java
@SuppressWarnings("unchecked")
@Override
public <T> T newInstance(Target<T> target) {
  // 1. 根据target，解析生成`MethodHandler`对象
  Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
  Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
  List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

  // 2. 对`MethodHandler`对象进行分类整理，整理成两类：default方法和基本方法
  for (Method method : target.type().getMethods()) {
    if (method.getDeclaringClass() == Object.class) {
      continue;
    } else if(Util.isDefault(method)) {
      DefaultMethodHandler handler = new DefaultMethodHandler(method);
      defaultMethodHandlers.add(handler);
      methodToHandler.put(method, handler);
    } else {
      methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
    }
  }
  // 3. 通过jdk动态代理生成代理对象，这儿是最关键的地方
  InvocationHandler handler = factory.create(target, methodToHandler);
  T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[]{target.type()}, handler);

  // 4. 将`DefaultMethodHandler`绑定到代理对象
  for(DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
    defaultMethodHandler.bindTo(proxy);
  }
  return proxy;
}
```

我们整理一下这个方法逻辑

1. 根据target，解析生成`MethodHandler`对象
2. 对`MethodHandler`对象进行分类整理，整理成两类：default方法和基本方法
3. 通过jdk动态代理生成代理对象，这儿是最关键的地方
4. 将`DefaultMethodHandler`绑定到代理对象

### 4. ParseHandlersByName.apply()方法

```java
public Map<String, MethodHandler> apply(Target key) {
  List<MethodMetadata> metadata = contract.parseAndValidatateMetadata(key.type());
  Map<String, MethodHandler> result = new LinkedHashMap<String, MethodHandler>();
  for (MethodMetadata md : metadata) {
    BuildTemplateByResolvingArgs buildTemplate;
    if (!md.formParams().isEmpty() && md.template().bodyTemplate() == null) {
      buildTemplate = new BuildFormEncodedTemplateFromArgs(md, encoder);
    } else if (md.bodyIndex() != null) {
      buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder);
    } else {
      buildTemplate = new BuildTemplateByResolvingArgs(md);
    }
    result.put(md.configKey(),
               factory.create(key, md, buildTemplate, options, decoder, errorDecoder));
  }
  return result;
}
```

1. 该方法使用`Contract`对象来解析接口和接口上的方法，然后生成接口方法元数据列表
2.  遍历接口方法元数据列表，根据工厂`SynchronousMethodHandler.Factory`生成`MethodHandler`对象列表
3. `Contract`可以有不同的实现，`Feign`自带了一套注解，实现类为`Contract.Default`。同时Spring Cloud对Spring MVC的注解（如@RequestMapping和@RequestParam等）也进行了封装，实现类为`SpringMvcContract`
4. `BaseContract.parseAndValidatateMetadata()`方法全是java反射的应用，限于篇幅，这儿不再深入到细节

接下来我们看看`SynchronousMethodHandler.Factory.create()`方法的实现，非常的简单，就是调用了`SynchronousMethodHandler`的全参构造方法。

```java
public MethodHandler create(Target<?> target, MethodMetadata md,
                            RequestTemplate.Factory buildTemplateFromArgs,
                            Options options, Decoder decoder, ErrorDecoder errorDecoder) {
  return new SynchronousMethodHandler(target, client, retryer, requestInterceptors, logger,
                                      logLevel, md, buildTemplateFromArgs, options, decoder,
                                      errorDecoder, decode404);
}
```

## 运行阶段

在`启动阶段`我们生成了接口的代理对象，运行阶段其实就是调用该代理对象的方法，来实现http远程调用。为了进一步加深我们队`Feign`的了解，我们这儿来分析一下代理对象方法是如何调用的。

回顾一下代理对象生成的代码。

```java
public <T> T newInstance(Target<T> target) {
  InvocationHandler handler = factory.create(target, methodToHandler);
  T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[]{target.type()}, handler);
}

static final class Default implements InvocationHandlerFactory {
  @Override
  public InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch) {
    return new ReflectiveFeign.FeignInvocationHandler(target, dispatch);
  }
}

static class FeignInvocationHandler implements InvocationHandler {

    private final Target target;
    private final Map<Method, MethodHandler> dispatch;

    FeignInvocationHandler(Target target, Map<Method, MethodHandler> dispatch) {
      this.target = checkNotNull(target, "target");
      this.dispatch = checkNotNull(dispatch, "dispatch for %s", target);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      if ("equals".equals(method.getName())) {
        try {
          Object
              otherHandler =
              args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
          return equals(otherHandler);
        } catch (IllegalArgumentException e) {
          return false;
        }
      } else if ("hashCode".equals(method.getName())) {
        return hashCode();
      } else if ("toString".equals(method.getName())) {
        return toString();
      }
      return dispatch.get(method).invoke(args);
    }

    @Override
    public boolean equals(Object obj) {
      if (obj instanceof FeignInvocationHandler) {
        FeignInvocationHandler other = (FeignInvocationHandler) obj;
        return target.equals(other.target);
      }
      return false;
    }

    @Override
    public int hashCode() {
      return target.hashCode();
    }

    @Override
    public String toString() {
      return target.toString();
    }
  }
```

关键的地方在于`FeignInvocationHandler.invoke()`方法中的这一行代码：`return dispatch.get(method).invoke(args);`。

`dispatch`正是`启动阶段`生成的`MethodHandler`列表，我们进去看看它的`invoke()`方法。

```java
final class SynchronousMethodHandler implements MethodHandler {
  @Override
  public Object invoke(Object[] argv) throws Throwable {
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Retryer retryer = this.retryer.clone();
    while (true) {
      try {
        return executeAndDecode(template);
      } catch (RetryableException e) {
        retryer.continueOrPropagate(e);
        if (logLevel != Logger.Level.NONE) {
          logger.logRetry(metadata.configKey(), logLevel);
        }
        continue;
      }
    }
  }
}
```

那么这个方法做了哪些事情呢？

1. `RequestTemplate`对象的生成
2. `Retryer`的重试机制
3. `executeAndDecode(template)`方法

### 1. RequestTemplate对象的生成

该方法首先生成了`RequestTemplate `对象，该对象是通过`BuildTemplateFromArgs`来创建的，而`BuildTemplateFromArgs`的创建过程在于`ParseHandlersByName.apply()`方法，我们回过头再来看看这个方法。这个方法会根据`参数的形式`来生成不同的类对象，他们处理参数逻辑会不一样。

1. 如果有表单参数，则生成`BuildFormEncodedTemplateFromArgs `类型对象
2. 如果带body，则生成`BuildEncodedTemplateFromArgs `类型对象
3. 其他情况，生成`BuildTemplateByResolvingArgs `类型对象

```java
static final class ParseHandlersByName {
  public Map<String, MethodHandler> apply(Target key) {
    List<MethodMetadata> metadata = contract.parseAndValidatateMetadata(key.type());
    Map<String, MethodHandler> result = new LinkedHashMap<String, MethodHandler>();
    for (MethodMetadata md : metadata) {
      BuildTemplateByResolvingArgs buildTemplate;
      if (!md.formParams().isEmpty() && md.template().bodyTemplate() == null) {
        buildTemplate = new BuildFormEncodedTemplateFromArgs(md, encoder);
      } else if (md.bodyIndex() != null) {
        buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder);
      } else {
        buildTemplate = new BuildTemplateByResolvingArgs(md);
      }
      result.put(md.configKey(),
                 factory.create(key, md, buildTemplate, options, decoder, errorDecoder));
    }
    return result;
  }
}
```

### 2. Retryer的重试机制

`Retryer`不是线程安全的对象，所以每一次方法调用我们都需要借助于原型模式来生成一个新的对象。`Retryer`关键的模板方法如下

> 注释说得相当清楚！如果重试允许的话，直接返回（可能在休眠一定时间之后）。其他情况，需要对外抛出异常对请求进行终止。

```java
/**
 * if retry is permitted, return (possibly after sleeping). Otherwise propagate the exception.
 */
void continueOrPropagate(RetryableException e);
```

我们回头来看看`invoke()`方法。在执行的过程中要想重试，需要抛出特定的异常`RetryableException`，否则重试将不生效。重试的时候，如果`Feign`设置的日志级别比`NONE`更高，将记录重试日志。限于篇幅，本文不展开讲述Retryer的实现。

```java
@Override
public Object invoke(Object[] argv) throws Throwable {
  RequestTemplate template = buildTemplateFromArgs.create(argv);
  Retryer retryer = this.retryer.clone();
  while (true) {
    try {
      return executeAndDecode(template);
    } catch (RetryableException e) {
      retryer.continueOrPropagate(e);
      if (logLevel != Logger.Level.NONE) {
        logger.logRetry(metadata.configKey(), logLevel);
      }
      continue;
    }
  }
}
```

### 3. `executeAndDecode(template)`方法

终于进入到了调用client关键的一环了，真是很不容易！我们进一步来看看这个方法，该方法比较长，前方高能！

```java
Object executeAndDecode(RequestTemplate template) throws Throwable {
  // 1. 根据`RequestTemplate`生成`Request`对象
  Request request = targetRequest(template);

  // 2. 记录请求日志
  if (logLevel != Logger.Level.NONE) {
    logger.logRequest(metadata.configKey(), logLevel, request);
  }

  Response response;
  long start = System.nanoTime();
  try {
    // 3. 调用`client`对象的`execute()`方法执行http调用逻辑。`execute()`内部可能设置request对象，也可能不设置，所以需要`response.toBuilder().request(request).build();`这一行代码
    response = client.execute(request, options);
    // ensure the request is set. TODO: remove in Feign 10
    response.toBuilder().request(request).build();
  } catch (IOException e) {
    // 4. IOException的时候，记录日志
    if (logLevel != Logger.Level.NONE) {
      logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
    }
    // 5. IOException的时候，包装成`RetryableException`异常
    throw errorExecuting(request, e);
  }
  // 6. 统计`client.execute()`花费的时间，并后续日志记录
  long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);

  boolean shouldClose = true;
  try {
    if (logLevel != Logger.Level.NONE) {
      response =
          logger.logAndRebufferResponse(metadata.configKey(), logLevel, response, elapsedTime);
      // ensure the request is set. TODO: remove in Feign 10
      response.toBuilder().request(request).build();
    }
    // 7. 如果元数据返回类型是`Response`，直接返回回去即可，不需要`decode()`解码
    if (Response.class == metadata.returnType()) {
      if (response.body() == null) {
        return response;
      }
      if (response.body().length() == null ||
              response.body().length() > MAX_RESPONSE_BUFFER_SIZE) {
        shouldClose = false;
        return response;
      }
      // Ensure the response body is disconnected
      byte[] bodyData = Util.toByteArray(response.body().asInputStream());
      return response.toBuilder().body(bodyData).build();
    }
    // 8. 将`response`解码返回，主要对`2xx`和`404`等进行解码，`404`需要特别的开关控制。其他情况，使用`errorDecoder`进行解码，以异常的方式返回
    if (response.status() >= 200 && response.status() < 300) {
      if (void.class == metadata.returnType()) {
        return null;
      } else {
        return decode(response);
      }
    } else if (decode404 && response.status() == 404 && void.class != metadata.returnType()) {
      return decode(response);
    } else {
      throw errorDecoder.decode(metadata.configKey(), response);
    }
  } catch (IOException e) {
    if (logLevel != Logger.Level.NONE) {
      logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime);
    }
    throw errorReading(request, response, e);
  } finally {
    // 9. 在对`Response`使用完成之后，需要关闭`Response`，因为`Response`可能有对输入流的操作
    if (shouldClose) {
      ensureClosed(response.body());
    }
  }
}
```

该方法做的事情，我们罗列一下：

1. 根据`RequestTemplate`生成`Request`对象
2. 记录请求日志
3. 调用`client`对象的`execute()`方法执行http调用逻辑。`execute()`内部可能设置request对象，也可能不设置，所以需要`response.toBuilder().request(request).build();`这一行代码
4. IOException的时候，记录日志
5. IOException的时候，包装成`RetryableException`异常
6. 统计`client.execute()`花费的时间，并后续日志记录
7. 如果元数据返回类型是`Response`，直接返回回去即可，不需要`decode()`解码
8. 将`response`解码返回，主要对`2xx`和`404`等进行解码，`404`需要特别的开关控制。其他情况，使用`errorDecoder`进行解码，以异常的方式返回
9. 在对`Response`使用完成之后，需要关闭`Response`，因为`Response`可能有对输入流的操作

我们接下来分析一下每个点涉及的逻辑。先来看看`targetRequest()`，这是先调用的拦截器方法，再生成`Request`对象。需要注意的是，这儿的拦截器并没有像tomcat那样使用`过滤器模式`来实现，而仅仅是简单的for循环。

```java
Request targetRequest(RequestTemplate template) {
  for (RequestInterceptor interceptor : requestInterceptors) {
    interceptor.apply(template);
  }
  return target.apply(new RequestTemplate(template));
}
```

`feign.Client.Default.execute()`方法使用了`HttpURLConnection`的方式来请求web服务器，并没有使用对象池技术，所以性能较低。如何使用`HttpURLConnection`和web服务器进行通信的底层细节并不是我们需要分析的重点，在此仅一笔带过。

接下来我们比较关心的是`decode()`方法，该方法将输入流转换为我们需要的POJO业务对象。也是非常简单，只是调用传入的`Decoder.decode()`方法。

```java
Object decode(Response response) throws Throwable {
  try {
    return decoder.decode(response, metadata.returnType());
  } catch (FeignException e) {
    throw e;
  } catch (RuntimeException e) {
    throw new DecodeException(e.getMessage(), e);
  }
}
```

至此，我们完整地分析完了`executeAndDecode()`方法。

## 总结

我们先给出`Feign`的整体结构图，使得我们有一个总的认识。然后通过`Feign`的`启动`和`运行`两个阶段的源码阅读和分析，已经对`Feign`有了比较深入的了解。

其实说到底，`Feign`有点类似于http客户端的门面(`Facade`)。我们可以扩展实现编码器、解码器、Client等组件。对于使用者提供了基于注解的规范，我们可以使用各种自己想要的注解来定制我们的调用方式。将我们从复杂的、多变的和细节的底层http客户端类库中解放出来。

笔者认为，站在抽象的角度来说，`Feign`和`Slf4j`是同一类东西，他们都可以看成是一套规范，或者是一套模式。

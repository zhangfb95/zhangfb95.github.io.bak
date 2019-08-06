---
layout: post
title:  深入理解feign（01）-使用入门
date:   2018-12-06 13:15:00 +0800
categories: Spring Cloud Feign
tag: java spring SpringCloud
---

* content
{:toc}

## 前言

feign的使用文档中，开篇第一句就是`Feign makes writing java http clients easier`，中文译为`Feign使得写java http客户端更容易`。言下之意，`Feign`其实是一个java http客户端的类库。

本文我们将对`Feign`做一个大概的了解，并对其基本用法进行掌握，后续章节我们将深入`Feign`的各种应用场景及源码，让我们不仅知其然，还要知其所以然。

## 为什么选择Feign

如果读者有用过`Jersey`、`Spring`或`CXF`来实现web服务端，那么一定会对他们通过注解方法来定义接口的方式大开眼界。那么，我们为什么选择`Feign`，它又有哪些优势呢？

1. `Feign`允许我们通过注解的方式实现http客户端的功能，给了我们除了`Apache HttpComponents`之外的另一种选择
2. `Feign`能用最小的性能开销，让我们调用web服务器上基于文本的接口。同时允许我们自定义`编码器`、`解码器`和`错误处理器`等等

## Feign如何工作的呢

`Feign`通过`注解和模板`的方式来定义其工作方式，参数（包括url、method、request和response等）非常直观地融入到了模板中。尽管`Feign`设计成了只支持`基于文本的接口`，但正是它的这种局限降低了实现的复杂性。而我们写http客户端代码的时候，超过90%的场景是基于文本的接口调用。另一个方面，使用`Feign`还可以简化我们的单元测试。

## 基本用法

典型的用法如下所示。

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

## Feign下面接口的注解

`Feign`通过`注解和模板`的方式来定义契约，那么又有哪些注解，分别是做什么用的呢？下面的表格参考了[Feign官网的Annotation](https://github.com/OpenFeign/feign#interface-annotations)，给出了基本用法。

|Annotation|	Interface Target|	Usage|
|---|---|---|
|@RequestLine|Method|用于定义method和uri模板，其值由`@Param`传入。|
|@Param|Parameter|模板变量，它的值将被用于替换表达式。|
|@Headers|Method, Type|用于定义header模板，其值由`@Param`传入。该注解可声明在Type上，也可声明在Method上。当声明在Type上时，相当于其下面的所有Method都声明了。当声明在Method上时，仅对当前Method有效。|
|@QueryMap|Parameter|可定义成一个key-value的Map，也可以定义成POJO，用以扩展进查询字符串。|
|@HeaderMap|Parameter|可定义成一个key-value的Map，用于扩展请求头。|
|@Body|Method|用于定义body模板，其值由`@Param`传入。|

## 模板和表达式

模板和表达式模式，是基于[URI Template - RFC 6570](https://tools.ietf.org/html/rfc6570)来实现的。表达式通过在方法上`@Param`修饰的参数来填充。

表达式必须以`{}`来包装变量名。也可使用正则表达式来验证，`变量名`+`:`+`正则表达式`的方式。表达式定义如下所示：

1. `{name}`
2. `{name:[a-zA-Z]*}`

可以运用表达式的地方有下面几处。

1. `@RequestLine`
2. `@QueryMap`
3. `@Headers`
4. `@HeaderMap`
5. `@Body`

他们将遵循[URI Template - RFC 6570](https://tools.ietf.org/html/rfc6570)规约。

1. 未正确匹配的表达式将被忽略（忽略的意思就是，该变量在表达式中将被设置为null）
2. 表达式值设置之前不会通过`Encoder`进行编码
3. `@Body`使用的时候必须在`Header`里通过`Content-Type `指明内容类型

`@Param`有`expander `属性，该属性为Class类型，可以通过编码的方式更灵活地进行转换。如果返回的结果为null或空字符串，表达式将被忽略。

```java
public interface Expander {
    String expand(Object value);
}
```

另外，`@Param`可同时运用到多处，如下所示：

```java
public interface ContentService {
  @RequestLine("GET /api/documents/{contentType}")
  @Headers("Accept {contentType}")
  String getDocumentByType(@Param("contentType") String type);
}
```

## Feign的自定义设置

可以通过`Feign.builder() `来自定义设置一些拦截器，用于增强其语义。比如我们可以增加超时拦截器、编码拦截器、解码拦截器、重试拦截器等等。如下所示：

```java
interface Bank {
  @RequestLine("POST /account/{id}")
  Account getAccountInfo(@Param("id") String id);
}

public class BankService {
  public static void main(String[] args) {
    Bank bank = Feign.builder().decoder(
        new AccountDecoder())
        .target(Bank.class, "https://api.examplebank.com");
  }
}
```

## Feign集成第三方组件

可以和很容易地和第三方组件结合使用，扩展了其功能，也增加了其灵活性。我们可以查阅官网文档，链接地址：[integrations](https://github.com/OpenFeign/feign#integrations)。

### 1. Gson

通过encoder和decoder来使用

```java
public class Example {
  public static void main(String[] args) {
    GsonCodec codec = new GsonCodec();
    GitHub github = Feign.builder()
                         .encoder(new GsonEncoder())
                         .decoder(new GsonDecoder())
                         .target(GitHub.class, "https://api.github.com");
  }
}
```

### 2. Jackson

和`Gson`一样，也是通过encoder和decoder来使用

```java
public class Example {
  public static void main(String[] args) {
      GitHub github = Feign.builder()
                     .encoder(new JacksonEncoder())
                     .decoder(new JacksonDecoder())
                     .target(GitHub.class, "https://api.github.com");
  }
}
```

### 3. JAXB

和`Gson`一样，也是通过encoder和decoder来使用

```java
public class Example {
  public static void main(String[] args) {
    Api api = Feign.builder()
             .encoder(new JAXBEncoder())
             .decoder(new JAXBDecoder())
             .target(Api.class, "https://apihost");
  }
}
```

### 4. JAX-RS

`JAX-RS`定义了自己的一套注解，我们可以通过和`JAX-RS`注解的集成来定义我们自己的`注解皮肤`。该功能需要结合`contract`使用。

```java
interface GitHub {
  @GET @Path("/repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@PathParam("owner") String owner, @PathParam("repo") String repo);
}

public class Example {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                       .contract(new JAXRSContract())
                       .target(GitHub.class, "https://api.github.com");
  }
}
```

### 5. OkHttp

`OkHttp`是一个http客户端类库，我们也可以将其包装成`Feign`的形式。该功能需要结合client来使用。

```java
public class Example {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                     .client(new OkHttpClient())
                     .target(GitHub.class, "https://api.github.com");
  }
}
```

### 6. Ribbon

`Ribbon`提供了客户端负载均衡功能。我们也可以和其一起集成使用。

```java
public class Example {
  public static void main(String[] args) {
    MyService api = Feign.builder()
          .client(RibbonClient.create())
          .target(MyService.class, "https://myAppProd");
  }
}
```

### 7. Hystrix

`Hystrix`是一个断路器组件，为了保证分布式系统的健壮性，在某一些服务不可用的情况下，可避免出现雪崩效应。也可以和`Feign`结合使用

```java
public class Example {
  public static void main(String[] args) {
    MyService api = HystrixFeign.builder().target(MyService.class, "https://myAppProd");
  }
}
```

### 8. SOAP

`SOAP`是基于http之上的一种协议，其通信方式使用的是xml格式。

```java
public class Example {
  public static void main(String[] args) {
    Api api = Feign.builder()
	     .encoder(new SOAPEncoder(jaxbFactory))
	     .decoder(new SOAPDecoder(jaxbFactory))
	     .errorDecoder(new SOAPErrorDecoder())
	     .target(MyApi.class, "http://api");
  }
}
```

### 9. SLF4J

`slf4j`是一个日志门面，给各种日志框架提供了统一的入口。

```java
public class Example {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                     .logger(new Slf4jLogger())
                     .target(GitHub.class, "https://api.github.com");
  }
}
```

### Decoders

当我们的接口返回类型不为`feign.Response`、`String`、`byte[]`和`void`时，我们必须定义一个非默认的解码器。以`Gson`为例

```java
public class Example {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                     .decoder(new GsonDecoder())
                     .target(GitHub.class, "https://api.github.com");
  }
}
```

当我们想在`feign.Response`进行解码之前做一些事情，我们可以通过`mapAndDecode `来自定义。

```java
public class Example {
  public static void main(String[] args) {
    JsonpApi jsonpApi = Feign.builder()
                         .mapAndDecode((response, type) -> jsopUnwrap(response, type), new GsonDecoder())
                         .target(JsonpApi.class, "https://some-jsonp-api.com");
  }
}
```

### Encoders

当我们定义的接口method为`POST`，且传入的类型不为`String`或者`byte[]`，我们需要自定义编码器。同时需要在header上指明`Content-Type `。

```java
static class Credentials {
  final String user_name;
  final String password;

  Credentials(String user_name, String password) {
    this.user_name = user_name;
    this.password = password;
  }
}

interface LoginClient {
  @RequestLine("POST /")
  void login(Credentials creds);
}

public class Example {
  public static void main(String[] args) {
    LoginClient client = Feign.builder()
                              .encoder(new GsonEncoder())
                              .target(LoginClient.class, "https://foo.com");

    client.login(new Credentials("denominator", "secret"));
  }
}
```

## 扩展功能

### 1. 基本使用

接口的定义可以是单一的接口，也可以是带继承层级的接口列表。

```java
interface BaseAPI {
  @RequestLine("GET /health")
  String health();

  @RequestLine("GET /all")
  List<Entity> all();
}
interface CustomAPI extends BaseAPI {
  @RequestLine("GET /custom")
  String custom();
}
```

我们也可以定义泛型类型

```java
@Headers("Accept: application/json")
interface BaseApi<V> {

  @RequestLine("GET /api/{key}")
  V get(@Param("key") String key);

  @RequestLine("GET /api")
  List<V> list();

  @Headers("Content-Type: application/json")
  @RequestLine("PUT /api/{key}")
  void put(@Param("key") String key, V value);
}

interface FooApi extends BaseApi<Foo> { }

interface BarApi extends BaseApi<Bar> { }
```

### 2. 日志级别

Feign会根据不同的日志级别，来输出不同的日志，在Feign里面定义了4种日志级别。

```java
/**
 * Controls the level of logging.
 */
public enum Level {
  /**
   * No logging.不记录日志
   */
  NONE,
  /**
   * Log only the request method and URL and the response status code and execution time.
   * 仅仅记录请求方法、url、返回状态码及执行时间
   */
  BASIC,
  /**
   * Log the basic information along with request and response headers.
   * 在记录基本信息上，额外记录请求和返回的头信息
   */
  HEADERS,
  /**
   * Log the headers, body, and metadata for both requests and responses.
   * 记录全量的信息，包括：头信息、body信息、请求和返回的元数据等
   */
  FULL
}
```

使用方式如下：

```java
public class Example {
  public static void main(String[] args) {
    GitHub github = Feign.builder()
                     .decoder(new GsonDecoder())
                     .logger(new Logger.JavaLogger().appendToFile("logs/http.log"))
                     .logLevel(Logger.Level.FULL)
                     .target(GitHub.class, "https://api.github.com");
  }
}
```

### 3. 请求拦截器

我们可以通过定义一个请求拦截器`RequestInterceptor `来对请求数据进行修改，比如添加一个请求头或者校验授权信息等等。

```java
static class ForwardedForInterceptor implements RequestInterceptor {
  @Override public void apply(RequestTemplate template) {
    template.header("X-Forwarded-For", "origin.host.com");
  }
}

public class Example {
  public static void main(String[] args) {
    Bank bank = Feign.builder()
                 .decoder(accountDecoder)
                 .requestInterceptor(new ForwardedForInterceptor())
                 .target(Bank.class, "https://api.examplebank.com");
  }
}
```

### 4. 动态查询参数@QueryMap

一般情况下，我们使用`@QueryMap`时，传入的参数为`Map<String, Object>`类型，如下所示：

```java
public interface Api {
  @RequestLine("GET /find")
  V find(@QueryMap Map<String, Object> queryMap);
}
```

但有时候，为了让我们的参数定义得更清晰易懂，我们也可以使用`POJO`方式，如下所示。这种方式是通过反射直接获取字段名称和值的方式来实现的。如果`POJO`里面的某个字段为null或者空串，将会从查询参数中移除掉（也就是不生效）。

 ```java
public interface Api {
  @RequestLine("GET /find")
  V find(@QueryMap CustomPojo customPojo);
}
```

如果我们更喜欢使用`getter`、`setter`的方式来读取和设置值，那么我们可以自定义查询参数编码器。

```java
public class Example {
  public static void main(String[] args) {
    MyApi myApi = Feign.builder()
                 .queryMapEncoder(new BeanQueryMapEncoder())
                 .target(MyApi.class, "https://api.hostname.com");
  }
}
```

### 5. 自定义错误处理器

`Feign`有默认的错误处理器，当我们想自行处理错误，也是可以的。可以通过自定义`ErrorDecoder`来实现。

```java
public class Example {
  public static void main(String[] args) {
    MyApi myApi = Feign.builder()
                 .errorDecoder(new MyErrorDecoder())
                 .target(MyApi.class, "https://api.hostname.com");
  }
}
```

它会捕获http返回状态码为`非2xx`的错误，并调用`ErrorDecoder. decode()`方法。我们可以抛出自定义异常，或者做额外的处理逻辑。如果我们想重复多次调用，需要抛出`RetryableException `，并定义且注册额外的`Retryer`。

### 6. 自定义Retry

我们可以通过实现`Retryer`接口的方式来自定义重试策略。Retry会对`IOException`和`ErrorDecoder `组件抛出的`RetryableException`进行重试。如果达到了最大重试次数仍不成功，我们可以抛出`RetryException `。

自定义`Retryer`的使用如下所示：

```java
public class Example {
  public static void main(String[] args) {
    MyApi myApi = Feign.builder()
                 .retryer(new MyRetryer())
                 .target(MyApi.class, "https://api.hostname.com");
  }
}
```

### 7. 接口的静态方法和默认方法

在java8及以上版本，我们可以在接口里面定义静态方法和默认方法。`Feign`也支持这种写法，但是有特殊的作用。

1. 静态方法可以写自定义的`Feign`定义
2. 默认方法可以在参数中传入默认值

```java
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);

  @RequestLine("GET /users/{username}/repos?sort={sort}")
  List<Repo> repos(@Param("username") String owner, @Param("sort") String sort);

  default List<Repo> repos(String owner) {
    return repos(owner, "full_name");
  }

  /**
   * Lists all contributors for all repos owned by a user.
   */
  default List<Contributor> contributors(String user) {
    MergingContributorList contributors = new MergingContributorList();
    for(Repo repo : this.repos(owner)) {
      contributors.addAll(this.contributors(user, repo.getName()));
    }
    return contributors.mergeResult();
  }

  static GitHub connect() {
    return Feign.builder()
                .decoder(new GsonDecoder())
                .target(GitHub.class, "https://api.github.com");
  }
}
```

## 总结

本文先简单介绍了`Feign`，然后给出了一个入门级的例子，最后对每个功能、组件和扩展进行了补充说明。楼主相信通过这些文字，足够让我们进入`Feign`的大门了。

后面我们将更加深入地了解`Feign`，尤其是`Feign`的源码。

## 参考链接

+ https://github.com/OpenFeign/feign
+ https://www.jianshu.com/p/3d597e9d2d67
+ https://my.oschina.net/joryqiao/blog/1925633

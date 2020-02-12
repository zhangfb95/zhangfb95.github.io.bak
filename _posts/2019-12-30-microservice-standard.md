---
layout: post
title:  关于微服务“统一标准”的一些看法
date:   2019-12-30 23:27:32 +0800
categories: SpringCloud
tags: SpringCloud
---

* content
{:toc}

## 前言

在Java领域，主流的微服务框架是Spring Cloud + Spring Boot。这套框架集成了各类常用的组件，也提供了足够的扩展性，方便对其他组件进行集成。简化我们的集成难度之外，也让我们能快速地实现分布式架构。

那么，问题来了。

这么多组件协同地工作，如果没有在包和类的命名上没有进行统一规划，我们的代码将变得凌乱而脆弱。业界比较好的开发模式是DDD（领域驱动设计），这种模式虽然好，但是使用门槛较高，快速开发难度大。而传统开发模式是三层架构，即：Controller、Service和Persistense的方式，这种方式在大型项目的后期维护成本较高，但是中型项目的前期和中期，还是能带来足够多的效益。

因此，在三层架构开发模式的基础之上，楼主站在自己的角度上对项目和包的结构进行了规划，统一了包和类的命名。并以此形成一套完整的、可落地的微服务命名解决方案。

> 注：这儿的解决方案限于Spring Cloud+Spring Boot的架构，其他架构仅供参考。

## 启动类

启动类统一命名为`Application`，且必须加上以下注解。

```java
@SpringBootApplication
@EnableEurekaClient
@ComponentScan("com.xxxcompany")
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

## 项目结构

项目统一使用maven进行管理，且包括以下模块。

+ {projectName} (folder: projectName) -- 根模块
    + ${projectName}-dataaccess (folder: dataaccess) -- 数据访问模块
    + ${projectName}-feign (folder: feign) -- 远程调用模块
    + ${projectName}-common (folder: common) -- 公共模块
    + ${projectName}-core (folder: core) -- 核心业务模块
    + ${projectName}-api (folder: api) -- 接口模块
    + ${projectName}-ipi (folder: ipi) -- 接口定义模块，可选

模块间依赖关系如下所示。

```
api
  -> ipi
  -> core
    -> dataaccess
    -> common
    -> feign
    -> ipi
```

项目间依赖关系如下所示。

```
Aapi -> Bipi
```

需要特别说明的是，为了避免各个微服务项目之间的相互影响，ipi这个模块其实并不是必须的。这儿之所以会罗列ipi模块，是因为在某些场景下，ipi模块的使用可将问题尽早暴露出来。问题暴露的阶段大致包括：

1. 编译期（最好）
2. 启动期（其次）
3. 运行期（再次）


包和类的命名规则应如下所示。
> 【注】：这儿的`[D]`表示是一个文件夹或者包，`[F]`表示是一个文件或者类。

```
# dataaccess
com.${companyName}.${projectName}.dataaccess
  -> dao [D]
    -> mysql [D]
      -> XxxMapper [F]
    -> cache [D]
      -> XxxCache [F]
  -> domain [D]
    -> mysql [D]
      -> XxxDO [F]
      -> ext [D]
        -> XxxExtDO [F]

# feign
com.${companyName}.${projectName}.feign
  -> ${thirdProjectName} [D]
    -> XxxFeign [F]
    -> param [D] [可选]
      -> request.xxx [D]
        -> XxxFeignReq [F]
      -> response.xxx [D]
        -> XxxFeignRes [F]

# common
com.${companyName}.${projectName}.common
  -> enums [D]
    -> XxxEnum [F]
  -> config [D]
    -> XxxConfig [F]
  -> configbean [D]
    -> XxxConfigBean [F]
  -> util [D]
    -> XxxUtil [F]

# core
com.${companyName}.${projectName}.core
  -> mq [D]
    -> rabbitmq [D]
      -> consumer [D]
        -> mo [D]
          -> XxxMO [F]
        -> XxxAmqpConsumer [F]
      -> producer [D]
        -> mo [D]
          -> XxxMO [F]
        -> XxxAmqpProducer [F]
    -> kafka [D]
      -> consumer [D]
        -> mo [D]
          -> XxxMO [F]
        -> XxxKafkaConsumer [F]
      -> producer [D]
        -> mo [D]
          -> XxxMO [F]
        -> XxxKafkaProducer [F]
  -> service [D]
    -> ${function} [D]
      -> XxxService [F]
      -> impl [D]
        -> XxxServiceImpl [F]

# api
com.${companyName}.${projectName}.api
  -> exception [D] [可选]
    -> GlobalExceptionHandler [F]
  -> controller [D]
    -> XxxApi [F]
  -> Application.java [F]

# ipi
# 为了使服务更容易被治理，我们对所有服务进行了分层，总共四层。
# 同时，为了让我们的请求和返回类更容易编码和阅读，我们对请求类和响应类进行后缀约定。
# 1. web层 (XxxReq / XxxRes)
# 2. 业务服务层 (MicReq / MicRes)
# 3. 业务中台层 (BcpReq / BcpRes)
# 4. 数据中台层 (DcpReq / DcpRes)
com.${companyName}.${projectName}.ipi
  -> icontroller [D]
    -> XxxIpi [F]
  -> param.request.${icontrollerName} [D]
    -> XxxReq [F]
  -> param.response.${icontrollerName} [D]
    -> XxxRes [F]
  -> statuscode [D]
    -> XxxStatusCode [F]
  -> validator [D]
    -> XxxAnnotationConstraintValidator [F]
    -> XxxAnnotation [F]
```

## 接口返回类

分布式架构中，通过对接口返回（也叫做接口响应）的状态码、描述信息、错误信息和挂载对象进行统一格式，我们将有可能对接口拦截进行更规范地处理。这儿，接口返回类，我们统一使用`Response`，该类包含以下属性。

```java
@Data
public class Resposne<T> {

    private boolean success; // 成功标记
    private long code; // 状态码。1000表示成功，其他表示失败
    private String msg; // 描述信息
    private String errorMsg; // 错误信息
    private T data; // 挂载对象

    public Response<T> fallback() {
        // Some converted codes is here.
        // When failed, We can throws more BusinessException.
        // If you want to handle your codes more simple, 
        // just overload the fallback() methods.
    }
}
```

当挂载数据不需要包含任何数据时，必须使用`java.lang.Void`作为泛型参数。如果挂载对象有数据，需使用POJOs方式定义，且统一使用lombok注解，注解类型及顺序`必须如下所示！`
如果挂载对象类需要继承其他类，必须加上注解`@ToString(callSuper = true)`，从而避免`toString()`方法在调用后没有输出父类字段属性的异常情况出现。

> 【注】：挂载类应尽量避免使用继承特性。可使用字段冗余定义+Orika转换的方式来代替，效果更好。

```java
@AllArgsConstructor
@NoArgsConstructor
@Data
@Accessors(chain = true)
```

## 状态码

状态码合理有效地定义，将降低我们排查线上问题的难度，加快我们定位问题的速度。
因此，我们对状态码进行如下约定：
1. 统一继承自`IStatusCode`接口，默认实现类为`StatusCode`。
2. `StatusCode`为公共状态码类，状态码范围为1000-9999。
其中，成功状态码统一为1000，业务错误状态码统一为1006。
3. 自定义状态码生成规则为：`serviceCode * 1000000L + productCode * 10000L + customCode`。
这种规则，支持的产品数量为100个（00 -> 99），而产品下的服务数量却不受限制。

```java
/**
 * 状态码抽象接口
 */
public interface IStatusCode {

    /**
     * 状态码
     */
    long getCode();

    /**
     * 描述信息
     */
    String getMsg();
}

/**
 * 公共状态码
 */
@Getter
@AllArgsConstructor
public enum StatusCode implements IStatusCode {

    SUCCESS(1000, "操作成功", "操作成功"),
    SYSTEM_ERROR(1001, "系统未知错误", "系统未知错误"),
    PERMISSION_DENIED(1002, "没有权限", "没有权限"),
    DATA_ERROR(1003, "数据错误", "数据错误"),
    REPEAT_SUBMIT(1004, "重复提交", "重复提交"),
    PARAM_ERROR(1005, "参数错误", "参数错误"),
    BUSINESS_ERROR(1006, "业务错误", "业务错误"),
    ALERT_ERROR(1007, "业务错误", "弹框提示类业务错误"),
    ;

    /**
     * 状态码
     */
    private long code;

    /**
     * 描述信息
     */
    private String msg;

    /**
     * 错误信息
     */
    private String errorMsg;
}
```

## 对象转换

相对于Apache和Spring内置的Bean转换类库来说，`Orika`提供了更为完善的映射转换机制，它是一个优秀的、高性能的、低侵入的对象转换工具类库。对于对象之间的转换，我们在这儿统一使用`Orika`进行操作。
所有的转换器统一继承自`DefaultConverter`，且统一命名为`XxxConverter`，格式大致如下所示：

```java
// 定义 start
@Component
public class SelfDefinedConverter extends BaseConverter {
    public SelfDefinedConverter() {
        // 特殊的规则定义放在这儿
    }

    public TargetType map(SourceType sourceObj) {
    }
}
// 定义 end

// 使用 start
@Autowired
private SelfDefinedConverter selfDefinedConverter;

TargetType targetObj = selfDefinedConverter.map(sourceObj);
// 使用 end
```

BaseConver如下所示：

```java
/**
 * 抽象的Converter，可用于类型注册、绑定mapperFacade，其目的为生成映射对象。
 */
public abstract class BaseConverter {

    protected MapperFactory mapperFactory = OrikaMapperHelper.mapperFactory;

    protected void register(Class<?> typeA, Class<?> typeB) {
        mapperFactory.classMap(typeA, typeB)
                .mapNulls(false)
                .mapNullsInReverse(false)
                .byDefault()
                .register();
    }

    public MapperFacade getMapperFacade() {
        return mapperFactory.getMapperFacade();
    }
}
```

## 总结

楼主基于自己过往的经验和思考，提出了对于RPC、返回类、Web校验器、状态码、关系型数据库、消息中间件、配置中心和转换器的统一命名，并对用法进行了简单的举例。其实，本文对微服务的囊括并不全面。后续，我们将根据集成的组件，进一步对命名和用法进行统一，使得我们的项目真正地达到`企业级`。增加我们代码的可阅读性、可维护性和健壮性。

楼主能想到的后续可集成的组件大致包括：

1. Elasticsearch -- 一个高性能的全文检索中间件
2. HBase -- 可扩展的列存储数据库
3. TIDB -- 关系型数据库的分布式架构版本
4. Quartz -- 定时调度器（单机版和集群版都有）
5. FastDFS -- 小文件分布式存储引擎
6. Zookeeper -- 基于CP实现的分布式协调服务
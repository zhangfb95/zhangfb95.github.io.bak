---
layout: post
title:  关于Mysql隔离级别问题的排查过程
date:   2020-01-14 22:45:32 +0800
categories: Mysql
tags: Mysql
---

* content
{:toc}

## 前言

我们都知道，Mysql中的InnoDB存储引擎是通过行级锁来保证数据完整性的，那么哪些时候会出现行级锁呢？总的来说，就是对同一行记录操作的时候，包括：插入、修改和删除。

Mysql的插入语句，可以附加ingore关键字来实现特殊的效果，这种方式可有效避免对异常的处理，增加了代码的灵活性。但是，在使用的时候，我们发现一个诡异的问题，即：在insert ignore不成功的时候，并不能通过唯一键查询到被别的线程插入的记录。

那么，接下来就让我们细细地分析下原因。

## 表结构

首先，创建一个带有唯一键的表`test_user`，后续我们所有的数据写入和查询都将基于这个表。

```sql
CREATE TABLE `test_user` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键id',
  `uuid` varchar(50) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT '唯一键',
  `name` varchar(50) COLLATE utf8mb4_unicode_ci NOT NULL COMMENT '姓名',
  `gmt_create` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `gmt_modified` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `unq_uuid` (`uuid`)
) ENGINE=InnoDB AUTO_INCREMENT=31 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='测试用户'
```

## 持久层代码

持久层方法只有两个

+ 一个是insertIgnore()，用于带返回结果的写入操作。
+ 一个是getByUuid()，用于根据uuid唯一键查询记录。

`TestUserMapper`代码如下：

```java
public interface TestUserMapper {
    int insertIgnore(TestUserDO testUserDO);
    TestUserDO getByUuid(@Param("uuid") String uuid);
}
```

`TestUserMapper.xml`代码如下：

```xml
<insert id="insertIgnore" useGeneratedKeys="true" keyColumn="id" keyProperty="id">
  INSERT IGNORE test_user(uuid, `name`) VALUES(#{uuid}, #{name})
</insert>

<select id="getByUuid" resultMap="BaseResultMap">
  SELECT * FROM test_user WHERE uuid = #{uuid}
</select>
```

## 服务层代码

我们创建了类`UserService`，为了保证事务的一致性，我们加上了注解`@Transactional`。这个类做了以下事情：

1. 首先，根据uuid唯一键在数据库查询一下记录
2. 如果记录存在，则直接返回
3. 如果记录不存在，那么执行insertIgnore操作
4. 在记录不存在的前提下，如果insertIgnore返回的记录数>0，那么休眠20秒，目的是让当前线程阻塞。通过这种方式，可以更容易地观察到其他线程执行insertIgnore方法是否会被阻塞
5. 在记录不存在的前提下，如果insertIgnore返回的记录数<1，那么再次根据uuid从数据库查询一下记录，然后记录查询结果

```java
@Slf4j
@Service
public class UserService {

    @Autowired
    private TestUserMapper testUserMapper;

    @Transactional(rollbackFor = Exception.class)
    public void addUser(TestUserDO testUserDO) {
        TestUserDO dbTestUserDO = testUserMapper.getByUuid(testUserDO.getUuid());
        log.info("dbTestUserDO is: {}", dbTestUserDO);

        if (dbTestUserDO == null) {
            int count = testUserMapper.insertIgnore(testUserDO);
            log.info("execute insertIgnore results: {}", count);
            if (count > 0) {
                // 这儿休眠20秒，目的是让当前线程阻塞。
                // 通过这种方式，可以更容易地观察到其他线程执行insertIgnore方法是否会被阻塞
                sleep20Seconds();
                log.info("insertIgnore success: {}", testUserDO);
            } else {
                TestUserDO queryAgainTestUserDO = testUserMapper.getByUuid(testUserDO.getUuid());
                log.info("insertIgnore failure, query again: {}", queryAgainTestUserDO);
            }
        }
    }

    private void sleep20Seconds() {
        try {
            Thread.sleep(20 * 1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

该方法就是一个普通的添加用户的方法，这种写法理论上可以实现幂等的特性。那么执行的结果是否和我们的预期一致呢？我们来看看吧！

## 执行结果

首先，我们写出我们的单元测试。

```java
@RunWith(SpringRunner.class)
@ActiveProfiles("dev")
@SpringBootTest
public class UserServiceTest {

    @Autowired
    private UserService userService;

    @Test
    public void testAddUser() throws ExecutionException, InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(2);

        // 生成全局唯一键
        final String uuid = UUID.randomUUID().toString().replaceAll("-", "");

        Future<?> firstFuture = executorService.submit(() -> {
            userService.addUser(new TestUserDO().setName("test").setUuid(uuid));
        });

        Future<?> secondFuture = executorService.submit(() -> {
            userService.addUser(new TestUserDO().setName("test").setUuid(uuid));
        });

        firstFuture.get();
        secondFuture.get();
    }
}
```

执行结果如下所示：

```
2020-01-13 15:42:03,815|DEBUG|pool-10-thread-1|c.h.e.p.s.a.d.d.m.T.getByUuid:159|==>  Preparing: SELECT * FROM test_user WHERE uuid = ? 
2020-01-13 15:42:03,815|DEBUG|pool-10-thread-2|c.h.e.p.s.a.d.d.m.T.getByUuid:159|==>  Preparing: SELECT * FROM test_user WHERE uuid = ? 
2020-01-13 15:42:03,863|DEBUG|pool-10-thread-1|c.h.e.p.s.a.d.d.m.T.getByUuid:159|==> Parameters: 2d20ec2478544d82915414469d8f709c(String)
2020-01-13 15:42:03,863|DEBUG|pool-10-thread-2|c.h.e.p.s.a.d.d.m.T.getByUuid:159|==> Parameters: 2d20ec2478544d82915414469d8f709c(String)
2020-01-13 15:42:03,872|DEBUG|pool-10-thread-1|c.h.e.p.s.a.d.d.m.T.getByUuid:159|<==      Total: 0
2020-01-13 15:42:03,872|DEBUG|pool-10-thread-2|c.h.e.p.s.a.d.d.m.T.getByUuid:159|<==      Total: 0
2020-01-13 15:42:03,872|INFO|pool-10-thread-1|c.h.e.p.s.a.core.service.UserService:24|dbTestUserDO is: null
2020-01-13 15:42:03,873|INFO|pool-10-thread-2|c.h.e.p.s.a.core.service.UserService:24|dbTestUserDO is: null
2020-01-13 15:42:03,873|DEBUG|pool-10-thread-1|c.h.e.p.s.a.d.d.m.T.insertIgnore:159|==>  Preparing: INSERT IGNORE test_user(uuid, `name`) VALUES(?, ?) 
2020-01-13 15:42:03,873|DEBUG|pool-10-thread-2|c.h.e.p.s.a.d.d.m.T.insertIgnore:159|==>  Preparing: INSERT IGNORE test_user(uuid, `name`) VALUES(?, ?) 
2020-01-13 15:42:03,903|DEBUG|pool-10-thread-2|c.h.e.p.s.a.d.d.m.T.insertIgnore:159|==> Parameters: 2d20ec2478544d82915414469d8f709c(String), test(String)
2020-01-13 15:42:03,903|DEBUG|pool-10-thread-1|c.h.e.p.s.a.d.d.m.T.insertIgnore:159|==> Parameters: 2d20ec2478544d82915414469d8f709c(String), test(String)
2020-01-13 15:42:03,905|DEBUG|pool-10-thread-2|c.h.e.p.s.a.d.d.m.T.insertIgnore:159|<==    Updates: 1
2020-01-13 15:42:03,906|INFO|pool-10-thread-2|c.h.e.p.s.a.core.service.UserService:28|execute insertIgnore results: 1
2020-01-13 15:42:23,906|INFO|pool-10-thread-2|c.h.e.p.s.a.core.service.UserService:32|insertIgnore success: TestUserDO(id=29, uuid=2d20ec2478544d82915414469d8f709c, name=test, gmtCreate=null, gmtModified=null)
2020-01-13 15:42:23,910|DEBUG|pool-10-thread-1|c.h.e.p.s.a.d.d.m.T.insertIgnore:159|<==    Updates: 0
2020-01-13 15:42:23,911|INFO|pool-10-thread-1|c.h.e.p.s.a.core.service.UserService:28|execute insertIgnore results: 0
2020-01-13 15:42:23,911|DEBUG|pool-10-thread-1|c.h.e.p.s.a.d.d.m.T.getByUuid:159|==>  Preparing: SELECT * FROM test_user WHERE uuid = ? 
2020-01-13 15:42:23,911|DEBUG|pool-10-thread-1|c.h.e.p.s.a.d.d.m.T.getByUuid:159|==> Parameters: 2d20ec2478544d82915414469d8f709c(String)
2020-01-13 15:42:23,912|DEBUG|pool-10-thread-1|c.h.e.p.s.a.d.d.m.T.getByUuid:159|<==      Total: 0
2020-01-13 15:42:23,912|INFO|pool-10-thread-1|c.h.e.p.s.a.core.service.UserService:35|insertIgnore failure, query again: null
```

我们看到两个线程的insertIgnore的确是串行化执行的，也就是说这儿有行级锁。后执行的线程，一定会等到先执行的线程提交事务之后再执行。但是最后的日志记录却让人感到很意外，为啥查询出的结果是null呢？

后来，我仔细想了一下，可能和Mysql隔离级别有关。在问了运维的同事之后，得知mysql的默认隔离级别为`可重复读`。所谓可重复读，就是在同一个事物里面，两条相同的查询语句，执行的结果是一样的。

再回过头来看我们的代码，我们发现在insertIgnore之前，我们有调用方法getByUuid。当Mysql的隔离级别是可重复读时，在同一个线程里面，如果第一次sql语句查询不到记录，那么第二次相同的sql语句也会查询不到记录。

终于找到原因了！那么如何避免这种问题呢，楼主觉得有以下两个思路：

1. 将事务的隔离级别由可重复读，更改为读提交。可在方法上加上隔离级别的声明，即：
`@Transactional(rollbackFor = Exception.class, isolation = Isolation.READ_COMMITTED)`。
2. 去掉第一次的查询操作

## 总结

虽然问题最终的解决办法很简单，但是最重要的还是排查的过程，思路很重要。
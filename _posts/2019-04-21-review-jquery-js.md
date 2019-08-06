---
layout: post
title:  前端js&jQuery温故而知新
date:   2019-04-21 23:36:00 +0800
categories: 其他
tag: 工具
---

* content
{:toc}

## 前言

最近这三个月，楼主利用空闲时间，一直在帮朋友做一个健康信息监测和管理的系统。该系统主体呈现方式为B/S结构。同时，因为只有我一个开发，所以前端和服务端得一起搞。

才毕业的那两年，前端还比较火，自己做的事情也比较杂，所以在那段时间里，楼主还接触过前端的技术，例如js、jQuery、css和html等等。后面就一直做服务端开发，前端的知识就基本上忘光光了。

为了让自己对项目中的前端知识有一个比较系统的总结，所以楼主决定将项目中接触到的前端知识点和技巧罗列出来。

## FreeMarker基础配置

Spring Boot中要引入FreeMarker需要做几件事情：

1. 引入Maven依赖
2. 配置请求上下文属性
3. 编写ftl模板文件

首先，引入Maven依赖。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-freemarker</artifactId>
</dependency>
```

其次，在`application.yml`中配置请求上下文属性。

```yml
spring:
  freemarker:
    request-context-attribute: rc
```

第三，在`/resources/templates/`下创建相应的`*.ftl`文件。

最后，因为本项目未采用前后端分离的模式，所以静态文件也放置在工程里面里。放置位置为`/resources/static/assets/`，访问方式为`#{${rc.contextPath}/assets/***`。
【注】：如果项目采用了权限控制，需要将静态文件访问的路径添加进白名单。

## FreeMarker基本使用

在`FreeMarker`中，可通过`<#include>`标签添加`*.ftl`的引用。如下所示：

```html
<!--
文件目录结构如下
> templates
    > inc
        > navbar.ftl
        > leftsidebar.ftl
        > rightsidebar.ftl
    > tgt.ftl
-->

<#include "inc/navbar.ftl">
<#include "inc/leftsidebar.ftl">
<#include "inc/rightsidebar.ftl">
```

## Spring-security支持

该项目采用了Spring-security作为用户、权限、角色的管理，同时集成了`FreeMarker`。

1. 引入`spring-boot-starter-security`的依赖。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

2. 创建相应的数据库表。

```sql
CREATE TABLE `authority` (
  `username` varchar(50) NOT NULL COMMENT '用户名',
  `authority` varchar(50) NOT NULL COMMENT '权限，admin：管理员，operator：操作员',
  UNIQUE KEY `username` (`username`) USING BTREE
) COMMENT='权限信息';

CREATE TABLE `user` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键id',
  `username` varchar(50) NOT NULL COMMENT '用户名',
  `password` varchar(50) NOT NULL COMMENT '密码',
  `enabled` tinyint(1) NOT NULL DEFAULT '1' COMMENT '是否可用，1：可用，2：不可用'
  PRIMARY KEY (`id`),
  UNIQUE KEY `username` (`username`)
) AUTO_INCREMENT=2 COMMENT='用户信息';

INSERT INTO `user` VALUES ('1', 'admin', 'admin', '1');
```

3. 在Java代码中增加相关配置。

```java
@Configuration @EnableWebSecurity public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired private DataSource dataSource;

    /**
     * 定义认证用户信息获取来源，密码校验规则等
     */
    @Override protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.jdbcAuthentication().dataSource(dataSource)
            // 指定查询用户SQL
            .usersByUsernameQuery("select username,password,enabled from user where username = ?")
            // 指定查询权限SQL
            .authoritiesByUsernameQuery("select username,authority from authority where username = ?");
    }

    /**
     * 定义安全策略
     */
    @Override protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/assets/**").permitAll() // 定义不需要验证的url
                .antMatchers("/download").permitAll()  //定义不需要验证的url
                .anyRequest().authenticated()          // 其余的所有请求都需要验证
                .and()
                .logout().permitAll()                  // 定义/logout不需要验证
                .and()
                .formLogin().loginPage("/login").permitAll(); // 使用form表单登录
        http.exceptionHandling().accessDeniedPage("/403");
        http.logout().logoutRequestMatcher(new AntPathRequestMatcher("/logout"));
    }
}
```

4. 为了在`*.ftl`中正确地进行权限管理，需要在`公共.ftl`中添加以下属性。

```html
<!-- token header值 -->
<input type="hidden" id="project_token" value="${_csrf.token}"/>
<!-- token header键 -->
<input type="hidden" id="project_headerName" value="${_csrf.headerName}"/>
<!-- 获取用户名 -->
<input type="hidden" id="project_username" 
 value="${Session.SPRING_SECURITY_CONTEXT.authentication.principal.username}"/>
<!-- 获取用户权限 -->
<input type="hidden" id="project_authority"
 value="${Session.SPRING_SECURITY_CONTEXT.authentication.authorities[0].authority}"/>
```

5. 使用Ajax请求服务端时，需要指定特定的请求header。

```js
$.ajax({
    url: "${rc.contextPath}/url",
    type: "POST",
    beforeSend: function(request) {
        request.setRequestHeader($('#project_headerName').val(), $('#project_token').val());
    },
    contentType: "application/json",
    dataType: "json",
    data: data,
    success: function (result) {
        // something todo is here
    },
    error: function () {
        alert("error");
    }
});
```

## 常用的js和jQuery语法

1. 如何包含ico、css和js文件？

css和js作为html的基础，我们往往需要在html中进行引入，引入方式如下：

```html
<!-- 图标 -->
<link rel="icon" href="favicon.ico" type="image/x-icon">
<!-- css文件 -->
<link rel="stylesheet" href="${rc.contextPath}/assets/css/main.css">
<!-- js文件 -->
<script src="${rc.contextPath}/assets/js/jquery.js"></script>
```

2. jQuery初始化方法块

jQuery的使用是从初始化方法块开始的，其使用方式如下：

```html
<script>
$(function () {
    // add jQuery codes here
});
</script>
```

3. jQuery选择器

| 选择器名称 | 选择器语法 |
| --- | --- |
| id选择器 | $("#id") |
| class选择器 | $(".class") |
| 子选择器 |  $("#parent .child") |
| 子选择器 | \$("a", $(this)) |
| checkbox选中选择器 | $("input[class='class']:checked") |

4. ajax基本语法

```js
// 设置body数据
var data = JSON.stringify({
    "id": $("#userId").val()
});

$.ajax({
    url: "url",
    async: false, // optional, 异步标记
    type: "POST",
    beforeSend: function(request) {
        // 设置请求头
        // request.setRequestHeader($('#project_headerName').val(), $('#project_token').val());
    },
    contentType: "application/json",
    dataType: "json",
    data: data,
    success: function (result) {
        if (result.code !== 1000) {
            alert(result.message);
        } else {
            // add codes here
        }
    },
    error: function () {
        alert("error");
    }
});
```

5. jQuery事件

普通的点击事件

```js
$('#btn_id').on('click', function () {
});
```

动态元素的点击事件如下。

```js
$("body").on("click", ".class", function () {
    var $this = $(this);
});
```

同时，在上述代码中， 我们可以通过`$(this)`的方式，在代码块中获取当前jQuery元素。

6. 其他

设置/获取元素值。

```js
// 获取元素值
var data = $("#btn_id").attr("data");

// 设置元素值
$("#btn_id").attr("data", "this is a value");
```

我们可以通过`setInterval()`方法，设置定时执行。

```js
// 每3秒执行一次
setInterval(function () {
    // add codes here
}, 3000);
```

我们可以通过`addClass()`给html标签添加class样式，也可以通过`removeClass`去掉html标签的class样式。

```js
$("#btn_id").addClass('flip');
$("#btn_id").removeClass('flip');
```

js对象的比较。（值比较和类型比较）

+ 两个等号`==`，在类型不同时，会自动进行类型转换，然后表示比较值。
+ 三个等号`===`，则会先比较类型，再比较值，如果类型和值都相等，才返回true，否则返回false。
+ `!=`和`==`意思相反，在类型不同时，会自动进行类型转换。
+ `!==`和`===`意思相反，先比较类型，再比较值，只要其一不等，就返回true，否则返回false。

在jQuery中，我们可以使用`show()`和`hide()`方法，来对元素进行显示和隐藏。还可以使用`remove()`方法来对元素进行删除。

```js
$("#btn_id").show();
$("#btn_id").hide();
$("#btn_id").remove();
```

如果我们想在当前页面跳转到其他页面，可使用以下方式：

```js
location.href = "${rc.contextPath}/user/detail?id=" + $("#userId").val();
```
我们还写了一个简单的日期时间格式化的`prototype`方法。

```js
// 方法定义
Date.prototype.format = function (fmt) { // author: juconcurrent
    var o = {
        "M+": this.getMonth() + 1,                   //月份
        "d+": this.getDate(),                        //日
        "h+": this.getHours(),                       //小时
        "m+": this.getMinutes(),                     //分
        "s+": this.getSeconds(),                     //秒
        "q+": Math.floor((this.getMonth() + 3) / 3), //季度
        "S": this.getMilliseconds()                  //毫秒
    };
    if (/(y+)/.test(fmt))
        fmt = fmt.replace(RegExp.$1, (this.getFullYear() + "").substr(4 - RegExp.$1.length));
    for (var k in o)
        if (new RegExp("(" + k + ")").test(fmt))
            fmt = fmt.replace(RegExp.$1, (RegExp.$1.length == 1) 
                     ? (o[k])
                     :  (("00" + o[k]).substr(("" + o[k]).length)));
    return fmt;
};

// 使用方式
var date1 = new Date().format("yyyy-MM-dd");
var date2 = new Date().format("yyyy-MM-dd hh:mm");

var currentTime = 333333;
var date2 = new Date(currentTime).format("yyyy-MM-dd hh:mm");
```

判断checkbox是否被选中

```js
var isChecked = $('#checkbox').is(":checked");
```

使用`slice()`切分字符串

```js
var date1 = "20190421";

// 这儿会将date1变为`2019-04-21`
var date2 = dat1.slice(0, 4) + "-" + data.slice(4, 6) + "-" + data.slice(6, 8);
```

获取/设置元素的值

```js
// 获取值
var val = $("#input").val();

// 设置值
$("#input").val(val + " suffix");
```

获取/设置/追加元素的内容

```js
// 获取值
var htmlVal = $("#span").html();

// 设置值
$("#span").html(htmlVal + " suffix");

// 追加值
$("#span").append(" added val");
```

简单的数组遍历

```js
var result = {"content": { "items": {"id": 1, "name": "juconcurrent" } } }
var items = result.content.items;

for (var i = 0; i < items.length; i++) {
    var item = items[i];
    var id = item["id"];
    var name = item["name"]
}
```

文件上传，我们一般使用表单的形式，那么在js里面是怎么做到的呢？

```js
// 初始化一个FormData表单对象
var formData = new FormData();

// 将文件塞入表单对象
formData.append("file", $("#file_id")[0].files[0]);

// 文件上传
$.ajax({
    url: "${rc.contextPath}/upload",
    type: "POST",
    beforeSend: function(request) {
        request.setRequestHeader($('#prject_headerName').val(), $('#prject_token').val());
    },
    data: formData,
    processData: false,  // 告诉jQuery不要去处理发送的数据
    contentType: false,   // 告诉jQuery不要去设置Content-Type请求头
    success: function (responseText) {
        if (responseText.code !== 1000) {
            alert(responseText.message);
        } else {
            // add codes here
        }
    }
});
```

## 总结

在本文中，我们的项目使用了Spring Boot + Spring Security + FreeMarker + jQuery + js的设计方式。
基于此项目，我们介绍了项目中用到的知识和技巧，主要偏向于前端。
由于楼主前端知识有限，仅站在使用者角度来看待问题，也更侧重于实践运用。如果读者有在文中发现错误和需要完善之处，请不吝提出。  

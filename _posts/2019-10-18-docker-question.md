---
layout: post
title:  Docker Desktop: Error response from daemon: driver failed programming external connectivity on en...
date:   2019-10-18 23:41:38 +0800
categories: Docker
tag: Docker
---

* content
{:toc}

转载自 [https://www.cnblogs.com/boazy/p/11661277.html](https://www.cnblogs.com/boazy/p/11661277.html)

右击任务栏Docker图标`Restart`或`Quit Docker Desktop`后之前正常的zk1容器不会自动启动。
通过命令`docker start zk1`启动报错如下错误：

```
Error response from daemon: driver failed programming external connectivity on endpoint zk1 (7cfb61e95c9ae834e3339d98574ac96f12ab94659bcf573a2a50204ff38164e6): Error starting userland proxy: /forwards/expose/port returned unexpected status: 500
Error: failed to start containers: zk1
```

解决方法：

1. Quit Docker Desktop（右击任务栏Desktop图标）
2. 进入`Windows任务管理器`，干掉进程`com.docker.backend.exe`进程
3. 过一会`com.docker.backend.exe`进程会自己重新启动好
4. 再打开Docker Desktop（双击桌面图标）
5. 当Docker Desktop启动好后，zookeeper容器也自动成功正常启动完了


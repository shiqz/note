## 前言
缓存层的目的是用来加速响应用户请求的，通常我们数据都是持久化在数据库中的，而直接访问数据获取数据会导致数据库服务奔溃，为此引入缓存来缓解
数据的访问压力。
> 内存读写速度比磁盘块好几个量级，合理使用内存缓存可以有效提高系统性能。

而缓存层通常会存在三个问题：缓存雪崩、缓存击穿、缓存穿透。
![img.png](img.png)

![img_1.png](img_1.png)
# Event Sourcing

阅读[超越“双十一”—— ebay百万TPS支付账务系统的设计与实现](<https://mp.weixin.qq.com/s?__biz=MzA3ODIxNjYxNQ==&mid=2247488215&idx=1&sn=c7ba153f9f602f2713f56cd9498dd926&chksm=9f477e1fa830f709c4f8f530b2577bac3a781556b3e8dbb2e34ad2fb890e2a7f8670cfecc86f&scrolltodown=1&key=a4f2e1c535176118cb99261cb9a5b2a7ac379830912c9b7800f4a0fb7d04f19ea9072bf1a41a1292035adb9690008dfb4c285423dab75929c3f4a844724dbdab14dd2394dc336d8910a098f6b5103227&ascene=1&uin=MjE5NzcwNTQ0MA%3D%3D&devicetype=Windows+10&version=62070152&lang=zh_CN&pass_ticket=uSOxCGD7jEvomTkSjwpXI%2BaDU37KQ7D%2Bf95mHgOBWm%2BmHOqS6bPd6P6frRgQGZ8c>)，看到Event Sourcing，和最近公司的新人的设计思路很像，便有了此文

## 1, 什么是Event Sourcing

Event Sourcing 也叫事件溯源，是由Martin Fowler提出的一种架构模式。

- 整个系统以事件为驱动，所有业务都由事件驱动来完成
- 事件是一等公民，系统的数据都以事件为基础，时间需要保存在数据库（Event Store）中
- 业务数据只是一些由时间产生的视图



聚合对象

聚合资源库

视图

​	将聚合对象的数据状态物化视图的形式保存

Event Handler: 事件处理器，监听相应事件，更新物化视图

事件重现

## 2， CQRS

俗称：读写分离

## 3，Event Sourcing 的优点与缺点

> 优点

- 方便进行溯源与历史重新
  - 在业务复杂的系统总，通过保存对事件的分析，知道现在系统状态是如何变更的
- 方便Bug修复
- 性能优越
  - 事件表只有写操作，没有更新。
  - 聚合对象根据事件的ID聚合，调用业务方法，最后由另一个Event Handler处理这些事件，更新物化视图的表

> 缺点

- 需要熟悉领域驱动设计，基于事件的响应式编程
- 事件结构的改变

![微软事件驱动](https://i.loli.net/2019/11/10/8S5YFTNUrapVKnD.png)

## 4，基于Event Sourcing的分布式系统的思考

生成事件的程序与处理事件的程序是分离的，解耦，可以通过MQ的方式分离，订阅

### 使用的场景

- 当想要获取数据的意图，目的或者原因
- 最小化的避免数据更新冲突
- 需求发生变化时，能够灵活的更改实例化模型和实体数据的格式
- 与CQRS结合使用时，更新读取模型时的最终一致性是可以接受时。

场景举例：

会议管理系统

12306购票



## 5，文献参考

- [微软Event Sourcing](https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing )

- [Event Sourcing 和 CQRS 落地（一）：UID-Generator 实现](https://www.infoq.cn/article/JmiZRu85W7i4W-HsoX5t)

- [极客时间：我对CQRS/EventSourcing架构的思考](https://mp.weixin.qq.com/s?__biz=MzI4MTY5NTk4Ng==&mid=2247489581&idx=1&sn=cbcf52fe80e03e1865289d2ed26e81c0&chksm=eba41bb0dcd392a675e82b3f0783838058b10bf400a072fdfccb260f499a5e64460a02e334c8&scene=27#wechat_redirect)

  


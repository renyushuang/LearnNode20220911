2 Handler原理是什么，简单说下

1 为什么不能在子线程更新UI？

2一个线程有几个looper? 如何保证，又可以有几个Handler

3 handler内存泄漏的原因，其他内部类为什么没有这个问题

4为什么主线程可以new Handler 其他子线程可以吗 怎么做

5 Message可以如何创建？哪种效果更好，为什么？

 6 Handler中的生产者-消费者设计模式你理解不？

7 子线程中维护Looper在消息队列无消息的时候处理方案是怎么样的

8 既然存在多个Handler往MessageQueue中添加数据（发消息时各个Handler处于不同线程），内部如何保证安全

9 我们使用Message是应该如何创建它

10 Looper死循环为什么不会导致线程卡死

11 关于ThreadLocal，谈谈你的理解？

11 使用Hanlder的postDealy()后消息队列会发生什么变化？

12 为什么不能在子线程更新UI？




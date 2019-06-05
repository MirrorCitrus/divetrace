[TOC]

# 20190605 线程和IO模型

- thread loop的几种形式
    - BusyWait
    - 等待用户输入/等待消息
    - 等待IO
- 线程的生产者消费者模型
    - 生产者/MessageQueue/消费者
    - 扩展
        - 事件队列
        - 缓冲队列
        - 一读多写，多读多写
        - 流量控制
    - 线程池
        - 基本组成：线程个数、空闲时间
        - 任务队列/生产者（递交任务）/消费者（线程）
        - 惊群效应
- IO模型
    - 阻塞IO => 非阻塞IO
    - 问题
        - 忙等 => IO多路复用
        - 单线程不响应新请求问题 => 线程池+请求队列
    - IO多路复用
        - select, poll, epoll
        - select处等待事件

## Reference
- [x] [关于线程和I/O模型的极简知识](https://mp.weixin.qq.com/s/qodCngOPXGSaaBy2ULAgqg?utm_source=androidweekly.io&utm_medium=website)
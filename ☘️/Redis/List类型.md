# List列表
> 相关笔记：[[Redis|Redis 知识总结]]


# 命令

![image](image-20260414224947-12c1bw4.png)

# 编码方式

以前Redis对List有两种编码方式，一种是ziplist压缩列表，另一种是linkedlist链表，使用逻辑和哈希的ziplist和hashtable是一样的：具体参考[[Redis/哈希类型|哈希类型]]

Redis 现采用了quicklist的编码方式，取消了ziplist和linkedlist，相当于将ziplist的压缩特性和linkedlist的链表连接特性结合起来

![image](image-20260414232709-qc8brli.png)

# 应用方式

- Redis 通常用list作为“数组”这样的结构，来存储多个元素，如

![image](image-20260414233559-xgminun.png)

- 作为消息队列，经典的模型就是生产者-消费者模型。消费者brpop，达到轮询的效果

  谁先执行到brpop命令，就谁先拿到这个请求，接下来又轮到消费者2、3。然后接着就是1、2、3...

![image](image-20260414233646-eo2ja7c.png)

brpop是block right pop，阻塞弹出，当列表为空时，就阻塞（相当于消费者在准备去“抢”这个新元素）。一直等到列表有新元素就push该元素，同时返回元素的内容。

- 另外还有分频道的消息队列，所谓分频道就是在 Redis中有多个列表

  这种多频道，就可以在某种数据发生问题的时候也不会影响到其他数据（低耦合）

  同时消费者也可以读取多个频道（读取多个key）

![image](image-20260414234108-hzn7aou.png)

# PineLine

叫做流水线或者叫管道

假设列表里有很多元素，你需要循环这个列表，每一次都去取列表里的键值对元素的话，就要执行o（N）次的hgetall key，也就是很多次的网络请求。如果请求非常多，那就会导致阻塞

pineline的作用就是将多次的命令合并成一次的网络请求再进行通信。

‍

选择列表类型时，请参考：

同侧存取：（lpush + lpop）/ （rpush + rpop）为栈

异侧存取：（lpush + lrpop）/ （rpush + lpop）为队列

‍

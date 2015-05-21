---
layout: post
title: "netlink bulk dump"
date: 2013-03-07 00:00
comments: true
categories: [netlink]
tags: [kernel, netlink]
---

```c
 93 struct netlink_callback {
 94         struct sk_buff          *skb;
 95         const struct nlmsghdr   *nlh;
 96         int                     (*dump)(struct sk_buff * skb,
 97                                         struct netlink_callback *cb);
 98         int                     (*done)(struct netlink_callback *cb);
 99         void                    *data;
100         /* the module that dump function belong to */
101         struct module           *module;
102         u16                     family;
103         u16                     min_dump_alloc;
104         unsigned int            prev_seq, seq;
105         long                    args[6];
106 };
```
```c
117 struct netlink_dump_control {
118         int (*dump)(struct sk_buff *skb, struct netlink_callback *);
119         int (*done)(struct netlink_callback *);
120         void *data;
121         struct module *module;
122         u16 min_dump_alloc;
123 };
```

有时候我们需要从内核输出大量的消息。 例如，dump interface, xfrm sa,sp（几千甚至几万条）等 这些信息显然无法放到一个skb里。

这是我们需要借助netlinkcallback机制。 原理： 1.首先被dump的数据要支持两个函数 a. dump: 每次输出时调用，接着上次的数据输出。如果全部输出完成返回0. b. done: 全部输出完成后被调用。

当dump的消息非常多时候，首先创建struct netlink_callback, 并创建这个cb挂到netlink socket(nlk)上。 此处的nlk是提出dump请求的那个socket。 调用dump函数输出第一次结果， 并将结果放到放到nlk的接受队列里，激发dataready。

因此应用程序这时rcvmsg就会返回。并得到第一次的输出结果， 在rcvmsg的系统调用再次netlink_dump。

这样应用程序每次通过系统调用rcv, 在将数据从内核中收上来的， 同时，这个系统调用rcv也激发的一次netlink_dump,新的数据被 追加到了socket的接受队列里。 直到所有的数据dump完成，cb->dump(skb, cb);返回0 （见line1747 in netlink_dump) 内核调用 cb->done(cb), 并将cb从netlink socket上删除并释放对应的内存。 nlk->cb = NULL; netlink_consume_callback(cb);


---
layout: post
title: "vhost net study"
date: 2015-05-21 17:41:07 +0800
comments: true
categories: 
---

vhost net 的目的是为了避免在host kerne上做一次qemu的调度，提升性能。
xmit: 让vm的数据报在 host的内核就把报文发送出去。
rcv：

### 核心数据结构
#### `vhost_poll`是vhost里最关键的一个数据结构。

```
 27 /* Poll a file (eventfd or socket) */
 28 /* Note: there's nothing vhost specific about this structure. */
 29 struct vhost_poll {
 30         poll_table                table;
 31         wait_queue_head_t        *wqh;
 32         wait_queue_t              wait;
 33         struct vhost_work         work;
 34         unsigned long             mask;
 35         struct vhost_dev         *dev;
 36 };
```

* table:每次负责把wait域放倒wqh里。vhost_net_open将它的执行函数vhost_poll_func
* wqh：它的wqh被初始化指向一个eventfd的ctx，
* wait：每次把wait放倒这个wqh链表里，当guest vm的发送报文时，wait被摘下，
并执行其对应的func,vhost_net_open将该func被初始化为vhost_poll_wakeup。
vhost_poll_wakeup负责将work放入对应vhost_dev下的work_list链表中。
* work: 每个vhost_dev有一个thread，负责从work_list链表里的摘除work节点，
并执行work节点对应的fn. fn是真正干活的的函数。 
	对于rx vhost_virqueue, vhost_net_open将该fn初始化为handle_rx_kick
	对于tx vhost_virqueue, vhost_net_open将该fn初始化为handle_tx_kick
	对于rx vhost_virqueue, vhost_net_open将该fn初始化为
handle_rx_kick.
* mask:是需要监听的eventfd的事件集合
* dev: 该vhost_poll对应的vhost_dev;

以guest VM发送一个报文为例：
```c
file: drivers/net/virtio_net.c
> .ndo_start_xmit      = start_xmit,
> >  static netdev_tx_t start_xmit(struct sk_buff *skb, struct net_device *dev)
> > > err = xmit_skb(sq, skb);
> > > > return virtqueue_add_outbuf(sq->vq, sq->sg, num_sg, skb, GFP_ATOMIC);
> > > > > return virtqueue_add(vq, &sg, num, 1, 0, data, gfp);
> > > virtqueue_kick(sq->vq);
> > > > virtqueue_notify(vq);
> > > > > vq->notify(_vq)
> > > > > >  iowrite16(vq->index, (void __iomem *)vq->priv);
```

前端驱动最终guest vm因io操作造成vm exit. host vm 调用`vmx_handle_exit`
处理这个io 请求。
```c
> static int vmx_handle_exit(struct kvm_vcpu *vcpu)
> > kvm_vmx_exit_handlers[exit_reason](vcpu);
> > > kvm_vmx_exit_handlers:  [EXIT_REASON_IO_INSTRUCTION]          = handle_io,
> > >  static int handle_io(struct kvm_vcpu *vcpu)
> > > >  return kvm_fast_pio_out(vcpu, size, port);
> > > > > static int emulator_pio_out_emulated(struct x86_emulate_ctxt *ctxt, int size, unsigned short port, const void *val, unsigned int count)
> > > > > > emulator_pio_in_out
> > > > > > kernel_pio(struct kvm_vcpu *vcpu, void *pd)
> > > > > > > int kvm_io_bus_write(struct kvm_vcpu *vcpu, enum kvm_bus bus_idx, gpa_t addr,
> > > > > > > > __kvm_io_bus_write
> > > > > > > > > dev->ops->write(vcpu, dev, addr, l, v) 
> > > > > > > > > dev->ops is ioeventfd_ops 相当ioeventfd_write
> > > > > > > > > > eventfd_signal(p->eventfd, 1);
> > > > > > > > > > > wake_up_locked_poll(&ctx->wqh, POLLIN);
```

vhost net init 
```
> vhost_net_open
> > vhost_dev_init
> > > vhost_poll_init(&vq->poll, vq->handle_kick, POLLIN, dev);
> > > > init_waitqueue_func_entry(&poll->wait, vhost_poll_wakeup);
> > > > init_poll_funcptr(&poll->table, vhost_poll_func);
> > > > vhost_work_init(&poll->work, fn);
> > vhost_poll_init(n->poll + VHOST_NET_VQ_TX, handle_tx_net, POLLOUT, dev);
> > vhost_poll_init(n->poll + VHOST_NET_VQ_RX, handle_rx_net, POLLIN, dev);
> > f->private_data = n;
```

#### `vhost_worker`
vhost set owner: 为每个vhost_dev,启动一个进程，该进程函数是vhost_worker，
它反复从该dev下的work_list链上取一个节点下来，
（该节点类型是vhost_work), 执行该节点下的fn函数。
每个dev下有个双向链work_list, 链中的每个节点是一个vhost_work
`vhost_worker`顺序摘下一个节点，执行这个节点下的fn函数。
对于vhost_virtqueue RX/TX来说，这个fn就是`handle_rx_kick,handle_tx_kick.

```
> vhost_net_set_owner
> > vhost_dev_set_owner
> > > worker = kthread_create(vhost_worker, dev, "vhost-%d", current->pid);
> > > dev->worker = worker;
> > > wake_up_process(worker);
```

每个vhost_virqueue下有一个vhost_poll。
VHOST_SET_VRING_KICK，将vhost_poll下的wqh(wait_queue_head_t类型)指向eventf下对应
的wqh，并将vhost_poll下的wait挂在wqh下。
#### poll->wqh被赋值指向 eventfd对应的eventfd_ctx下的wqh, 这样一个virqueue下的poll就跟eventfd关联上了!!!
最终poll的wait被放入到了eventfd的wqh链表里。

`vhost_net_ioctl VHOST_SET_VRING_KICK` 
```
> vhost_vring_ioctl
> > case VHOST_SET_VRING_KICK:
> > > eventfp = f.fd == -1 ? NULL : eventfd_fget(f.fd);
> > > vq->kick = eventfp
> > > vhost_poll_start(&vq->poll, vq->kick);
> > > > mask = file->f_op->poll(file, &poll->table); 此处file是vq->kick, 即vq对应的eventfd
> > > > 所以相当于调用 eventfd_poll
> > > > > poll_wait(file, &ctx->wqh, wait);
> > > > vhost_poll_wakeup(&poll->wait, 0, 0, (void *)mask);
> > > > > vhost_poll_queue(poll);
> > > > > > vhost_work_queue(poll->dev, &poll->work);
> > > > > > > list_add_tail(&work->node, &dev->work_list);
> > > > > > > wake_up_process(dev->worker);
> > > > > > > > static int vhost_worker(void *data)
> > > > > > > > > work = list_first_entry(&dev->work_list, struct vhost_work, node);
> > > > > > > > > work->fn(work);
```

eventfd在vhost net里被用来处理从vm发出的报文。
以vhost_dev_net 下的rx virqueue为例。 
poll是该virqueue下的poll。
```
> eventfd_poll(struct file *file, poll_table *wait) file:vrqueue使用的eventfd. wait virqueue下的tx poll对应的poll_table
> > poll_wait(file, &ctx->wqh, wait); 
> > >  p->_qproc(filp, wait_address, p); 
> > > 相当于 vhost_poll_func(filp, wait_address, p)
p为poll table。
 _qproc: 是vhost_poll_func。通过vhost_dev_init入口，被init_poll_funcptr(&poll->table, vhost_poll_func);  
wait_address: 
```
vhost_poll_func:每次都将vhost_poll下的wait放倒vhost_poll下的wqh链表里（即evetnfd下的ctx的wqh链表里）。
以为它被初始化为vhost_poll下的poll_table的函数指针，因此对vhost_poll下的poll_table调用poll_wait的时候，
相当于对vhost_poll调用vhost_poll_func。

vhost_poll_wakeup 将传入的wait转换为一个vhost poll
将该poll下work挂在到poll对应的vhost_dev.
```
> vhost_poll_wakeup
> > vhost_poll_queue(poll);
> > > vhost_work_queue(poll->dev, &poll->work);
> > > > list_add_tail(&work->node, &dev->work_list);
> > > > wake_up_process(dev->worker);
```

```
117 static unsigned int eventfd_poll(struct file *file, poll_table *wait)
118 {
119         struct eventfd_ctx *ctx = file->private_data;
120         unsigned int events = 0;
121         u64 count;
122
123         poll_wait(file, &ctx->wqh, wait);
124         smp_rmb();
125         count = ctx->count;
126
127         if (count > 0)
128                 events |= POLLIN;
129         if (count == ULLONG_MAX)
130                 events |= POLLERR;
131         if (ULLONG_MAX - 1 > count)
132                 events |= POLLOUT;
133
134         return events;
135 }
```
```
 42 static inline void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)
 43 {
 44         if (p && p->_qproc && wait_address)
 45                 p->_qproc(filp, wait_address, p);
 46 }
```

TODO1: Why `vp->notify` is `vp_notify`
```
drivers/virtio/virtio_pci_modern.c
315 static struct virtqueue *setup_vq(struct virtio_pci_device *vp_dev,
316                                   struct virtio_pci_vq_info *info,
317                                   unsigned index,
318                                   void (*callback)(struct virtqueue *vq),
319                                   const char *name,
320                                   u16 msix_vec)
...
353         /* create the vring */
354         vq = vring_new_virtqueue(index, info->num,
355                                  SMP_CACHE_BYTES, &vp_dev->vdev,
356                                  true, info->queue, vp_notify, callback, name);
```

TODO2: why vm exit call `vmx_handle_exit`

TODO3: why dev->ops->write 相当

TODO4: "wake_up_locked_poll add_wait_queue 为例说明wait list的使用。
__wake_up_common "
```
wake_up_locked_poll
> __wake_up_locked_key
> >  list_for_each_entry_safe(curr, next, &q->task_list, task_list) {
> > > curr->func(curr, mode, wake_flags, key)
```


---
layout: post
title: linux kernel专题：swappiness参数解析
categories: [操作系统]
description: linux kernel swappiness参数解析
keywords: 操作系统
---

# 解释
- swappiness参数是一个内核参数，用于改变匿名内存空间（anonymous memory：包含堆栈等通过malloc或new生成的内存）与文件页缓存（file-backed memory）的在发生swap时的交换配比。
- swappiness取值区间为0-100，取值越低，表示内核将越少地swap，取值越高，内核将积极的使用swap分区。


# swappiness对内核回收机制的影响
- 由于swappiness参数决定了回收程序运行时内存和页缓存两者的平衡比，所以一个明确的分配机制是有必要的。
## swap_tendency参数
- swap_tendency计算公式：
```
	swap_tendency = mapped_ratio/2 + distress + vm_swappiness;
```
### distress
- distress参数描述了内核回收内存时的困难度。内核第一次决定开始回收页时，distress被置为0，当请求回收的数量增加时，值也会增加，直至到达最大值100。
### mapped_ratio
- mapped_ratio参数描述了系统全部的映射的内存空间与所分配的内存空间的比例
### vm_swappiness
- vm_swappiness就是内核swappiness参数，默认60
### swap_tendency影响机制
当swap_tendency<100时，内核只会回收页缓存类型的内存。一旦它的值大于100，一些属于进程的地址空间的页也会被考虑回收至swap分区。所以，如果distress值小且swappiness被设为60，系统不会交换进程内存直到内存的80%比例被分配。如果用户不希望看到进程内存被交换，可以将swappiness设置为一个很低的值，例如5或10，这会使得内核忽略进程内存的交换直到distress值变得非常高


# 总结
- 提高swappiness参数可以使系统更倾向于利用swap分区，从而给文件缓存留下更多的内存空间；减少该值会使系统更少地使用swap分区，从而提高进程性能
- swappiness参数在不同的应用场景下对系统性能的影响是不同的，对于需要找到符合应用场景的参数值，可以通过每次改变一个小值，测试性能表现最好的值范围。

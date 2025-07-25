---
title: "golang八股整理"
description: 
date: 2025-01-03T23:25:36+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: true
tags:   
     - go
categories:
     - go
---

#  GMP模型

G是goroutine是go协程的抽象，是用户级线程，比线程更轻量，有自己的运行栈及任务函数，要绑定到P才能被执行

M是操作系统线程，M数量与CPU核心数有关，M不直接执行P，而是先和P绑定，M必须绑定P才能执行G

P是go调度器，管理G的本地队列，负责将G分配给M执行，P数量默认为CPU核心数，但能通过`GOMAXPROCS`设置，决定了G最大并行数量，超过CPU核数无意义

对M而言，P是其执行代理，为M提供必要信息的同时，对其隐藏细节，由P承上启下实现了G与M的结合

**协程-线程模型与GMP模型对比**

协程：线程=M：1     ---- 无法并行，一个协程失败，全部停止

GMP = M:N 

M个goroutine与N个线程通过go调度器来实现并行，栈大小会动态扩展

**线程-协程-goroutine对比**

|           | 弱依赖内核 | 可并行 | 可应对阻塞 | 可动态扩展 |
| --------- | ---------- | ------ | ---------- | ---------- |
| 线程      | ×          | √      | √          | ×          |
| 协程      | √          | ×      | ×          | ×          |
| goroutine | √          | √      | √          | √          |

**GMP执行过程**

1. P会先将本地队列中的G与M绑定运行
2. 本地队列中的G全部运行完毕之后，会从全局队列中获取（此时会加锁）
3. 本地队列与全局队列都没有G时，会从别的P的本地队列中偷取（会加锁）

通过这种机制实现弱加锁运行

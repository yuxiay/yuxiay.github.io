---
title: "go面经整理1"
description: 
date: 2025-04-06T23:25:36+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags:   
     - Golang八股面经
categories:
     - Golang八股面经
---

## go后端面经

### go语言

#### make 和 new有什么区别

make是声明一块内存，而new是指向该内存的指针，make只能用来创建切片，通道等类型，但new 没有类型限制

#### defer的执行时机

后进先出，栈结构

#### defer 常见的用法

错误恢复，资源释放，记录执行时间

#### panic 怎么处理

可以用defer进行回滚

#### 协程发生阻塞的情况有哪些？

channel阻塞， 同步IO操作，同步锁，死锁

#### channel满了 消费者和生产者会怎么样。 对值为nil的channel读取会发生什么？

1. 当channel满时，生产者的发送操作会被阻塞，直到有空间可用。
2. 消费者在channel为空时会被阻塞，直到有数据发送。
3. 对nil的channel进行读写操作会导致永久阻塞，引发死锁。
4. 解决方法包括使用带缓冲的channel、正确初始化channel、使用select超时或context取消等

#### map的底层结构是什么样的

Golang的map就是使用哈希表作为底层实现，**map 实际上就是一个指针，指向hmap结构体**。

#### map是并发安全的吗？ sync.Map

map不是并发安全，但sync.Map是并发安全

#### map的遍历是有序的还是无需的？ 如果需要实现有序的遍历如何做（不知道这个有啥意义）

map的遍历是无序的，如果需要实现有序遍历，可以放入切片中

#### GMP是什么？介绍一下。M和P的数量是怎么指定的

GMP是go的一种调度机制，G指的是gorountine，M指的是内核线程，P指的是处理器，M的数量是go语言有个默认的最大数量，P是通过环境变量进行指定，都有一个函数进行指定

#### 协程什么情况下会退出

 自然执行完毕，发生panic未恢复，主动退出，通道关闭或阻塞，上下文取消，父协程退出

#### 如何实现协程池

结构体中包含：一个通道，一个等待，一个协程的数量

```go
type Pool struct {
	taskQueue   chan Task          // 任务队列
	workerNum   int                // 协程数量
	wg          sync.WaitGroup     // 等待所有协程完成
	ctx         context.Context    // 控制上下文
	cancel      context.CancelFunc // 取消函数
}
```



#### GC是什么？ 什么时候会发生GC

GC是垃圾回收，定期触发，手动触发，空间不足时，根对象失效时

讲讲数组和切片的区别

什么时候用数组什么时候用切片

讲讲defer 执行顺序 注意事项

讲讲chanal和goroutine 底层、使用

什么时候chanal会发生panic

### Redis

#### Redis常见数据结构

字符串，列表，哈希，集合，有序集合

#### Redis key的删除策略

Redis 键的删除分为 **过期键删除** 和 **内存淘汰策略**

#### 如果有一大批redis命令 怎么优化

- 批量操作：使用MGET/MSET
- 管道：使用管道进行打包发送
- 是否使用事务

#### redis实现分布式锁

```
SETNX lock_key unique_value  # 成功返回 1，失败返回 0
# 释放锁（需确保原子性）
DEL lock_key
```

### MySQL

#### MySQL的聚集索引和非聚集索引的区别

聚集索引通过主键，非聚集索引是通过其它值

#### 如何分析MySQL执行计划

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 100;
```



#### MySQL怎么实现乐观锁和悲观锁

乐观锁假设冲突不常发生，所以在读取数据时不加锁，但在更新时检查数据是否被其他事务修改过。通常使用版本号或时间戳来实现。而悲观锁则相反，它假设冲突会发生，所以在读取数据时就加锁，防止其他事务修改，直到当前事务结束。

### 计网操作系统等

#### **、OSI 七层模型**

OSI（Open Systems Interconnection）模型是理论上的网络通信分层框架，分为 **7 层**，每层负责特定功能：

|     **层级**      |                **核心功能**                 |      **典型协议/设备**       |
| :---------------: | :-----------------------------------------: | :--------------------------: |
|   **7. 应用层**   | 直接为用户提供服务（如 Web 请求、邮件收发） |     HTTP, FTP, SMTP, DNS     |
|   **6. 表示层**   |     数据格式转换（如加密、压缩、编码）      |    SSL/TLS, JPEG, Base64     |
|   **5. 会话层**   |     管理会话（如建立、维护、终止连接）      |         RPC, NetBIOS         |
|   **4. 传输层**   |     提供端到端通信（可靠性、流量控制）      |           TCP, UDP           |
|   **3. 网络层**   |         数据包路由（IP 寻址、转发）         |       IP, ICMP, 路由器       |
| **2. 数据链路层** |    可靠传输帧（MAC 地址寻址、错误检测）     | Ethernet, Wi-Fi, ARP, 交换机 |
|   **1. 物理层**   |      比特流传输（物理介质、信号传输）       |      光纤, 双绞线, 网卡      |

------

#### **二、TCP/IP 模型**

TCP/IP 是实际广泛应用的简化四层模型，与 OSI 的对应关系如下：

|    **层级**    | **对应 OSI 层级** |             **核心协议**             |
| :------------: | :---------------: | :----------------------------------: |
|   **应用层**   |    OSI 5-7 层     |         HTTP, FTP, DNS, SMTP         |
|   **传输层**   |     OSI 4 层      |               TCP, UDP               |
|   **网络层**   |     OSI 3 层      |         IP, ICMP, OSPF, BGP          |
| **网络接口层** |    OSI 1-2 层     | Ethernet, Wi-Fi, PPP, 交换机, 路由器 |

------

### **三、核心协议详解**

#### **1. TCP（Transmission Control Protocol）**

- **特性**：面向连接、可靠传输、流量控制、拥塞控制。

- 

  三次握手（建立连接）

  复制

  ```text
  客户端 → SYN → 服务器  
  服务器 → SYN-ACK → 客户端  
  客户端 → ACK → 服务器  
  ```

  四次挥手（终止连接

  复制

  ```text
  客户端 → FIN → 服务器  
  服务器 → ACK → 客户端  
  服务器 → FIN → 客户端  
  客户端 → ACK → 服务器  
  ```

- **应用场景**：文件传输（FTP）、网页浏览（HTTP）、邮件（SMTP）。

#### **2. UDP（User Datagram Protocol）**

- **特性**：无连接、不可靠传输、低延迟、高效。
- **应用场景**：实时音视频（Zoom）、DNS 查询、在线游戏。

#### **3. HTTP/HTTPS**

HTTP 协议

- **无状态**：每次请求独立（通过 Cookie/JSESSIONID 维持状态）。

- **方法**：GET（获取数据）、POST（提交数据）、PUT（更新）、DELETE（删除）。

- 

  状态码

  - `200 OK`：成功
  - `301 Moved Permanently`：永久重定向
  - `404 Not Found`：资源不存在
  - `500 Internal Server Error`：服务器错误

- **HTTPS**：HTTP + SSL/TLS 加密，通过证书（CA 签发）实现双向认证。

#### **4. IP 协议**

- **IPv4**：32 位地址（如 `192.168.1.1`），子网掩码划分网络。
- **IPv6**：128 位地址（如 `2001:0db8::1`），解决地址耗尽问题。
- **ICMP**：网络诊断工具（如 `ping` 使用 ICMP Echo Request/Reply）。


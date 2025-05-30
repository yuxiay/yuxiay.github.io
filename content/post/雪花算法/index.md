---
title: "雪花算法"
description: 
date: 2024-10-02T21:14:10+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags:   
categories:
---

## nowflake算法介绍

雪花算法是由64位整数组成的分布式ID，性能高

1. 第一位   占用1bit，其值始终为0，无实际意义

2. 时间戳  占用41位，单位为毫米，可以从开始时间一直持续下去，总共可以容纳约69年时间

3. 工作机器id  占用10位，为项目的工作机器，其中高位5bit是数据中心ID，低位5bit是工作节点ID

4. 序列号  占用12bit，用来记录同毫秒内产生不同的id号

   ```go
   package main
   
   import (
   	"fmt"
   	"time"
   
   	"github.com/bwmarrin/snowflake"
   )
   
   var node *snowflake.Node
   
   // init函数 初始化全局node节点
   // startTime 开始时间    machineID 工作节点
   func Init(startTime string, machineID int64) (err error) {
   	var st time.Time
   	//获取开始时间的时间戳
   	st, err = time.Parse("2006-01-02 15:04:05", startTime)
   	if err != nil {
   		return
   	}
   
   	//初始化开始时间
   	snowflake.Epoch = st.UnixNano() / 1000000
   	//拿到机器id，生成node节点
   	node, err = snowflake.NewNode(machineID)
   	return
   }
   
   // 将节点转化为64位int类型
   func Gen() int64 {
   	return node.Generate().Int64()
   }
   func main() {
   	err := Init("2020-07-01", 1)
   	if err != nil {
   		fmt.Println("init failed, err: %v\n", err)
   	}
   	id := Gen()
   	fmt.Println(id)
   }
   
   ```


---
title: "docker初步认识"
description: 
date: 2025-04-10T19:04:06+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags:   
     - docker
categories:
      - docker
---

# docker初步认识

## 常见命令

```
#镜像相关
docker search  #检索
docker images  #列表
docker pull	   #下载
docker rmi	   #删除

#容器相关
docker run	   #运行
docker ps	   #查看
docker stop    #停止
docker start   #启动
docker restart #重启
docker stats   #状态
docker logs	   #日志
docker exec    #进入
docker rm 	   #删除
```

### run细节

```
docker run -d --name mynginx -p 80:80 nginx
-d 后台启动
--name xxx 声明容器名字xxx
-p 80:88  端口映射 在不同容器中88可以重复
```

### 保存镜像

```
docker commit	#提交
docker save		#保存
docker load		#加载
```

### 分享社区

```
docker login	#登陆
docker tag		#命名
docker push		#推送
```

## 数据存储

### 目录挂载

docker允许将内部目录挂载到宿主机某一目录下

```
docker run -d -p 80:80 -v /app/nghtml:/usr/share/nginx/html --name mynginx nginx

-v 宿主机目录：容器内目录
```

### 卷映射








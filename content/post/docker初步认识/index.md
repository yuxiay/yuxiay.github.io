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

## 数据存储与网络

### 目录挂载

docker允许将内部目录挂载到宿主机某一目录下

```
docker run -d -p 80:80 -v /app/nghtml:/usr/share/nginx/html --name mynginx nginx

-v 宿主机目录：容器内目录
```

### 卷映射

当内部有文件时，进行目录挂载会将内部文件重新从外部进行初始化，这样内部文件就会被外部进行覆盖，如果内部本身有文件，而外部目录无文件时，会导致内部文件被覆盖丢失

这时我们采用卷映射

```
-v ngconf:/etc/nginx
卷名 ngconf
```

docker 默认将目录放在

```
/var/lib/docker/volumes/<volume-name>
```

卷操作

```
docker volume ls					列出卷
docker volume create volume-name	创建卷
docker volume inspect volume-name	查看卷的详情
注意：删除容器时，卷不会跟随删除
```

### 自定义网络

查看容器的细节

```
docker container inspect container-name
```

容器在内部有内部网络，docker为每个容器分配唯一ip，使用容器ip+容器端口可以互相访问，但ip由于各种原因可能会变化，docker0默认不支持主机域名，此时我们可以创建自定义网络，容器名也就是稳定域名

```
docker network ls
docker network create network-name
```

创建在自定义网络上生成的容器

```
docker run -d -p 80:80 --name mynginx 
--network network-name nginx
```

同一自定义网络下，彼此容器进行访问时，通过域名进行访问，域名即容器名

```
curl http://mynginx:80
```



## docker应用实现

### 如何启动一个容器？

- 网络相关。考虑端口，是否需要暴露端口让外界访问，加入自定义网络
- 存储相关。容器是否有什么配置文件或者数据需要挂载在外面方便修改或者防止丢失
- 环境变量。看官方文档是否需要添加环境变量

### Redis主从集群

主机实现：

```
docker run -d -p 6379:6379 \
-v /app/rd1:/bitnami/redis/date \
-e REDIS_REPLICATION_MODE=master \
-e REDIS_PASSWORD=123456 \
--network mynet --name redis01 \
bitnami/redis
```

从机实现：

```
docker run -d -p 6380:6379 \
-v /app/rd2:/bitnami/redis/date \
-e REDIS_REPLICATION_MODE=slave \
-e REDIS_MASTER_HOST=redis01
-e REDIS_MASTER_PORT_NUMBER=6379 \
-e REDIS_MASTER_PASSWORD=123456 \
-e REDIS_PASSWORD=123456 \
--network mynet --name redis02 \
bitnami/redis
```

#### MySQL实例

```
docker run -d -p 3306:3306 \
-v /app/myconf:/etc/mysql/conf.d \
-v /app/mydata:/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=123456 \
--name mysql \
mysql:8.0.37-debian
```

## docker-compose文件

```
docker compose up -d		上线
docker compose down			下线
docker compose start x1 x2	启动
docker compose start x1 x2	停止
docker compose scale x2=3	扩容
```

顶级元素：

- name	名字
- services    服务
- networks  网络
- volumes    卷
- configs       配置
- secrets       密钥	

```yaml
name: myblog
services:
	mysql:
		container_name: mysql
		image: mysql:8.0
		ports: 
			- "3306:3306"
		environment:
			- MYSQL_ROOT_PASSWORD=123456
			- MYSQL_DATABASE=wordpress
		volumes:
			- mysql-data:/var/lib/mysql
			- /app/myconf:/etc/mysql/conf.d
		restart: always
		networks:
			- blog
			
	wordpress:
		container_name: wordpress
		image: wordpress:latest
		ports:
			- "8080:80"
		environment:
			- WORDPRESS_DB_HOST=mysql 
			- WORDPRESS_DB_USER=root
			- WORDPRESS_DB_PASSWORD=123456
			- WORDPRESS_DB_NAME=wordpress
		volumes:
			- wordpress:var/www/html
		restart: always
		networks:
			- blog
		depends_on:
			- mysql

volumes:
	mysql-data:
	wordpress:
	
networks:
	blog:
```

注意：卷映射时要在顶部元素声明

## dockerfile文件

![1744722324414](1744722324414.png)

```dockerfile
FROM alpine
WORKDIR /Initial
COPY ./target/project-user .
COPY ./config/config-docker.yaml .
RUN  mkdir config && mv config-docker.yaml config/config.yaml
EXPOSE 8080 8881
ENTRYPOINT ["./project-user"]
```

```bat
docker build -f dockerfile -t project-user:latest

```

通过下面文件将go项目编译成exe文件

```bat
chcp 65001
@echo off
:loop
@echo off&amp;color 0A
cls
echo,
echo 请选择要编译的系统环境：
echo,
echo 1. Windows_amd64
echo 2. linux_amd64

set/p action=请选择:
if %action% == 1 goto build_Windows_amd64
if %action% == 2 goto build_linux_amd64

:build_Windows_amd64
echo 编译Windows版本64位
SET CGO_ENABLED=0
SET GOOS=windows
SET GOARCH=amd64
go build -o project-user/target/project-user.exe project-user/main.go
go build -o project-api/target/project-api.exe project-api/main.go
:build_linux_amd64
echo 编译Linux版本64位
SET CGO_ENABLED=0
SET GOOS=linux
SET GOARCH=amd64
go build -o project-user/target/project-user project-user/main.go
go build -o project-api/target/project-api project-api/main.go
```



```bat
docker image inspect nginx
```


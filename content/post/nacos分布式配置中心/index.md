---
title: "Nacos分布式配置中心"
description: 
date: 2025-03-01T16:09:16+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
---

# nacos分布式配置中心

通过nacos进行远程配置，可以实现配置更改之后第一时间进行更新操作，本地需要对nacos进行连接

通过viper进行映射

```go
nacos:
  namespace: de94f0c7-5ad9-4ecc-bcf1-2f9d2732e766
  group: dev
  ipAddr: 127.0.0.1
  port: 8848
  scheme: http
  contextPath: "/nacos"
```

```
type NacosConfig struct {
	Namespace   string
	Group       string
	IpAddr      string
	Port        int
	ContextPath string
	Scheme      string
}
```



进行nacos 连接

```go

type NacosClient struct {
	confClient config_client.IConfigClient
	group      string
}

func InitNacosClient() *NacosClient {
	bootConf := InitBootstrap()
	clientConfig := constant.ClientConfig{
		NamespaceId:         bootConf.NacosConfig.Namespace, //we can create multiple clients with different namespaceId to support multiple namespace.When namespace is public, fill in the blank string here.
		TimeoutMs:           5000,
		NotLoadCacheAtStart: true,
		LogDir:              "/tmp/nacos/log",
		CacheDir:            "/tmp/nacos/cache",
		LogLevel:            "debug",
	}
	serverConfigs := []constant.ServerConfig{
		{
			IpAddr:      bootConf.NacosConfig.IpAddr,
			ContextPath: bootConf.NacosConfig.ContextPath,
			Port:        uint64(bootConf.NacosConfig.Port),
			Scheme:      bootConf.NacosConfig.Scheme,
		},
	}
	configClient, err := clients.NewConfigClient(
		vo.NacosClientParam{
			ClientConfig:  &clientConfig,
			ServerConfigs: serverConfigs,
		},
	)
	if err != nil {
		log.Fatalln(err)
	}
	nc := &NacosClient{
		confClient: configClient,
		group:      bootConf.NacosConfig.Group,
	}
	return nc
}

```



nacos可以通过监听，来实现当配置文件更改时，可以重新加载配置文件，但要注意此时数据库已经进行连接初始化，要重新进行连接数据库

```go
err2 = nacosClient.confClient.ListenConfig(vo.ConfigParam{
		DataId: "config.yaml",
		Group:  nacosClient.group,
		OnChange: func(namespace, group, dataId, data string) {
			//
			log.Printf("load nacos config changed %s \n", data)
			err := conf.viper.ReadConfig(bytes.NewBuffer([]byte(data)))
			if err != nil {
				log.Printf("load nacos config changed err : %s \n", err.Error())
			}
			//所有的配置应该重新读取
			conf.ReLoadAllConfig()
		},
	})
```


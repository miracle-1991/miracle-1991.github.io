---
title: 服务注册与发现
date: 2023-11-05 14:59:00 +0800
categories: [gateway,service_mesh]
tags: [apisix, gateway,consul]
author: miracle
mermaid: true
---
在配置[反向路由](https://miracle-1991.github.io/posts/apsixReverseProxy/)时，目标节点被配置为具体的ip，
这在实际的工作中是几乎不可用的，主要存在以下问题:
- **无法动态收缩**：生产环境中，一般会部署多个实例，如果使用静态ip的话，无法动态的新增或者减少节点，以应对业务流量的变化
- **无法动态通知服务消费者**: 实例可能随时被关闭、重启、替换，因此ip会发生变化，或者ip不变化但是服务不可用，如果这些变化不能及时被消费者知道，就会导致消费者得到错误的响应

因此需要引入服务注册与发现中心，用来支持服务动态地注册或者注销、持续检查服务状态并提供发现服务的能力。也就是要实现以下功能:
- 微服务启动时，将自身的实例信息(ip、端口、服务名、标签等)注册到注册中心，注册中心存储这些数据。
- 消费者从注册中心获取到服务提供者的实例信息，通过ip+端口方式访问该服务。
- 微服务通过心跳来上报注册中心，注册中心以某个时间段有没有接收到上报信息，来决定是否下线某服务实例。
- 微服务发生变动时比如增加实例或ip变动，重新注册信息到注册中心。这样的话服务消费者就无需改动，直接从注册中心获取最新信息即可。

![服务注册与发现](/assets/img/apisix/serviceRegisterAndFound.png)

常用的服务注册中心对比：

![对比服务发现中心](/assets/img/apisix/compareServiceRegister.png)

由于consul提供了多数据中心、服务发现与健康检查、键值存储等功能，从而成为了更加流行的一种解决方案。
因此选择将consul作为服务注册与发现中心接入到apisix中，从而使得apisix可以动态的获取服务的ip，而不再手动指定：

![apisix结合服务注册中心](/assets/img/apisix/apisixCombineServiceRegisterAndFound.drawio.png)


## 准备服务后端
我在[register](https://github.com/miracle-1991/apiGateWay/blob/master/server/echo/register/register.go)中实现了consul的注册功能：
```
func (c *consul) RegisterService(serviceName string, ip string, port int, tags []string) error {
	//health check
	srv := &api.AgentServiceRegistration{
		ID:      fmt.Sprintf("%s-%s-%d", serviceName, ip, port),  //服务唯一ID
		Name:    serviceName,                                     //服务名
		Tags:    tags,                                            //自定义标签
		Address: ip,                                              //主机地址
		Port:    port,                                            //主机端口
		Check: &api.AgentServiceCheck{
			HTTP:                           fmt.Sprintf("http://%s:%d/health", ip, port), //HTTP健康检查地址
			Timeout:                        config.HEALTH_CHECK_TIMEOUT,                  //超时时间，默认10s
			Interval:                       config.HEALTH_CHECK_INTERVAL,                 //健康检查的频率，默认1s/次
			DeregisterCriticalServiceAfter: config.DEREGISTER_CRITICAL_SERVICE_AFTER,     //指定时间后自动注销不健康的服务节点
		},
	}
	return c.client.Agent().ServiceRegister(srv)
}
```
在服务启动时自动将本服务(echo)注册到consul上，并在停止服务之前将服务从consul上注销掉：
```
func main() {
	route := router.Init()
	endPoint := fmt.Sprintf(":%d", config.PORT)

	//register
	c, err := register.NewConsul(config.CONSUL_ADDR)  //通过环境变量获知consul的地址
	if err != nil {
		panic("failed to create consul client: " + err.Error())
	}
	ipOBJ, _ := register.GetOutboundIP()              //获取本机ip
	ip := ipOBJ.String()
	serviceID := fmt.Sprintf("%s-%s-%d", config.SERVICENAME, ip, config.PORT) //构建唯一标识
	tags := []string{"version:" + strconv.Itoa(config.VER)}                   //组装标签
	err = c.RegisterService(config.SERVICENAME, ip, config.PORT, tags)        //注册
	if err != nil {
		panic("failed to register to consul: " + err.Error())
	}
	fmt.Printf("success register to consul, serviceID: %s\n", serviceID)

	go func() {
		server := &http.Server{
			Addr:    endPoint,
			Handler: route,
		}
		err = server.ListenAndServe()
		if err != nil {
			panic("failed to serve http: " + err.Error())
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
	<-quit                                                                    //监听终止信号(kill/Ctrl+C)
	err = c.Deregister(serviceID)                                             //注销
	if err == nil {
		fmt.Printf("success deregister to consul, serviceID: %s\n", serviceID)
	}
}
```
## 准备consul
我一直在统一使用docker-compose管理所有的服务，于是在[docker-compose.yml](https://github.com/miracle-1991/apiGateWay/blob/master/server/docker-compose.yml)中
将consul与apisix、后端服务一起启动:
```
services:
  consul:
    image: hashicorp/consul:latest
    ports:
      - "8300:8300"
      - "8400:8400"
      - "8500:8500"
    restart: always
    volumes:
      - consul-data:/consul/data  //将consul-data卷挂在到/consul/data目录下，这是consule默认数据目录，用来持久化数据
    networks:
      apisix:
```
## 配置Apisix
apisix需要在[配置](https://github.com/miracle-1991/apiGateWay/blob/master/server/apisix_conf/config.yaml)文件中指定consul的地址:
```
discovery:
  consul:
    servers:
      - "http://consul:8500"
```
在配置完后，apisix内部会启动一个定时器，定时地与consul进行交互，获取服务实例地址，并保存在内存中:

![apisix定时器](/assets/img/apisix/apisixConsulTimer.png)

## 配置路由
首先在[docker-compose.yml](https://github.com/miracle-1991/apiGateWay/blob/master/server/docker-compose.yml)的同级目录下
执行
```docker-compose up -d```
启动consul、etcd、apisix、apisix-dashboard、后端服务,
首先能在[consul控制台](http://127.0.0.1:8500/ui/dc1/services)中看到服务在启动后注册到了consul:

![consul控制台](/assets/img/apisix/consulDashboard.png)

然后在[apisix dashboard](http://127.0.0.1:9000/dashboard)上配置一条路由:

![配置apisix dashborad](/assets/img/apisix/configConsulRouteInApisixDashboard.png)

具体的配置如下:
```
{
  "uri": "/testecho/*",
  "name": "echoRoute",
  "methods": [
    "GET",
    "POST",
    "PUT",
    "DELETE",
    "PATCH",
    "HEAD",
    "OPTIONS",
    "CONNECT",
    "TRACE",
    "PURGE"
  ],
  "plugins": {
    "proxy-rewrite": {
      "regex_uri": [
        "^/testecho/(.*)",
        "/$1"
      ]
    }
  },
  "upstream": {
    "timeout": {
      "connect": 6,
      "send": 6,
      "read": 6
    },
    "type": "roundrobin",
    "scheme": "http",
    "discovery_type": "consul",  //服务发现方式consul
    "pass_host": "pass",
    "service_name": "echo",      //要发现的服务为echo
    "keepalive_pool": {
      "idle_timeout": 60,
      "requests": 1000,
      "size": 320
    }
  },
  "status": 1
}
```
配置完该路由后，当收到消费者发来的请求时，apisix会执行以下操作将请求反向代理到后端节点：

![apisix解析路由请求并进行负载均衡](/assets/img/apisix/apisixDealWithRequestToConsulServer.png)

## 请求测试
```
curl --location 'http://127.0.0.1:9080/testecho/hello'
```
获得：
```
{
    "message": "hello world",
    "version": 1
}
```
或者
```
{
    "message": "hello world",
    "version": 2
}
```


---
title: 反向代理
date: 2023-10-24 08:00:00 +0800
categories: [gateway]
tags: [apisix, gateway]
author: miracle
mermaid: true
---

反向代理是一个网关的基本功能。反向代理是指API网关接受互联网上的连接请求，然后将请求转发给上游的内网服务器，并将结果返回给客户端。
此处演示如果使用apisix的反向代理功能。

首先，我使用golang编写了一个简单的[echo服务](https://github.com/miracle-1991/apiGateWay/tree/master/server/echo)，
该echo服务在启动时从环境变量中读VERSION，然后在返回response时将版本信息写到body中：
```
➜ curl -X POST http://0.0.0.0:9081/echo
{"message":"","version":1}
➜ curl -X POST http://0.0.0.0:9082/echo
{"message":"","version":2}                                                                                                                                                                                                                                                 ➜  code curl -X POST http://0.0.0.0:9081/echo
```
然后我使用[docker-compose](https://github.com/miracle-1991/apiGateWay/blob/master/server/docker-compose.yml)启动了apisix、apisix dashboard、server等多个容器，期望通过配置搭建成如下网络：

![testReverseProxy](/assets/img/apisix/testReverseProxy.png)


首先启动这个docker-compose:
```
docker-compose up -d
```

然后就可以在apisix dashborad中编辑反向代理了。
首先说明一下Upstreram的概念:
* upstream是对上游虚拟主机的抽象，即应用层服务或者节点的抽象
* 在上游中可以按照配置规则对服务节点进行负载均衡，其地址信息可以配置到路由上
* 多个路由可以引用同一个上游

首先创建upstream:
![testReverseProxyAddUpstream](/assets/img/apisix/testReverseProxyAddUpstream.png)
* 此处选择的负载均衡方式是带权轮询
* 创建时需要上游的ip, ip地址可以到容器中的/etc/hosts中查看

然后就可以在路由中绑定该服务。首先新建一个路由，在该路由中，我希望将所有

```
http://{domain}/testecho/*
```

形式的请求转发到服务时去掉testcho，变成：

```
http://{domain}/*
```

所以使用了路径改写

![testReverseProxyAddRoute](/assets/img/apisix/testReverseProxyAddRoute.png)

最后选择已经配置好的路由即可：
![testReverseProxyChooseUpstream](/assets/img/apisix/testReverseProxyChooseUpstream.png)

最后进行测试：
可以看到请求能正确的到达后端两个节点：

![testReverseProxyGetVersion1](/assets/img/apisix/testReverseProxyGetVersion1.png)

![testReverseProxyGetVersion2](/assets/img/apisix/testReverseProxyGetVersion2.png)


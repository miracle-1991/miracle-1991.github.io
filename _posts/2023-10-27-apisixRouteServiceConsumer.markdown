---
title: 路由(Route)、服务(Service)、消费者(Consumer)
date: 2023-10-27 08:00:00 +0800
categories: [gateway]
tags: [apisix, gateway]
author: miracle
mermaid: true
---

在apisix中，路由(Route)、服务(Service)、消费者(Consumer)是三种主要的配置实体，有各自的用途：
1. **路由(Route)**: 路由是最基础也是最核心的资源对象，客户端请求与API服务之间的桥梁，主要的作用是
   2. 根据路由规则来匹配客户端请求
   3. 根据匹配结果加载并执行相应的插件
   4. 将请求转发到指定的上游服务
2. **服务(Service)**: 服务是某类API的抽象，也可以理解为一组路由的抽象，通常与上游服务一一对应，但与路由之间是1:N的关系
3. **消费者(Consumer)**: 消费者主要进行API的权限控制，主要用于身份认证和鉴权

这三者都能配置插件，但是运行起来的优先级是**消费者 > 路由 > 服务**

例子：
首先使用我的[docker-compose](https://github.com/miracle-1991/apiGateWay/blob/master/server/docker-compose.yml)工具将以下几个容器启动：
* apisix: 数据面
* apisix-dashborad: 控制面
* etcd: 配置中心，控制面将json配置下发到etcd，数据面以低于毫秒的速度获取最新的配置
* web1,web2: 后端服务，在8088端口上提供了两个api，分别是/hello和/echo，这两个api都是简单的返沪字符串

启动后通过http://127.0.0.1:9000/访问dashboard，进入后首先配置upstream, 在upstream中可以指定上游能提供的ip：

![upstream](/assets/img/apisix/apisixRSC_Upstream.png)
```
{
  "nodes": [
    {
      "host": "172.19.0.2",
      "port": 8088,
      "weight": 1
    },
    {
      "host": "172.19.0.7",
      "port": 8088,
      "weight": 1
    }
  ],
  "timeout": {
    "connect": 6,
    "send": 6,
    "read": 6
  },
  "type": "roundrobin",
  "scheme": "http",
  "pass_host": "pass",
  "name": "echo",
  "keepalive_pool": {
    "idle_timeout": 60,
    "requests": 1000,
    "size": 320
  }
}
```
然后配置服务，认为一组upstream就是一个服务, 在配置服务的过程中顺便配置了一下key-auth：

![server](/assets/img/apisix/apisixRSC_Server.png)
```
{
  "name": "echoService",
  "upstream_id": "484588100589191874",
  "plugins": {
    "key-auth": {
      "_meta": {
        "disable": false
      }
    }
  }
}
```
然后配置路由，将/testecho/*的流量导到echoService服务中:
![Router](/assets/img/apisix/apisixRSC_Router.png)
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
  "service_id": "484591237089723073",
  "status": 1
}
```
此时，使用postman发送请求，由于仅仅给服务配置了key-auth插件，并没有为消费者配置可以访问路由的apikey，导致无法访问路由:
![notAuth](/assets/img/apisix/apisixRSC_PostmanNotAuth.png)
需要在消费者中对其进行鉴权，配置后，在遇到头部中带有apikey:auth-apisix的请求时，会认为这个请求是某一类消费者(此处是postman)发送来的请求：
![consumer](/assets/img/apisix/apisixRSC_Consumer.png)
```
{
  "username": "postman",
  "plugins": {
    "key-auth": {
      "_meta": {
        "disable": false
      },
      "key": "auth-apisix"
    }
  }
}
```
然后再通过postman发送请求就能成功了:
![postmanSuccess](/assets/img/apisix/apisixRSC_postmanSuccess.png)

---
title: Logging Of Envoy
date: 2023-10-19 09:00:00 +0800
categories: [service_mesh]
tags: [envoy, tracing]
author: miracle
mermaid: true
---

# Envoy接入EFK
***
__本[程序](https://github.com/miracle-1991/envoy-study/tree/main/logging)在安装了docker-compose的机器上执行__
拓扑图:
![logging-test拓扑图](/assets/img/envoy/logging/logging.png)

> 容器说明
> > front: 接收http请求，向service发起grpc请求
> > service：接收grpc请求
> > filebeat: 日志采集、聚合，发送到es进行存储
> > elasticsearch：数据保存分析检索
> > kibana: 对es存储的数据进行图形化的界面展示

## 启动项目
首次运行需要初始化一些环境变量：
```
source init.sh
```
编译运行：
```
docker-compose build
docker-compose up
```
发起http请求，分别使front向三个service发起grpc请求
```
curl -v http://localhost:8000/logging?service=1
curl -v http://localhost:8000/logging?service=2
curl -v http://localhost:8000/logging?service=3
```
## 查看ES索引状态
```
http://localhost:9200/_cat/indices?v&pretty
```
![ES索引状态](/assets/img/envoy/logging/es-index-status.png)

## 查看日志
```
http://localhost:5601/app/kibana
```
![logging-test日志查询](/assets/img/envoy/logging/kibana-log-search.png)

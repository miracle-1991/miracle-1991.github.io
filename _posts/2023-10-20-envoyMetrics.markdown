---
title: Metrics Of Envoy
date: 2023-10-19 08:00:00 +0800
categories: [service_mesh]
tags: [envoy, metric]
author: miracle
mermaid: true
---

# Envoy接入prometheus/grafana监控
***
__本[程序](https://github.com/miracle-1991/envoy-study/tree/main/metrics)在安装了docker-compose的机器上执行__

拓扑图:
![viewcontest-test拓扑图](/assets/img/envoy/metrics/metrics.png)

> 容器说明
> > front: 接收http请求，向service发起grpc请求
> > service：接收grpc请求
> > prometheus: 采集各个exporter的数据采集器
> > alertmanager：负责告警推送
> > grafana： 可视化图形面板
> > blackbox_exporter: TCP/HTTP/SSH等协议的存活探测

## 部署指南
### 1、修改alertmanager配置
```
vim alertmanager/alertmanager.yml
```
> + 设置配置文件中smtp发件邮箱line3-6
> + 设置接收报警邮件的邮箱line23
> + 设置邮件主题line26
> + 其余高阶配置请自行查找资料

### 2、修改被监控实例
```
vim prometheus/prometheus.yml
```
> + 依照scrape_configs下的定义格式，修改或添加export实例
> + 其余配置可按需修改，默认不用改
## 启动项目
```
docker-compose build
docker-compose up
```
发起http请求，分别使front向三个service发起grpc请求
```
curl -v http://localhost:8000/metrics?service=1
curl -v http://localhost:8000/metrics?service=2
curl -v http://localhost:8000/metrics?service=3
```
如果遇到git.garena.com代码下载不下来的问题可以执行一下init脚本:
```
source init.sh
```
## 访问服务
```
http://宿主机IP:3000    # Grafana
http://宿主机IP:9090    # Prometheus
http://宿主机IP:9093    # Alertmanager
http://宿主机IP:9115    # Blackbox_exporter
```
### Prometheus
查看采集到的服务状态(up or down)
![查看采集到的服务状态](/assets/img/envoy/metrics/Prometheus-stats.png)

### Grafana
默认登陆密码为admin/admin
配置数据源, 地址可以是http://宿主机IP:9090或http://prometheus:9090:
![配置Prometheus Data Source](/assets/img/envoy/metrics/prometheus-set-datasource.png)
查看指标
![查看upstream请求速率的变化](/assets/img/envoy/metrics/grafana-dashboard-test.png)

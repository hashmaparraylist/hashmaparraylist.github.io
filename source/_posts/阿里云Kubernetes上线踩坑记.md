---
title: 阿里云Kubernetes上线踩坑记
date: 2020-04-01 09:47:38
tags:
 - Aliyun
 - Kubernetes
categories:
  - 后端
  - Cloud
---

```
Update:
2020-04-08 增加istio-ingressgateway高可用的设置
```

最近公司因为项目需要，在阿里云上部署了一个Kubernetes集群。虽然阿里云的文档说的还算细致，但是还是有些没有明确说明的细节。

# 1. 购买篇

申请项目预算的时候，只考虑到Worker节点，1个SLB节点以及域名和证书的预算。但是实际购买的时候发现还有许多额外的开销。

## 1.1 SNAT

这个和EIP一并购买，可以方便通过公网使用kubectl访问集群。关于SNAT网关至今不是很明白需要购买这个服务的意义何在，只是为了一个EIP来访问集群吗？

## 1.2 Ingress

这个选上了后，阿里云会给你买个SLB而且还是带公网访问的，如果你后期考虑使用Istio的话，建议你集群创建后，直接停止这个SLB，以免产生额外的费用。

## 1.3 日志服务

通过阿里云的日志服务来收集应用的的日志，挺好用的。但是另外收费，如果有能力的自建日志服务的可不购买。

# 2. Istio

阿里云的Kubernetes集群完美集成了Istio，根据向导就能很简单的部署成功。

## 2.1 额外的SLB

Istio的Gateway 需要绑定一个新的SLB，和Ingress的SLB不能是同一个，又是一笔额外的开销

## 2.2 集群外访问

这个在阿里云的Istio FAQ中有提到，按照指导很容易解决

## 2.2 SLB的443监听

为了方便443端口的证书绑定，我们直接删除了SLB上原有的443监听(TCP协议), 重新建了一个443监听(HTTPS协议)，指向和80端口同样的虚拟服务器组。但是设置健康检查时一直出错，经过排查发现SLB健康检查发送的请求协议是HTTP 1.0的，Istio的envoy直接反悔了`426(Upgrade Required)`这个状态码，所以我们无奈只能把健康检查的检查返回状态改为http_4xx，这样就能通过SLB的健康检查了。

## 2.3 istio-ingressgateway的高可用

`istio-ingressgateway`要达成高可用，只需要增加通过伸缩POD就可以实现，于`istio-ingressgateway`对应的SLB中的虚拟服务器组也会自动增加，完全不需要进行额外的手动设定。

由于`istio-ingressgateway`中挂载了HPA`HorizontalPodAutoscaler`(简称HPA)，通常三节点的集群中最小POD数只有1台，在3节点的集群中，要实现高可用，需要手动修改HPA，增加最小POD数。

---

基本上现在遇到了这些坑，再有在总结吧。

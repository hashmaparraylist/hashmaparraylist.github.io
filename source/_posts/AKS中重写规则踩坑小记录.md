---
title: AKS中重写规则踩坑小记录
date: 2022-09-23 13:51:49
tags:
  - Aliyun
  - Kubernetes
categories:
  - 后端
  - Cloud
---

# 前言

最近在做标准产品在不同云平台中的部署验证，有幸体验了一下微软的Azure。负责采购的运维部门这次采用了`Application Gateway`来搭配`AKS`(`Azure Kubernetes Service`)对外暴露服务，正好借着这个机会来体验一下`Application Gateway`。

# 应用场景

1. 域名`api.demo.com`指向`Application Gateway`的IP地址
2. 在`AKS`内部2个Service, `gateway-service`和`backend-service`分别需要通过`Application Gateway`对外暴露。
3. `/gateway/`指向`gateway-service`, 然后`/backend/`指向`backend-service`。而且两个Service都没有context-path，所以需要做一个Rewrite重写URI到Service的根目录上。

# 定义重写集

打开`AKS`对应的应用程序网关`设置` > `重写`。选择`添加重写集`。在`1. 名称和关联`这个Tab上只需要填写名称这项即可(名称后面在做ingress时需要使用), `关联的传递规则`不需要选择。`2. 重写规则配置`里添加一个重写规则，然后填上重写规则的名称，并添加条件(默认新建重写规则时，只会生成操作，不会生成条件)

## `条件`做如下设置

- **要检查的变量类型** : `服务器变量`
- **服务器变量**: `request_uri`
- **区分大小写**: `否` 
- **运算符**: `等号(=)`
- **要匹配的模式**: `/(gateway|backend)/?(.*)`

## `操作`做如下设置

- **重写类型**: `URL`
- **操作类型**: `设置`
- **组件**: `URL路径和URL查询字符串`
- **URL路径值**: `/{var_request_uri_2}`
- **重新计算路径映射**: `不选中`
- **URL查询字符串值**: `留空不设值`

## 特殊说明

`操作`里的`URL路径值`不能使用正则表达式GROUP替换组，例如`$1`和`$2`之类的。Azure自己定义了一套对应的替换组命名规则。具体可以参考这个网页[使用应用程序网关重写 HTTP 标头和 URL](https://docs.azure.cn/zh-cn/application-gateway/rewrite-http-headers-url)。

另外一个需要注意一点，如果在`条件`里选择了`服务器变量`的`request_uri`的时候，注意这个`request_uri`是完整的原始请求URI(携带了查询参数)。例如: 在请求`http://api.demo.com/gateway/search?foo=bar&hello=world`中，`request_uri`的值将为`/gateway/search?foo=bar&hello=world`。由于`request_uri`里包含了查询参数，所以在`操作`的`组件`中建议勾选`URL路径和URL查询字符串`。如果只选择`URL路径`的情况下可能出现无法预期的错误。以我们上述的配置来说明。

对象URL: `http://api.demo.com/gateway/search?foo=bar&hello=world`

**组件** | `URL路径和URL查询字符串` | `URL路径`
--- | --- | ---
**结果** | `/search?foo=bar&hello=world` | `/search?foo=bar&hello=world?foo=bar&hello=world`

# `ACK`的Ingress设置

当选择了`Application Gateway`作为对外暴露Service的方式时，Kubernetes集群里(`kube-system`命名空间里)多一个`Application Gateway Ingress Controller`(Azure工单时通常会简称为`agic`)的Deployment,所以对外暴露服务时可以像传统`nginx ingress controller`一样添加一个`Ingress`对象即可(甚至配置也和ngic大致相同，只是多了2个annotations)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    # 这里指定重写规则集(不是重写规则的名字)
    appgw.ingress.kubernetes.io/rewrite-rule-set: rule-backend
    # 指定说明你这里ingress的类型是agic
    kubernetes.io/ingress.class: azure/application-gateway
  name: backend-ingress
  namespace: default
spec:
  rules:
  - host: api.demo.com
    http:
      paths:
      - backend:
          service:
            name: gateway-service
            port:
              number: 8080
        path: /gateway/
        pathType: Prefix
      - backend:
          service:
            name: backend-service
            port:
              number: 8080
        path: /backend/
        pathType: Prefix
```

# 总结

由于微软云这块文档有部分缺失，导致在配置这块花了一点时间去排查，甚至开了工单。总结下来Ingress的配置主要是根据请求路径路由到对应的Service，重写规则集才是实际负责根据正则来进行匹配重写。
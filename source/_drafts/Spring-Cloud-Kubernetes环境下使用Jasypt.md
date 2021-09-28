---
title: Spring Cloud Kubernetes环境下使用Jasypt
tags:
  - Java
  - Spring
categories:
  - 后端
---

# 前言

最近半年着手开始做了基于微服务的中台项目，整个项目的技术栈采用的是`Java` + `Spring Cloud` + `Kubernetes` + `Istio`。

业务开放上还是相当顺利的。但是在安全审核上，运维组提出了一个简易。现在项目一些敏感配置，例如MySQL用户的密码，Redis的密码等现在都是明文保存在Kubernetes的ConfigMap中的(是的，我们并没有Nacos作为微服务的配置中心)。这样可能存在安全隐患。

# 首次尝试

既然有问题，那就解决问题。要给配置文件中的属性项目加密很简单，稍微Google一下，就有现成的方案了。

现在比较常用的解决方案就是集成`Jasypt`,然后通过`jasypt-spring-boot-starter`来融合进Spring。

### POM包加入`jasypt-spring-boot-starter`

```xml
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>3.0.4</version>
</dependency>
```

### Dockerfile中增加java参数

```

```

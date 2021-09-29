---
title: Spring Cloud Kubernetes环境下使用Jasypt
tags:
  - Java
  - Spring
categories:
  - 后端
date: 2021-09-29 11:28:11
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
...
ENTRYPOINT ["sh","-c","java $JAVA_OPTS -jar app.jar --jasypt.encryptor.password=helloworld $PARAMS"]
```

### 在ConfigMap中添加加密属性

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo
data:
  application.yaml: |-
    test2: ENC(94Y7Ds3+RKraxQQlura9sDx+9yF0zDLMGMwi2TjyCFZOkkHfreRFSb6fxbyvCKs7)
```

### 利用`actuator`接口测试

在`management.endpoints.web.exposure.include`属性中增加`env`，这样我们就可以通过调用`/actuator/env`来查看一下`env`接口返回的整个Spring 容器中所有的PropertySource。

```json
{
    ...
    "propertySources": [
        {
            "name": "bootstrapProperties-configmap.demo.default",
            "properties": {
                "test2": {
                    "value": "Hello,world"
                }
            }
        }
        ...
    ]
}
```

OK, 这下配置项已经加密了。问题解决了。

但是...

## 新的问题

自从项目集成了`Jayspt`以后，出现了一个奇怪的问题。每次项目试图通过修改ConfigMap的配置文件，然后试图通过`spring-cloud-starter-kubernetes-fabric8-config`来做自动Reload，都失败了。然而查阅应用日志，并没有出现任何异常。无奈只能打开`spring-cloud`和`jasypt-spring-boot`的`DEBUG`日志。

进过几天对日志和两边源代码的分析。终于找到了原因

### 原因

在Spring Boot启动时`jasypt-spring-boot`会将下面6种配置(并不仅限与这6种配置文件)

- `Classpath`下的`application.yaml`
- `Classpath`下的`bootstrap.yaml`
- 集群里名称为`${spring.cloud.kubernetes.config.name}`的ConfigMap
- 集群里名称为`${spring.cloud.kubernetes.config.name}-kubernetes`的ConfigMap
- Java启动参数
- 环境变量

转换成`jasypt-spring-boot`自己的PropertySource实现类`EncryptableMapPropertySourceWrapper`。

但是如果使用Kubernetes的ConfigMap来作微服务配置中心的时候，Spring Cloud会在`ConfigurationChangeDetector`中查找配置类`org.springframework.cloud.bootstrap.config.BootstrapPropertySource`, 并依据`BootstrapPropertySource`的类型来判断容器内的配置与集群中ConfigMap里的配置是否有差异,来触发配置reload。

由于`jasypt-spring-boot`已经将所有的配置文件转型成了`EncryptableMapPropertySourceWrapper`, 所以`ConfigurationChangeDetector`无法找到`BootstrapPropertySource`所以会一直任务ConfigMap的里的配置没有变化，导致整个Reload失效(无论是使用polling还是event方式)

### 解决问题

为了保证ConfigMap变化后自动Reload的功能，所以`jasypt-spring-boot`不能把`BootstrapPropertySource`转换成`EncryptableMapPropertySourceWrapper`

所以我们需要设置`jasypt.encryptor.skip-property-sources`配置项, Classpath中的application.yaml需要增加配置

```yaml
jasypt:
  encryptor:
    skip-property-sources: org.springframework.cloud.bootstrap.config.BootstrapPropertySource
```

`skip-property-sources`配置项配置后，加密项目就不能配置在ConfigMap里了，毕竟已经被我们忽略了。那么我们只能另外找一个PropertySource来存放加密项目了。

`Classpath`中的两个Yaml由于编译时会被Maven打包进Jar文件，会牵涉多个CI/CD多个流程显然不合适，启动参数配置项的也要影响到Docker镜像制作这个流程。所以判断下来最适合的PropertySource就是环境变量了。

### 环境变量增加加密项

在Kubernetes的部署Yaml中，添加加密数据项`application.test.str`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: demo
  name: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
        - env:
            - name: TZ
              value: Asia/Shanghai
            - name: application.test.str
              value: >-
                ENC(94Y7Ds3+RKraxQQlura9sDx+9yF0zDLMGMwi2TjyCFZOkkHfreRFSb6fxbyvCKs7)
    ....
```

如果需要更加严密的加密方针的话，我们可以把环境变量的内容放进Kubernetes的Secrets中。

### 在ConfigMap中引用`application.test.str`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo
data:
  application.yaml: |-
    test2: ENC(94Y7Ds3+RKraxQQlura9sDx+9yF0zDLMGMwi2TjyCFZOkkHfreRFSb6fxbyvCKs7)
    test3: ${application.test.str}
```

### 通过`actuator`接口来测试

通过`actuator\env`接口来测试一下

```json
{
    ...
    "propertySources": [
        {
            "name": "bootstrapProperties-configmap.demo.default",
            "properties": {
                "test2": {
                    "value": "ENC(94Y7Ds3+RKraxQQlura9sDx+9yF0zDLMGMwi2TjyCFZOkkHfreRFSb6fxbyvCKs7)"
                },
                "test3": {
                    "value": "Hello,world"
                }
            }
        }
        ...
    ]
}
```

这样ConfigMap中的配置项`test3`就可以通过环境变量引用并使用加密配置项了。同时修改ConfigMap依然可以触发auto reload了。这下终于算是解决了。
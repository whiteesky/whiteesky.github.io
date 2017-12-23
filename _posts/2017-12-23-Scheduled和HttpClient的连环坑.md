---
layout: post
title: Scheduled和HttpClient的连环坑
categories: 技术总结
author: whiteesky
date: 2017-12-23 11:23:04
tags:
  - Java
  - Spring
  - Http
header-img: img/article/4.jpg
copyright: true
---
**曾经踩过一个大坑：**
由于业务特殊性，会定时跑很多定时任务，对业务数据进行补偿操作等。
在Spring使用过程中，我们可以使用@Scheduled注解可以方便的实现定时任务。
有一天早上突然发现，从前一天晚上某一时刻开始，所有的定时任务全部都卡死不再运行了。

@Scheduled默认单线程
---------------

经排查后发现，我们使用@Scheduled注解默认的配置的话，所有的任务都是单线程去跑的。写了一个测试的task让它sleep住，就很容易发现，其他所有的task在时间到的时候都没有触发。

如果需要开启多线程处理，则需要进行如下的配置，设置一下线程数：

```java
@Configuration
public class ScheduleConfig implements SchedulingConfigurer {
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        taskRegistrar.setScheduler(Executors.newScheduledThreadPool(5));
    }
}
```

这样就解决了如果一个task卡住，会引起所有task全部卡住的问题。

但是为什么会有task卡住呢？

HttpClient默认参数配置
----------------

原来，有些task会定时请求外部服务的restful接口，而HttpClient的配置如下：

```java
PoolingHttpClientConnectionManager connManager = new PoolingHttpClientConnectionManager();
        connManager.setMaxTotal(maxConnection);
        httpClient = HttpClients.custom()
                .setConnectionManager(connManager)
                .build();
```

在最开始使用HttpClient的时候，根本没有想这么多，基本也都是用用默认配置。

追踪源码可以发现，在使用上述方式进行配置的时候，HttpClient的timeout时间竟然全部都是-1，也就是说如果对方服务有问题，HttpClient的请求会永不超时，一直等待。源码如下：

```java
Builder() {
        super();
        this.staleConnectionCheckEnabled = false;
        this.redirectsEnabled = true;
        this.maxRedirects = 50;
        this.relativeRedirectsAllowed = true;
        this.authenticationEnabled = true;
        this.connectionRequestTimeout = -1;
        this.connectTimeout = -1;
        this.socketTimeout = -1;
        this.contentCompressionEnabled = true;
}
```

所以我们这时候必须手动指定timeout时间，问题就解决了。例如：

```java
PoolingHttpClientConnectionManager connManager = new PoolingHttpClientConnectionManager();
        connManager.setMaxTotal(maxConnection);
        RequestConfig defaultRequestConfig = RequestConfig.custom()
                .setSocketTimeout(3000)
                .setConnectTimeout(3000)
                .setConnectionRequestTimeout(3000)
                .build();
        httpClient = HttpClients.custom()
                .setDefaultRequestConfig(defaultRequestConfig)
                .setConnectionManager(connManager)
                .build();
```

联想到另一个问题
--------

其实HttpClient的使用过程中也遇到过另外一个配置的问题，就是defaultMaxPerRoute这个参数。
最开始使用的时候也没有注意过这个参数，只是设置过连接池的最大连接数maxTotal。

defaultMaxPerRoute参数其实代表了每个路由的最大连接数。比如你的系统需要访问另外两个服务：google.com 和 bing.com。如果你的maxTotal设置了100，而defaultMaxPerRoute设置了50，那么你的每一个服务的最大请求数最大只能是50。

那么如果defaultMaxPerRoute没有设置呢，追踪源码：

```java
public PoolingHttpClientConnectionManager(
        final HttpClientConnectionOperator httpClientConnectionOperator,
        final HttpConnectionFactory<HttpRoute, ManagedHttpClientConnection> connFactory,
        final long timeToLive, final TimeUnit tunit) {
        super();
        this.configData = new ConfigData();
        //这里使用的CPool构造方法，第二个参数即为defaultMaxPerRoute，也就是默认为2。
        this.pool = new CPool(new InternalConnectionFactory(
                this.configData, connFactory), 2, 20, timeToLive, tunit);
        this.pool.setValidateAfterInactivity(2000);
        this.connectionOperator = Args.notNull(httpClientConnectionOperator, "HttpClientConnectionOperator");
        this.isShutDown = new AtomicBoolean(false);
}
```

这里发现，原来默认值竟然只有2。怪不得当时在高并发情况下总会出现超时，明明maxTotal已经设的很高。

所以如果你的服务访问很多不同的外部服务，并且并发量比较大，一定要好好配置maxTotal和defaultMaxPerRoute两个参数。

所以后来再使用任何新的东西，都有好好看下都什么配置，有疑问的一定要先查一下，不要网上copy一段代码直接就用。当时可能没问题，但是以后没准就被坑了。

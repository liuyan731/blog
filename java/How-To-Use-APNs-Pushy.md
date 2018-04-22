---
title: 使用Pushy进行APNs消息推送
date: 2017-12-05 23:11:00
categories: Web
tags: [apns,pushy,apple]

---
![APNs](https://raw.githubusercontent.com/liuyan731/tuchuang/master/blog/apns.jpeg)

*Blog的Github地址：[https://github.com/liuyan731/blog](https://github.com/liuyan731/blog)*

***

最近对项目组的老的苹果IOS推送进行了升级修改。看了看苹果的接口文档，感觉自己直接来写一个保证稳定和高效的接口还是有点难度，同时为了避免重复造轮子（懒），囧....调研了一些开源常用的库之后，选择了Turo团队开发和维护的[pushy](https://github.com/relayrides/pushy)。

## APNs和Pushy
苹果设备的消息推送是依靠苹果的APNs（Apple Push Notification service）服务的，[APNs](https://developer.apple.com/library/content/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/APNSOverview.html#//apple_ref/doc/uid/TP40008194-CH8-SW1)的官方简介如下：

>Apple Push Notification service (APNs) is the centerpiece of the remote notifications feature. It is a robust, secure, and highly efficient service for app developers to propagate information to iOS (and, indirectly, watchOS), tvOS, and macOS devices.

IOS设备（tvOS、macOS）上的所有消息推送都需要经过APNs，APNs服务确实非常厉害，每天需要推送上百亿的消息，可靠、安全、高效。就算是微信和QQ这种用户级别的即时通讯app在程序没有启动或者后台运行过程中也是需要使用APNs的（当程序启动时，使用自己建立的长连接），只不过腾讯优化了整条从他们服务器到苹果服务器的线路而已，所以觉得推送要快（参考[知乎](https://www.zhihu.com/question/20654687)）。

项目组老的苹果推送服务使用的是苹果以前的基于二进制socket的APNs，同时使用的是一个[javapns](https://github.com/fernandospr/javapns-jdk16)的开源库，这个javapns貌似效果不是很好，在网上也有人有过讨论。javapns现在也停止维护DEPRECATED掉了。作者建议转向基于苹果新APNs服务的库。

苹果新APNs基于HTTP/2，通过连接复用，更加高效，当然还有其它方面的优化和改善，可以参考[APNs的一篇介绍](http://www.jianshu.com/p/ace1b422bad4)，讲解的比较清楚。

再说一下我们使用的[Pushy](https://github.com/relayrides/pushy)，官方简介如下：

>Pushy is a Java library for sending APNs (iOS, macOS, and Safari) push notifications. It is written and maintained by the engineers at Turo......**We believe that Pushy is already the best tool for sending APNs push notifications from Java applications**, and we hope you'll help us make it even better via bug reports and pull requests. 

Pushy的文档和说明很全，讨论也很活跃，作者基本有问必答，大部分疑问都可以找到答案，使用难度也不大。

## 使用Pushy进行APNs消息推送

### 首先加入包

```
<dependency>
    <groupId>com.turo</groupId>
    <artifactId>pushy</artifactId>
    <version>0.11.1</version>
</dependency>
```

### 身份认证

苹果APNs提供了两种认证的方式：基于JWT的身份信息token认证和基于证书的身份认证。Pushy也同样支持这两种认证方式，这里我们使用证书认证方式，关于token认证方式可以查看Pushy的文档。

如何获取苹果APNs身份认证证书可以查考[官方文档](https://developer.apple.com/library/content/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/APNSOverview.html)。

### Pushy使用
```
ApnsClient apnsClient = new ApnsClientBuilder()
    .setClientCredentials(new File("/path/to/certificate.p12"), "p12-file-password")
    .build();
```
ps. 这里的setClientCredentials函数也可以支持传入一个InputStream和证书密码。

同时也可以通过setApnsServer函数来指定是开发环境还是生产环境：
```
ApnsClient apnsClient = new ApnsClientBuilder().setApnsServer(ApnsClientBuilder.DEVELOPMENT_APNS_HOST)
    .setClientCredentials(new File("/path/to/certificate.p12"), "p12-file-password")
    .build();
```

Pushy是基于Netty的，通过ApnsClientBuilder我们可以根据需要来修改ApnsClient的连接数和EventLoopGroups的线程数。

```
EventLoopGroup eventLoopGroup = new NioEventLoopGroup(4);
ApnsClient apnsClient = new ApnsClientBuilder()
        .setClientCredentials(new File("/path/to/certificate.p12"), "p12-file-password")
        .setConcurrentConnections(4).setEventLoopGroup(eventLoopGroup).build();
```

关于连接数和EventLoopGroup线程数官网有如下的说明，简单来说，不要配置EventLoopGroups的线程数超过APNs连接数。
>Because connections are bound to a single event loop (which is bound to a single thread), it never makes sense to give an ApnsClient more threads in an event loop than concurrent connections. A client with an eight-thread EventLoopGroup that is configured to maintain only one connection will use one thread from the group, but the other seven will remain idle. Opening a large number of connections on a small number of threads will likely reduce overall efficiency by increasing competition for CPU time.

关于消息的推送，注意一定要使用**异步操作**，Pushy发送消息会返回一个Netty Future对象，通过它可以拿到消息发送的情况。

```
for (final ApnsPushNotification pushNotification : collectionOfPushNotifications) {
    final Future sendNotificationFuture = apnsClient.sendNotification(pushNotification);

    sendNotificationFuture.addListener(new GenericFutureListener<Future<PushNotificationResponse>>() {
        
        @Override
        public void operationComplete(final Future<PushNotificationResponse> future) throws Exception {
            // This will get called when the sever has replied and returns immediately
            final PushNotificationResponse response = future.getNow();
        }
    });
}
```
APNs服务器可以保证同时发送1500条消息，当超过这个限制时，Pushy会缓存消息，所以我们不必担心异步操作发送的消息过多（当我们的消息非常多，达到上亿时，我们也得做一些控制，避免缓存过大，内存不足，Pushy给出了使用Semaphore的解决方法）。
> The APNs server allows for (at the time of this writing) 1,500 notifications in flight at any time. If we hit that limit, Pushy will buffer notifications automatically behind the scenes and send them to the server as in-flight notifications are resolved.

> In short, asynchronous operation allows Pushy to make the most of local resources (especially CPU time) by sending notifications as quickly as possible.

以上仅是Pushy的基本用法，在我们的生产环境中情况可能会更加复杂，我们可能需要知道什么时候所有推送都完成了，可能需要对推送成功消息进行计数
，可能需要防止内存不足，也可能需要对不同的发送结果进行不同处理....不多说，上代码。

### 最佳实践
参考Pushy的[官方最佳实践](https://github.com/relayrides/pushy/wiki/Best-practices)，我们加入了如下操作：
- 通过Semaphore来进行流控，防止缓存过大，内存不足
- 通过CountDownLatch来标记消息是否发送完成
- 使用AtomicLong完成匿名内部类operationComplete方法中的计数
- 使用Netty的Future对象进行消息推送结果的判断

具体用法参考如下代码：

```
public class IOSPush {

    private static final Logger logger = LoggerFactory.getLogger(IOSPush.class);

    private static final ApnsClient apnsClient = null;

    private static final Semaphore semaphore = new Semaphore(10000);

    public void push(final List<String> deviceTokens, String alertTitle, String alertBody) {

        long startTime = System.currentTimeMillis();

        if (apnsClient == null) {
            try {
                EventLoopGroup eventLoopGroup = new NioEventLoopGroup(4);
                apnsClient = new ApnsClientBuilder().setApnsServer(ApnsClientBuilder.DEVELOPMENT_APNS_HOST)
                        .setClientCredentials(new File("/path/to/certificate.p12"), "p12-file-password")
                        .setConcurrentConnections(4).setEventLoopGroup(eventLoopGroup).build();
            } catch (Exception e) {
                logger.error("ios get pushy apns client failed!");
                e.printStackTrace();
            }
        }

        long total = deviceTokens.size();

        final CountDownLatch latch = new CountDownLatch(deviceTokens.size());

        final AtomicLong successCnt = new AtomicLong(0);

        long startPushTime =  System.currentTimeMillis();

        for (String deviceToken : deviceTokens) {
            ApnsPayloadBuilder payloadBuilder = new ApnsPayloadBuilder();
            payloadBuilder.setAlertBody(alertBody);
            payloadBuilder.setAlertTitle(alertTitle);
            
            String payload = payloadBuilder.buildWithDefaultMaximumLength();
            final String token = TokenUtil.sanitizeTokenString(deviceToken);
            SimpleApnsPushNotification pushNotification = new SimpleApnsPushNotification(token, "com.example.myApp", payload);

            try {
                semaphore.acquire();
            } catch (InterruptedException e) {
                logger.error("ios push get semaphore failed, deviceToken:{}", deviceToken);
                e.printStackTrace();
            }
            final Future<PushNotificationResponse<SimpleApnsPushNotification>> future = apnsClient.sendNotification(pushNotification);

            future.addListener(new GenericFutureListener<Future<PushNotificationResponse>>() {
                @Override
                public void operationComplete(Future<PushNotificationResponse> pushNotificationResponseFuture) throws Exception {
                    if (future.isSuccess()) {
                        final PushNotificationResponse<SimpleApnsPushNotification> response = future.getNow();
                        if (response.isAccepted()) {
                            successCnt.incrementAndGet();
                        } else {
                            Date invalidTime = response.getTokenInvalidationTimestamp();
                            logger.error("Notification rejected by the APNs gateway: " + response.getRejectionReason());
                            if (invalidTime != null) {
                                logger.error("\t…and the token is invalid as of " + response.getTokenInvalidationTimestamp());
                            }
                        }
                    } else {
                        logger.error("send notification device token={} is failed {} ", token, future.cause().getMessage());
                    }
                    latch.countDown();
                    semaphore.release();
                }
            });
        }

        try {
            latch.await(20, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            logger.error("ios push latch await failed!");
            e.printStackTrace();
        }

        long endPushTime = System.currentTimeMillis();

        logger.info("test pushMessage success. [共推送" + total + "个][成功" + (successCnt.get()) + "个], 
            totalcost= " + (endPushTime - startTime) + ", pushCost=" + (endPushTime - startPushTime));
    }
}

```
- 关于多线程调用client

> Pushy ApnsClient是线程安全的，可以使用多线程来调用

- 关于创建多个client

创建多个client是可以加快发送速度的，但是提升并不大，作者建议：

> ApnsClient instances are designed to stick around for a long time. They're thread-safe and can be shared between many threads in a large application. We recommend creating a single client (per APNs certificate/key), then keeping that client around for the lifetime of your application.

- 关于APNs响应信息（错误信息）
> 可以查看官网的error code表格（[链接](https://developer.apple.com/library/content/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/CommunicatingwithAPNs.html#//apple_ref/doc/uid/TP40008194-CH11-SW1)），了解出错情况，及时调整。

## Pushy性能
作者在Google讨论组中说Pushy推送可以单核单线程达到10k/s-20k/s，如下图所示：

![pushy-discuss](https://raw.githubusercontent.com/liuyan731/tuchuang/master/blog/How-To-Use-APNs-Pushy.jpg)
*作者关于创建多client的建议及Pushy性能描述*

但是可能是网络或其他原因，我的测试结果没有这么好，把测试结果贴出来，仅供参考（时间ms）：

ps. 由于是测试，没有大量的设备可以用于群发推送测试，所以以往一个设备发送多条推送替代。这里短时间往一个设备发送大量的推送，APNs会报TooManyRequests错误，Too many requests were made consecutively to the same device token。所以会有少量消息无法发出。

ps. 这里的推送时间，没有加上client初始化的时间。

ps. 消息推送时间与被推消息的大小有关系，这里我在测试时没有控制消息变量（都是我瞎填的，都是很短的消息）所以数据**仅供参考**。

- ConcurrentConnections: 1, EventLoopGroup Thread: 1

||推送1个设备|推送13个设备|同一设备推100条|同一设备推1000条|
|:----:| :---- | ----: | :----: | :----: |
|平均推送成功(个)|1|13|100|998|
|平均推送耗时(ms)|222|500|654|3200|

- ConcurrentConnections: 5, EventLoopGroup Thread: 1

||推送1个设备|推送13个设备|同一设备推100条|同一设备推1000条|
|:----:| :---- | ----: | :----: | :----: |
|平均推送成功(个)|1|13|100|999|
|平均推送耗时(ms)|310|330|1600|1200|

- ConcurrentConnections: 4, EventLoopGroup Thread: 4

||推送1个设备|推送13个设备|同一设备推100条|同一设备推1000条|
|:----:| :---- | ----: | :----: | :----: |
|平均推送成功(个)|1|13|100|999|
|平均推送耗时(ms)|250|343|700|1700|

关于性能优化也可以看看官网作者的建议：[Threads, concurrent connections, and performance](https://github.com/relayrides/pushy/wiki/Best-practices)

大家有测试的数据也可以分享出来一起讨论一下。

**今天（12.11）又测了一下，推送给3个设备，每个重复推送1000条，共3000条，结果如下（时间为ms）：**

|thread/connection|No.1|No.2|No.3|No.4|No.5|No.6|No.7|No.8|No.9|No.10|Avg|
|----| :----: | :----: | :----: | :----: |:----: |:----: |:----: |:----: |:----: | :----: | :----: |
|1/1|12903|12782|10181|10393|11292|13608|-|-|-|-|11859.8|
|4/4|2861|3289|6258|5488|6649|6113|7042|5393|4591|7269|5495.3|
|20/20|1575|1456|1640|2761|2321|2154|1796|1634|2440|2114|1989.1|
|40/40|1535|2134|3312|2311|1553|2088|1734|1834|1530|1724|1975.5|

**同时测了一下，给这3个设备重复推送100000条消息，共300000条的时间，结果如下（时间为ms）：**

|thread/connection|No.1|
|:----:| :----: |
|20/20|43547|

## 思考
苹果APNs一直在更新优化，一致在拥抱新技术（HTTP/2，JWT等），是一个非常了不起的服务。

自己来直接调用APNs服务来达到生成环境要求还是有点困难。Turo给我们提供了一个很好的Java库：Pushy。Pushy还有一些其他的功能与用法（Metrics、proxy、Logging...），总体来说还是非常不错的。

同时感觉我们使用Pushy还可以调优...

***
*2017/12/07 done*

*此文章也同步至[个人Github博客](https://liuyan731.github.io/)*
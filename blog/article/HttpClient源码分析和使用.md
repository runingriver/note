---
title: HttpClient源码分析和使用
date: 2018/02/24 11:12:22
toc: false
list_number: false
categories:
- HttpClient
tags:
- HttpClient
---

前言：httpclient是我们日常开发不和绕过的一个话题，开发中遇到很多疑问，看过很多blog，这里通过阅读关键部分的源码，总结了部分个人觉得比较重要的部分。
具体使用强烈建议阅读《亿级流量网站架构核心技术》这本书的第10.3,12.2节，写得非常好。

## 一. 重要参数含义
1. `setConnectTimeout`：跟目标服务建立连接超时时间。
2. `setSocketTimeout`：socket连接超时时间。
3. `setConnectionRequestTimeout`：请求等待超时时间。
4. `keepAliveStrategy`：一个http连接存活时间，通常根据http头的`Keep-Alive`参数来决定，如果`Keep-Alive`没有定义则默认`-1`，即无限期的保活这个http连接。如果服务设置了将多久不活动的连接丢掉以释放系统资源的话，服务器丢掉这个连接时是不会通知client端的。
如果有必要，可以自定义连接在pool中的保活时长：
 ```
ConnectionKeepAliveStrategy keepAliveStrategy = new DefaultConnectionKeepAliveStrategy() {
            @Override
            public long getKeepAliveDuration(final HttpResponse response, final HttpContext context) {
                long keepAlive = super.getKeepAliveDuration(response, context);
                if (keepAlive == -1) {
                    //保活5s
                    keepAlive = 5000;
                }
                return keepAlive;
            }
        };
 ```
5. `connTimeToLive`，连接在pool中存活时长，默认`-1`，即永不失效。
6. `evictExpiredConnections`和`evictIdleConnections`，过期和空闲连接剔除策略，如果启用其一，则会独立起一个线程每隔固定时间（默认5s）去检查一遍所有连接，将过期和空闲连接剔除。
`IdleConnectionEvictor`中实现线程定时检测，具体逻辑见：`connectionManager.closeExpiredConnections()`和`connectionManager.closeIdleConnections()`

**注：HttpClient的所有各种策略，参数，逻辑，我们都可以在`HttpClientBuilder.build()`中找到解释！**

- - -

## 二. 连接池
实例化一个`HttpClient`时，如果我们没有手动指定`PoolingHttpClientConnectionManager`对象，则会创建一个默认的`PoolingHttpClientConnectionManager`对象。
**默认`PoolingHttpClientConnectionManager`，默认创建`5`个`per route`（域名链路），`10`个链接connection**。
**Tip：`per route`表示路由个数，即能够同时请求的域名个数，默认情况，我们对每隔域名的请求只会开两个端口，即两条tcp链路，去请求。这样我们手动开启100个线程去同时请求同一个接口时，其实并发只是2，有98个线程会处于阻塞状态。**
**Tip：**通常如果我们需要频繁请求某个接口时，可以将`PoolingHttpClientConnectionManager`作为全局变量保存，以支持route和connection的重用
**Tip：**当同时需要进行多次http请求时，我们可以在方法内手动`new PoolingHttpClientConnectionManager`并指定`setMaxTotal()`和`setDefaultMaxPerRoute()`，并启用多线程去请求。
### (1) PoolingHttpClientConnectionManager使用
```
public class PoolingHttpClientHelper {
    private static final Logger logger = LoggerFactory.getLogger(PoolingHttpClientHelper.class);

    PoolingHttpClientConnectionManager poolingConnManager;

    public <T> T get(String path, Class<T> clazz) {
        CloseableHttpClient httpClient = HttpClients.custom().setConnectionManager(poolingConnManager)
                .build();
        //实例化GET请求
        HttpGet httpget = new HttpGet(path);
        //JSON数据格式
        String json = null;
        CloseableHttpResponse response = null;
        try {
            response = httpClient.execute(httpget);
            InputStream in = response.getEntity().getContent();
            json = IOUtils.toString(in, Charset.defaultCharset());
            in.close();
        } catch (UnsupportedOperationException | IOException e) {
            logger.error("http get request exception.path:{}", path, e);
        } finally {
            if (response != null) {
                try {
                    response.close();
                } catch (IOException e) {
                    logger.error("http get request release exception.path:{}", path, e);
                }
            }
        }
        return JSON.parseObject(json, clazz);
    }
}
```
**注意：这里要` in.close();`和` response.close();`都释放，以备下次重用。**
**注意：这里没有做线程安全考虑，代码仅做参考！**

### (2)BasicClientConnectionManager
`BasicClientConnectionManager`是一个简单的连接管理类，内部仅维护一个`connection`，即每次只允许一个http请求，虽然它线程安全，多线程下还是串行执行，如果临近两次请求的是同一个route，则这个connection会被重用，否则先关闭然后再新建一个connection。

- - -

## 三. 重试机制
1. 请求异常重试
**HttpClient默认对http请求`IOException`类型异常重试3次**，通过`DefaultHttpRequestRetryHandler`类实现（是一个单例模式类），重试之间没有时间间隔，如果请求过程中抛以下异常：`InterruptedIOException`,`UnknownHostException`,`ConnectException`,`SSLException`，**则不进行重试**，这些不启动异常重试的类可以自定义，初始化时保存在`Set<Class<? extends IOException>> nonRetriableClasses`中。
Tip：重试对象的单例模式：`public static final DefaultHttpRequestRetryHandler INSTANCE = new DefaultHttpRequestRetryHandler();`
Tip：可以通过`disableAutomaticRetries()`方法关闭请求异常重试。
Tip：可以通过`setRetryHandler()`方法自定义重试逻辑。
2. 服务不可用重试
如果服务器返回`503`（服务器过载或维护等），则进行`Service Unavailable Retry`，默认重试一次，每次间隔`1s`，通过`ServiceUnavailableRetryStrategy`类实现，`ServiceUnavailableRetryExec`对其进行`decorator`，以便在请求完成之后调用，确定是否进行重试。
Tip：`setServiceUnavailableRetryStrategy()`方法可以自定义服务器过载重试逻辑！
Tip：只有在服务器返回`503`才会进行重试，HttpClient默认启用重试，重试一次，间隔一秒！

**注：**当实际中，我们依赖外部HTTP接口时，建议对接口做相关重试和降级机制，如请求失败，我们每隔1s，30s，1min去再次请求服务器；若仍不行，我们可以报警，并降级，直接返回预定的内容！

- - -

## 四. 调优
上述几点基本也是向着调优策略进行的，下面列出几点通用调优：
1. 连接数，通过`HttpClientBuilder#maxConnTotal`和`#maxConnPerRoute`分别设置最大连接数和单route连接数，增加吞吐能力。
2. 获取连接的超时时间，通过`RequestConfig#connectionRequestTimeout`进行设置。调小获取连接超时时间能够有效提高响应速度并且降低积压请求量，但相应的也会增加请求失败的几率。
3. 建立连接和route响应的超时时间，调小能够有效的降低**bad request**对连接的占用，留给质量更好的请求，有效提高系统提高吞吐能力及响应速度。否则有可能在峰值期被慢请求占满连接池，导致系统瘫痪。两者分别可通过`RequestConfig#connectionTimeout`和`socketTimeout`进行设置。 
4. 开启`BackoffStrategyExec`，对状况差的route进行降级处理，将连接让给其他route。

## 参考
1. 《亿级流量网站架构核心技术》第10.3,12.2节
2. http://hc.apache.org/httpcomponents-client-4.3.x/quickstart.html
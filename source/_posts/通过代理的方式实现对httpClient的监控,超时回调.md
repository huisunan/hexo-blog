---
title: 通过代理的方式实现对httpClient的监控,超时回调
date: 2021-05-21 15:53
tags: java
categories: 
---

<!--more-->

# 通过代理的方式实现对httpClient的监控,超时回调

> 实现的功能
> 
> 1.记录请求时间  
> 2.记录请求内容  
> 3.简化回调  
> 4.重时重试功能

通过静态代理的方式实现

## 代理类

```java
/**
 * 1.记录请求时间
 * 2.记录请求内容
 * 3.简化回调
 */
@Slf4j
public class RetryFutureProxy implements FutureCallback<HttpResponse>, RetryFutureCallBack {
    protected static final Logger requestLogger = LoggerFactory.getLogger("requestLogger");
    /**
     * 已重试次数
     */
    int retryUse = 0;
    /**
     * 总重试次数
     */
    int retryCount;
    /**
     * 请求
     */
    protected HttpUriRequest request;
    /**
     * 日志体
     */
    private RequestLogEntity requestLogEntity;

    /**
     * 当前重试
     */
    private RequestLogEntity nowRetryEntity;

    //回调接口
    protected RetryFutureCallBack callBack;

    //
    public RetryFutureProxy(HttpUriRequest request){
        this(request,new EmptyRetryFutureCallBack());
    }
    /**
     * 默认为不重试的请求
     *
     * @param request 请求体
     */
    public RetryFutureProxy(HttpUriRequest request, RetryFutureCallBack callBack) {
        this(-1, request,callBack);
    }

    /**
     * @param count   重试次数
     * @param request 请求
     */
    public RetryFutureProxy(int count, HttpUriRequest request, RetryFutureCallBack callBack) {
        this.callBack = callBack;
        this.retryCount = count;
        this.request = request;
        this.requestLogEntity = new RequestLogEntity();
        this.requestLogEntity.setStartTime(new Date());
        this.requestLogEntity.generateUrlInfo(request.getURI().toString());
        this.requestLogEntity.setStatus(RequestEnum.FAIL);
        this.requestLogEntity.setRetryCount(0);
        if (retryCount > 0) {
            this.requestLogEntity.setRetryList(new ArrayList<>());
        }
    }


    //记录请求结果和打印日志
    private void logEndAndPrint(String result) {
        requestLogEntity.setUseTime(requestLogEntity.getEndTime().getTime() - requestLogEntity.getStartTime().getTime());
        requestLogEntity.setResult(result);
        log.info("{}请求总用时:{}ms", requestLogEntity.getUrl(), requestLogEntity.getUseTime());
        requestLogger.info(JacksonUtil.toJson(requestLogEntity));
    }



    //记录成功时的日志信息
    public void doSuccess(String result) {
        this.requestLogEntity.setEndTime(new Date());
        this.requestLogEntity.setStatus(RequestEnum.SUCCESS);
        try {
            //自定义回调
            success(result);
			//处理请求
            handleRetryEntity(RequestEnum.SUCCESS);
        } catch (Exception e) {
            this.requestLogEntity.setStatus(RequestEnum.FAIL);
            handleRetryEntity(RequestEnum.FAIL);
            throw e;
        }finally {
            logEndAndPrint(result);
        }
    }


	//记录失败时的信息
    public void doFail(String result) {
        this.requestLogEntity.setEndTime(new Date());
        handleRetryEntity(RequestEnum.FAIL);
        this.requestLogEntity.setStatus(RequestEnum.FAIL);
        try {
            //失败回调
            fail(result);
        } finally {
            logEndAndPrint(result);
        }
    }

   
    public void doFail() {
        doFail(null);
    }

    //HttpClinet FutureCallback的回调
    @Override
    public void completed(HttpResponse result) {
        try {
            HttpEntity entity = result.getEntity();
            String string = EntityUtils.toString(entity);
            //自定义错误处理,以<html> 开头则为nginx失败页面
            if (HttpUtil.isError(string)) {
                doFail(string);
            } else {
                doSuccess(string);
            }
        } catch (Exception e) {
            e.printStackTrace();
            log.error("completed error", e);
            doFail();
        }
    }


    //日志处理 填入请求结束时间，请求用时等数据
    protected void handleRetryEntity(RequestEnum requestEnum){
        if (this.nowRetryEntity != null){
            this.nowRetryEntity.setEndTime(new Date());
            this.nowRetryEntity.setUseTime(this.nowRetryEntity.getEndTime().getTime() - this.nowRetryEntity.getStartTime().getTime());
            this.nowRetryEntity.setStatus(requestEnum);
        }
    }

    //失败重试
    protected void doRetry() {
        handleRetryEntity(RequestEnum.FAIL);
        retryUse++;
        RequestLogEntity retryEntity = new RequestLogEntity();
        retryEntity.setStartTime(new Date());
        retryEntity.setRetryCount(retryUse);
        this.requestLogEntity.getRetryList().add(retryEntity);
        this.requestLogEntity.setRetryCount(retryUse);
        this.nowRetryEntity = retryEntity;

        log.info("{}超时重试第{}次", request.getURI().toString(), retryUse);
        HttpUtil.asyncExecute(request, this);
    }

    @Override
    public void failed(Exception ex) {
        if (ex instanceof SocketTimeoutException && retryUse < retryCount && retryCount != -1) {
            //超时并且总请求次数小于3，执行重试
            doRetry();
        } else {
            ex.printStackTrace();
            //超时后的异常处理
            doFail();
        }
    }

    //按失败处理
    @Override
    public void cancelled() {
        doFail();
    }

    //调用实现类的success的方法
    @Override
    public void success(String result) {
        callBack.success(result);
    }

    //调用实现类的fail的方法
    @Override
    public void fail(String result) {
        callBack.fail(result);
    }

    public HttpUriRequest getRequest() {
        return request;
    }
```

## 简化的回调接口

```java
public interface RetryFutureCallBack {
    //成功回调
    void success(String result);
    //失败回调
    void fail(String result);
}
```

## 不通过回调，通过阻塞的方式获取结果

```java
public class EmptyRetryFutureCallBack implements RetryFutureCallBack {
    private String result;

    private boolean completed;

    
    public synchronized String getResult() throws InterruptedException {
        while (!completed) {
            //超时等待3s
            wait(1000 * 3);
        }
        return result;
    }

    private synchronized void handleResult(String result) {
        this.completed = true;
        this.result = result;
        notifyAll();
    }

    @Override
    public void success(String result) {
        handleResult(result);
    }

    @Override
    public void fail(String result) {
        handleResult(result);
    }
}

```

## 日志类

```java
@Data
@Slf4j
public class RequestLogEntity {
    //请求开始时间
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "GMT+8")
    private Date startTime;
    //请求结束时间
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "GMT+8")
    private Date endTime;
   	//用时ms
    private Long useTime;
    //请求域名或者主机ip
    private String host;
    //请求url参数
    private Map<String, Object> params;
    @JsonIgnore
    //请求url
    private String url;
    //请求结果
    private String result;
    //重试次数
    private Integer retryCount;
    //解析数据后，数据的结果集大小
    private int resultSize;
    /**
     * 状态
     */
    private RequestEnum status;
    private List<RequestLogEntity> retryList;

    public void generateUrlInfo(String url) {
        this.url = url;
        //获取host
        if (StringUtils.isBlank(url))
            return;
        try {
            String[] split = StringUtils.split(url, "?");
            if (split == null)
                return;
            if (split.length > 0) {
                this.host = split[0];
            }
            if (split.length > 1) {
                generateParam(split[1]);
            }

        } catch (Exception e) {
            log.warn("解析url失败", e);
        }

    }

    private void generateParam(String paramString) {
        String[] split = StringUtils.split(paramString, "&");
        this.params = new HashMap<>();
        for (String item : split) {
            String[] itemSplit = StringUtils.split(item, "=");
            if (itemSplit == null || itemSplit.length == 0)
                continue;
            this.params.put(itemSplit[0], itemSplit.length == 2 ? itemSplit[1] : "");
        }

    }
}

```

## 自定义回调类实现

```java
@Sl4j
public abstract class MyCallBack implements RetryFutureCallBack {
     public void success(String result) {
       	log.info("成功请求结果:{}",result);
    }
    
    public void fail(String result){
        log.error("请求失败结果:{}",result);
    }

}
```

## HttpUtil类

```java

@Slf4j
public class HttpUtil {

    private static final String APPLICATION_JSON = "application/json";

    private static final String CONTENT_TYPE_FORM_URL = "application/x-www-form-urlencoded";

    private static volatile CloseableHttpAsyncClient client = null;

    private static final PoolingHttpClientConnectionManager connectionManager = new PoolingHttpClientConnectionManager();

    // setConnectTimeout表示设置建立连接的超时时间
    // setConnectionRequestTimeout表示从连接池中拿连接的等待超时时间
    // setSocketTimeout表示发出请求后等待对端应答的超时时间
    public static int CONNECT_TIMEOUT = 3_000;
    public static int CONNECTION_REQUEST_TIMEOUT = 3_000;
    public static int SOCKET_TIMEOUT = 3_000;
    private static final RequestConfig requestConfig = RequestConfig
            .custom()
            .setConnectTimeout(CONNECT_TIMEOUT)
            .setConnectionRequestTimeout(CONNECTION_REQUEST_TIMEOUT)
            .setSocketTimeout(SOCKET_TIMEOUT)
            .build();


    static {
        // 总连接池数量
        connectionManager.setMaxTotal(1000);
        //将每个路由的默认最大连接数增加到500
        connectionManager.setDefaultMaxPerRoute(500);
        //连接存活时间
        connectionManager.setValidateAfterInactivity(60_000);
    }


    private static final Object lock = new Object();

    public static CloseableHttpAsyncClient getHttpAsyncClient() {
        if (client == null) {
            synchronized (lock) {
                if (client == null) {


                    //配置io线程
                    IOReactorConfig ioReactorConfig = IOReactorConfig.custom().
                            setIoThreadCount(Runtime.getRuntime().availableProcessors() * 2)
                            .setSoKeepAlive(true)
                            .build();
                    //设置连接池大小
                    ConnectingIOReactor ioReactor = null;
                    try {
                        ioReactor = new DefaultConnectingIOReactor(ioReactorConfig);
                    } catch (IOReactorException e) {
                        e.printStackTrace();
                    }

                    PoolingNHttpClientConnectionManager connManager = new PoolingNHttpClientConnectionManager(ioReactor);
                    connManager.setMaxTotal(1000);//最大连接数设置
                    connManager.setDefaultMaxPerRoute(500);//per route最大连接数设置
                    client = HttpAsyncClients.custom()
                            .setConnectionManager(connManager)
//                            .setKeepAliveStrategy(keepAliveStrategy)
                            .setDefaultRequestConfig(requestConfig)
                            .build();
                    client.start();


                    //开启监控线程,对异常和空闲线程进行关闭
                    ScheduledExecutorService monitorExecutor = Executors.newScheduledThreadPool(1);
                    monitorExecutor.scheduleAtFixedRate(new TimerTask() {
                        @Override
                        public void run() {
                            //关闭异常连接
                            connectionManager.closeExpiredConnections();
                            //关闭5s空闲的连接
                            connectionManager.closeIdleConnections(5, TimeUnit.SECONDS);
                            log.debug("close expired and idle for over 5s connection");
                        }
                    }, 3, 3, TimeUnit.SECONDS);
                }
            }
        }
        return client;
    }


    public static Future<HttpResponse> asyncExecute(RetryFutureProxy retryFutureProxy) {

        CloseableHttpAsyncClient httpAsyncClient = getHttpAsyncClient();
        return httpAsyncClient.execute(retryFutureProxy.request, retryFutureProxy);
    }

    public static Future<HttpResponse> asyncExecute(HttpUriRequest request, FutureCallback<HttpResponse> callback) {
        CloseableHttpAsyncClient httpAsyncClient = getHttpAsyncClient();
        return httpAsyncClient.execute(request, callback);
    }


    public static boolean isError(String result) {
        /*
         * nginx 500 页面
         */
        if (result != null && result.startsWith("<html>")) {
            log.info(result);
            return true;
        } else {
            return false;
        }
    }

    //阻塞方式获取结果内容
    private static String httpRequest(String urlString) {
        HttpGet httpGet = new HttpGet(urlString);
        EmptyRetryFutureCallBack callBack = new EmptyRetryFutureCallBack();
        asyncExecute(httpGet, new RetryFutureProxy(httpGet, callBack));
        String result = null;
        try {
            //HttpEntity被EntityUtils.toString(entity)消费过了，采用这种方式获取结果集
            result = callBack.getResult();
        } catch (Exception e) {
            e.printStackTrace();
            log.error("httpUtil exception", e);
        }
        return result;

    }



}

```

## 调用

```java
public static void main(String[] args){
	HttpGet httpGet =new HttpGet("http://www.baidu.com");
	RetryFutureProxy retryFutureProxy = new RetryFutureProxy(2,httpGet,new MyCallBack() );
	HttpUtil.asyncExecute(retryFutureProxy);
	//防止主线程结束，回调还没结束
	Thread.sleep(3000);
}
```

## 阻塞式调用

```java
public static void main(String[] args){
	String result = HttpUtil.httpRequest("http://www.baidu.com");
	System.out.println(result);
}
```
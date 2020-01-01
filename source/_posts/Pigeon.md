---
title: Pigeon
categories:
  - 微服务
tags:
  - 动态代理
  - IO模型
date: 2019-11-05 17:08:08
---

# Pigeon

调用者会有以下的配置，ReferenceBean是代理类，被代理的是远程服务。

<!-- more --> 

<embed src="Pigeon.svg" width="100%" type="image/svg+xml"/>
```xml
<bean id="echoService" class="com.dianping.pigeon.remoting.invoker.config.spring.ReferenceBean" init-method="init">
    <!-- 服务全局唯一的标识url，默认是服务接口类名，必须设置 -->
	<property name="url" value="http://service.dianping.com/demoService/echoService_1.0.0" />
	<!-- 接口名称，必须设置 -->
	<property name="interfaceName" value="com.dianping.pigeon.demo.EchoService" />
	<!-- 超时时间，毫秒，默认5000，建议自己设置 -->
	<property name="timeout" value="2000" />
	<!-- 序列化，hessian/fst/protostuff，默认hessian，可不设置-->
	<property name="serialize" value="hessian" />
	<!-- 调用方式，sync/future/callback/oneway，默认sync，可不设置 -->
	<property name="callType" value="sync" />
	<!-- 失败策略，快速失败failfast/失败转移failover/失败忽略failsafe/并发取最快返回forking，默认failfast，可不设置 -->
	<property name="cluster" value="failfast" />
	<!-- 是否超时重试，默认false，可不设置 -->
	<property name="timeoutRetry" value="false" />
	<!-- 重试次数，默认1，可不设置 -->
	<property name="retries" value="1" />
</bean>
```





```java
//ReferenceBean代理类的初始化
public void init() throws Exception {
    if (StringUtils.isBlank(interfaceName)) {
        throw new IllegalArgumentException("invalid interface:" + interfaceName);
    }
    this.objType = ClassUtils.loadClass(this.classLoader, this.interfaceName.trim());
    InvokerConfig<?> invokerConfig = new InvokerConfig(this.objType, this.url, this.timeout, this.callType,
                                                       this.serialize, this.callback, this.suffix, this.writeBufferLimit, this.loadBalance, this.cluster,
                                                       this.retries, this.timeoutRetry, this.vip, this.version, this.protocol);
    invokerConfig.setClassLoader(classLoader);
    invokerConfig.setSecret(secret);
    invokerConfig.setRegionPolicy(regionPolicy);

    if (!CollectionUtils.isEmpty(methods)) {
        Map<String, InvokerMethodConfig> methodMap = new HashMap<String, InvokerMethodConfig>();
        invokerConfig.setMethods(methodMap);
        for (InvokerMethodConfig method : methods) {
            methodMap.put(method.getName(), method);
        }
    }

    checkMock(); // 降级配置检查
    invokerConfig.setMock(mock);
    checkRemoteAppkey();
    invokerConfig.setRemoteAppKey(remoteAppKey);

    //获取被代理的服务
    this.obj = ServiceFactory.getService(invokerConfig);
    configLoadBalance(invokerConfig);
}


//获取被代理的服务
public Object proxyRequest(InvokerConfig<?> invokerConfig) throws SerializationException {
    return Proxy.newProxyInstance(ClassUtils.getCurrentClassLoader(invokerConfig.getClassLoader()),
                                  new Class[] { invokerConfig.getServiceInterface() },
                                  new ServiceInvocationProxy(invokerConfig,InvokerProcessHandlerFactory.selectInvocationHandler(invokerConfig)));
}
```





客户端请求的发送是责任链模式，发送前会经过许多的Filter

```java
public static void init() {
    if (!isInitialized) {
        if (Constants.MONITOR_ENABLE) {
            registerBizProcessFilter(new RemoteCallMonitorInvokeFilter());
        }
        registerBizProcessFilter(new TraceFilter());
        registerBizProcessFilter(new FaultInjectionFilter());//模拟故障
        registerBizProcessFilter(new DegradationFilter());//降级
        registerBizProcessFilter(new ClusterInvokeFilter());
        registerBizProcessFilter(new GatewayInvokeFilter());
        registerBizProcessFilter(new ContextPrepareInvokeFilter());
        registerBizProcessFilter(new SecurityFilter());
        registerBizProcessFilter(new RemoteCallInvokeFilter());//调用远程服务
        bizInvocationHandler = createInvocationHandler(bizProcessFilters);
        isInitialized = true;
    }
}
```





```java
//客户端接收响应
public void receiveResponse(InvocationResponse response) {
    //同个客户端可能同时会有多个请求在等待结果，那么如何区分返回的结果是哪个请求的呢
    //每个请求发送出去都会带有一个唯一标识，在响应返回时会带上这个标识
    //标识相同说明请求和响应是一对的
    //标识的实现是原子变量，在JVM维度中是唯一的
    //标识在集群是会重复的，但不影响配对。因为通信协议会保证响应返回对应的机器。
    RemoteInvocationBean invocationBean = invocations.get(response.getSequence());//
    //...
    try {
        Callback callback = invocationBean.callback;
        callback.callback(response);
        callback.run();
    } finally {
        invocations.remove(response.getSequence());
    }
    //...
}
```





ReentrantLock配合Condition实现等待。

```java
//SYNC、FUTURE都会使用到以下等待方法
protected InvocationResponse waitResponse(long timeoutMillis) throws InterruptedException {
	//...
    lock.lock();
    try {
        while (!isDone()) {
            condition.await(timeoutLeft, TimeUnit.MILLISECONDS);
        }
    } finally {
        lock.unlock();
    }
	//...
    return this.response;
}
```





发送请求都是调用netty的channel.write()，该方法是异步的，会在另一个线程里运行，待io结束之后会调用用户写好的listener。



```java
//io结束，netty进行回调，释放条件锁
public void run() {
    lock.lock();
    try {
        this.done = true;
        if (condition != null) {
            condition.signal();
        }
    } finally {
        lock.unlock();
    }
}
```


初始化一个单例，用于全局的网络框架
![构建单例模式](https://chenyoyoy.github.io/image/2017/03/WechatIMG6.jpeg)

构建单个网络请求，发起请求，获取数据
![发起一个普通的网络请求](https://chenyoyoy.github.io/image/2017/03/WechatIMG18.jpeg)

>流程控制

![](https://chenyoyoy.github.io/image/2017/03/OkHttp流程控制.png)

Dispatcher里面放有一个线程池-executorService，异步网络请求的整个过程，就是在线程池里完成的。

enQueue的实际执行是把请求封装成一个Runnable，丢到dispatcher中去执行
![发起一个普通的网络请求](https://chenyoyoy.github.io/image/2017/03/WechatIMG1.jpeg)

线程池与外面请求的请求队列的读取模型，并不是生产消费模型，而是基于 事件通知的。

一个请求扔过来的时候，如果线程池有空闲的线程，就直接执行，并且放到runningSyncCalls中 ；
如果当前线程池满了，先把请求放到readyAsyncCalls中，等待执行中的请求完成，产生finish事件回来，通知执行等待队列中的请求。

![](https://chenyoyoy.github.io/image/2017/03/dispacher执行call.png)

请求的执行过程，从AsynCall的excute方法开始，getResponseWithInterceptorChain() 
![](https://chenyoyoy.github.io/image/2017/03/execute.png)


>拦截器链的调用


![发起一个普通的网络请求](https://chenyoyoy.github.io/image/2017/03/WechatIMG2.jpeg)

将代码转变成图形，就变成了以下这个样子。

![发起一个普通的网络请求](https://chenyoyoy.github.io/image/2017/03/拦截器的调用顺序.png)

* client.interceptors() 请求框架级别的拦截器，能够做到全局的请求处理
* retryAndFollowUpInterceptor 重定向以及重试拦截器，网络请求发生错误的时候会自动重试
* BridgeInterceptor 一些Header字段的填入，包括cookie的自动写入
* CacheInterceptor 缓存拦截器，对请求结果进行保存，以及使用缓存策略从缓存中读取数据
* ConnectInterceptor 连接拦截器，获得一个连接，用于数据传输
* networkInterceptors 网络请求拦截器，能获取到网络请求最原始的数据
* CallServerInterceptor 网络请求的最后一步，将数据写入流中，读取服务器返回的流数据

getResponseWithInterceptorChain，中chain.proceed(originalRequest)，开始发起链式调用；
![发起一个普通的网络请求](https://chenyoyoy.github.io/image/2017/03/interceptor.jpeg)

proceed方法中会new 新的Chain ,同时从interceptors里面 取出下一个 interceptor，继续调用intercept,interceptor里面再调用proceed ，于是形成了调用链； 

每次new 一个 chain 的时候 会传递 index+1 ，到下一个Interceptor中，不断地往下传递，实际的index 就会往上递增，从而实现从interceptors中依次取出interceptor的目的。

以重试拦截器为例，intercept中一定要调用 chain.proceed ;

![发起一个普通的网络请求](https://chenyoyoy.github.io/image/2017/03/WechatIMG11.jpeg)

在CacheInterceptor或者CallServerInterceptor获取到数据后，会通过调用链将结果返回到Call里面，call 通过CallBack回调给业务层

![](https://chenyoyoy.github.io/image/2017/03/请求结果的回调.png)

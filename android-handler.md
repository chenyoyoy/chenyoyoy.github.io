###1. Handler Looper MessageQueue 模型
---

![线程通信模型](http://upload-images.jianshu.io/upload_images/1239900-0a9bda4d2e62c277.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 -  Handler  提供sendMessage方法，将消息放置到队列中 ；
 -  Handler  提供handleMessage方法，定义个各种消息的处理方式；
 -  Looper.prepare(); 为当前线程生成一个消息队列；
 -  Looper.loop() ；循环从消息队列中获取消息，交给Handler处理；此时线程处于无限循环中，不停的从MessageQueue中获取Message 消息 ；如果没有消息就阻塞 ；

-  MessageQueue 提供enqueueMessage 方法，将消息根据时间放置到队列中；
-  MessageQueue 提供next方法，从队列中获取消息，没有消息的时候阻塞；


### 2. Handler的最基本使用方式
---
    Handler handler = new Handler(){
    Override
    public void handleMessage(Message msg) {
    super.handleMessage(msg); 
    textView.setText("") ;
    }} ;

    new Thread(new Runnable() {
    @Override
     public void run() {
     //do something
     Message message = new Message() ;
     message.arg1 = 0 ;
     message.arg2 = 0 ; 
     message.what = 1 ; 
     message.obj = new Object() ;
     handler.sendMessage(message);
    }}).start();
</code>
   - 上面的代码，展示了最基本的android 线程通信方式，在线程中完成耗时操作后，通过Handler刷新主界面的过程；
   - 应用场景：一次性的耗时操作，完成后线程就销毁掉了

### 3. Handler Looper 的配合使用
---
     官方例子：
     class LooperThread extends Thread {
     public Handler mHandler;
     public void run() {
     Looper.prepare();
     mHandler = new Handler() {
     public void handleMessage(Message msg) {
     }
    };
    Looper.loop();
    }

>**应用场景：**
较长生命周期的线程使用，需要不停得和其他线程进行通信；
线程周期较长，里面的耗时任务会一个个的执行，可以做到异步转同步的作用；
比较常见的使用情景如下：

![Handler 线程间通信](http://upload-images.jianshu.io/upload_images/1239900-2f217d371b988612.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>说明：
其他线程通过 mThreadHandler 给任务线程发送消息，执行任务；
任务线程得到任务执行结果后，通过mOutMessenger（其他线程Handler 封装而来） 发送结果；

###3.  一些额外的知识点
---
3.1 主线程也是无限循环队列
      默认的，一个进程会有一个主线程，主线程生成的时候就初始化了消息队列，之后的各种组件的初始化，事件的传递，都是通过主线程中的Handler 发送消息到消息队列中后执行的；

3.2 线程的退出
      Looper.prepare() 生成的消息队列是可以退出的，调用Looper.quit() 后能够结束循环，退出线程；
      主线程的消息循环是不能跳出来的，即不可能通过结束主线程的无限循环，退出主线程，达到退出app的目的；

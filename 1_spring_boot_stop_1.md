# Spring boot 2.0 之优雅停机一

### 什么叫优雅停机

简单说就是，在对应用进程发送停止指令之后，能保证正在执行的业务操作不受影响。应用接收到停止指令之后的步骤应该是，停止接收访问请求，等待已经接收到的请求处理完成，并能成功返回，这时才真正停止应用。

这种完美的应用停止方式如何实现呢？就Java语言生态来说，底层的技术是支持的，所以我们才能实现在Java语言之上的各个web容器的优雅停机。

在普通的外置的tomcat中，有shutdown脚本提供优雅的停机机制，但是我们在使用Spring boot的过程中发现web容器都是内置（当然也可使用外置，但是不推荐），这种方式提供简单的应用启动方式，方便的管理机制，非常适用于微服务应用中，但是默认没有提供优雅停机的方式。这也是本文探索这个问题的根本原因。

应用是否是实现了优雅停机，如何才能验证呢？这需要一个处理时间较长的业务逻辑，模拟这样的逻辑应该很简单，使用线程sleep或者长时间循环。我的模拟业务逻辑代码如下：

```java
@GetMapping(value = "/sleep/one", produces = "application/json")
public ResultEntity<Long> sleepOne(String systemNo){
    logger.info("模拟业务处理1分钟，请求参数：{}", systemNo);
    Long serverTime = System.currentTimeMillis();
    while (System.currentTimeMillis() < serverTime + (60 * 1000)){
        logger.info("正在处理业务，当前时间：{}，开始时间：{}", System.currentTimeMillis(), serverTime);
    }
    ResultEntity<Long> resultEntity = new ResultEntity<>(serverTime);
    logger.info("模拟业务处理1分钟，响应参数：{}", resultEntity);    return resultEntity;
}
```

验证方式就是，在触发这个接口的业务处理之后，业务逻辑处理时间长达1分钟，需要在处理结束前，发起停止指令，验证是否能够正常返回。验证时所使用的kill指令：`kill -2（Ctrl + C）`、`kill-15`、`kill -9`。

### Java 语言的优雅停机

从上面的介绍中我们发现，Java语言本身是支持优雅停机的，这里就先介绍一下普通的java应用是如何实现优雅停止的。

当我们使用`kill PID`的方式结束一个Java应用的时候，JVM会收到一个停止信号，然后执行shutdownHook的线程。一个实现示例如下：

```java
public class ShutdownHook extends Thread {    
    private Thread mainThread;    
    private boolean shutDownSignalReceived; 

    @Override
    public void run() {
        System.out.println("Shut down signal received."); 
        this.shutDownSignalReceived=true;
        mainThread.interrupt();        
        try {
            mainThread.join(); //当收到停止信号时，等待mainThread的执行完成
        } catch (InterruptedException e) {
        }
        System.out.println("Shut down complete.");
    }    
    public ShutdownHook(Thread mainThread) {        
        super();        
        this.mainThread = mainThread;        
        this.shutDownSignalReceived = false;
        Runtime.getRuntime().addShutdownHook(this);
    }    
    public boolean shouldShutDown(){        
        return shutDownSignalReceived;
    }
}
```

其中关键语句`Runtime.getRuntime().addShutdownHook(this);`，注册一个JVM关闭的钩子，这个钩子可以在以下几种场景被调用：

1. 程序正常退出
2. 使用System.exit\(\)
3. 终端使用Ctrl+C触发的中断
4. 系统关闭
5. 使用Kill pid命令干掉进程

测试shutdownHook的功能，代码示例：

```java
public class TestMain {
    private ShutdownHook shutdownHook;    
    public static void main( String[] args ) {
        TestMain app = new TestMain();
        System.out.println( "Hello World!" );
        app.execute();
        System.out.println( "End of main()" );
    }    
    public TestMain(){        
        this.shutdownHook = new ShutdownHook(Thread.currentThread());
    }    
    public void execute(){        
        while(!shutdownHook.shouldShutDown()){
            System.out.println("I am sleep");            
            try {
                Thread.sleep(1*1000);
            } catch (InterruptedException e) {
                System.out.println("execute() interrupted");
            }
            System.out.println("I am not sleep");
        }
        System.out.println("end of execute()");
    }
}
```

启动测试代码，之后再发送一个中断信号，控制台输出：

```text
I am sleep
I am not sleep
I am sleep
I am not sleep
I am sleep
I am not sleep
I am sleep
Shut down signal received.
execute() interrupted
I am not sleep
end of execute()
End of main()
Shut down complete.

Process finished with exit code 130 (interrupted by signal 2: SIGINT)
```

可以看出，在接收到中断信号之后，整个main函数是执行完成的。

### actuator/shutdown of Spring boot

我们知道了java本身在支持优雅停机上的能力，然后在Spring boot中又发现了`actuator/shutdown`的管理端点。于是我把优雅停机的功能寄希望于此，开始配置测试，开启配置如下：

```yaml
management:
  server:
    port: 10212
    servlet:
      context-path: /
    ssl:
      enabled: false
  endpoints:
    web:
      exposure:        
      include: "*"
  endpoint:
    health:
      show-details: always
    shutdown:
      enabled: true #启用shutdown端点
```

测试结果很失望，并没有实现优雅停机的功能，就是将普通的kill命令，做成了HTTP端点。于是开始查看Spring boot的官方文档和源代码，试图找到它的原因。

在官方文档上对shutdown端点的介绍：

```text
shutdown    Lets the application be gracefully shutdown.
```

从此介绍可以看出，设计上应该是支持优雅停机的。但是为什么现在还不够优雅，在github上托管的Spring boot项目中发现，有一个[issue](https://github.com/spring-projects/spring-boot/issues/4657)一直处于打开状态，已经两年多了，里面很多讨论，看完之后发现在Spring boot中完美的支持优雅停机不是一件容易的事，首先Spring boot支持web容器很多，其次对什么样的实现才是真正的优雅停机，讨论了很多。想了解更多的同学，把这个issue好好阅读一下。

这个issue中还有一个重要信息，就是这个issue曾经被加入到2.0.0的milestone中，后来由于没有完成又移除了，现在状态是被添加在2.1.0的milestone中。我测试的版本是2.0.1，期待官方给出完美的优雅停机方案。

### Spring boot 优雅停机

虽然官方暂时还没有提供优雅停机的支持，但是我们为了减少进程停止对业务的影响，还是要给出能满足基本需求的方案来。

针对tomcat的解决方案是：

```java
package com.epay.demox.unipay.provider;
import org.apache.catalina.connector.Connector;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.web.embedded.tomcat.TomcatConnectorCustomizer;
import org.springframework.context.ApplicationListener;
import org.springframework.context.event.ContextClosedEvent;
import org.springframework.stereotype.Component;
import java.util.concurrent.Executor;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * @Author: guoyankui
 * @DATE: 2018/5/20 12:59 PM
 *
 * 优雅关闭 Spring Boot tomcat
 */@Componentpublic class GracefulShutdownTomcat implements TomcatConnectorCustomizer, ApplicationListener<ContextClosedEvent> {    
     private final Logger log = LoggerFactory.getLogger(GracefulShutdownTomcat.class);    
     private volatile Connector connector;    
     private final int waitTime = 30;  

     @Override
     public void customize(Connector connector) {        
         this.connector = connector;
     }    

     @Override
     public void onApplicationEvent(ContextClosedEvent contextClosedEvent) { 
         this.connector.pause();
         Executor executor = this.connector.getProtocolHandler().getExecutor(); 
         if (executor instanceof ThreadPoolExecutor) {           
             try {
                ThreadPoolExecutor threadPoolExecutor = (ThreadPoolExecutor) executor;
                threadPoolExecutor.shutdown();                
                 if (!threadPoolExecutor.awaitTermination(waitTime, TimeUnit.SECONDS)) {
                    log.warn("Tomcat thread pool did not shut down gracefully within " + waitTime + " seconds. Proceeding with forceful shutdown");
                }
            } catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

```java
public class UnipayProviderApplication {    
    public static void main(String[] args) {
        SpringApplication.run(UnipayProviderApplication.class);
    }    

    @Autowired
    private GracefulShutdownTomcat gracefulShutdownTomcat;    

    @Bean
    public ServletWebServerFactory servletContainer() {
        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory();
        tomcat.addConnectorCustomizers(gracefulShutdownTomcat);        
        return tomcat;
    }
}
```

该方案的代码来自官方issue中的讨论，添加这些代码到你的Spring boot项目中，然后再重新启动之后，发起测试请求，然后发送kill停止指令（`kill -2（Ctrl + C）`、`kill -15`）。测试结果：

1. Spring boot的健康检查，为`UP`。
2. 正在执行操作不会终止，直到执行完成。
3. 不再接收新的请求，客户端报错信息为：`Connection reset by peer`。
4. 最后正常终止进程（业务执行完成后，立即进程停止）。

从测试结果来看，是满足我们的需求的。当然如果发送指令`kill -9`，进程会立即停止。

### 结束

到此为止，对Java和Spring boot应用的优雅停机机制有了基本的认识。虽然实现了需求，但是这其中还有很多知识点需要探索，比如Spring上下文监听器，上下文关闭事件等，还有undertow提供的`GracefulShutdownHandler`的原理是什么，为什么是1分钟之后进程再停止，这些问题等研究明白，再来一篇续。如果又哪位同学能解答我的疑惑，请在评论区留言。

---

---

### 参考：

[https://cloud.spring.io/spring-cloud-static/spring-cloud-zookeeper/2.1.0.RC2/single/spring-cloud-zookeeper.html\#\_instance\_status](https://cloud.spring.io/spring-cloud-static/spring-cloud-zookeeper/2.1.0.RC2/single/spring-cloud-zookeeper.html#_instance_status)

[https://www.imooc.com/article/47291](https://www.imooc.com/article/47291)


# Spring boot 2.0 之优雅停机二

### 与Spring boot应用的不同之处

在Spring Cloud环境中的服务，一般都会注册到注册中心，客户端会定时通过注册中心来获取服务端的注册信息，来完成服务完成服务间的调用。如果在这个定时刷新的时间内，服务端下线，客户端就会调用失败，所以这个定时的频率间隔就是我们需要优化的地方

### 理想的流程

使用Spring Cloud中提供的一些方法，我们可以让停机更优雅

```text
graph LR; 
节点1 --> 服务反注册;
服务反注册 --> 等待同步,默认30s;
等待同步,默认30s --> 停止服务;
停止服务 --> 启动服务;
启动服务 --> 服务注册;
服务注册 --> 等待30s;
等待30s --> 完成
```

于是我们需要在停机的过程中增加`服务反注册`的步骤

#### 如何完成“服务反注册”？

* **方式一**:通过_service-registry_接口

`service-registry`是Spring Cloud提供给我们服务注册相关的接口，通过POST的方式，可以修改响应的注册状态，这个状态会同步到对应的注册中心

```bash
$ http POST http://localhost:8081/service-registry status=OUT_OF_SERVICE
```

调用该接口，客户端会主动修改注册中心中节点的状态

```javascript
{
  "name": "boss-risk-td",
  "id": "b8c79cdd-e519-4e4e-a907-c8896287777a",
  "address": "192.168.11.168",
  "port": 10011,
  "sslPort": null,
  "payload": {
    "@class": "org.springframework.cloud.zookeeper.discovery.ZookeeperInstance",
    "id": "application-1",
    "name": "boss-risk-td",
    "metadata": {
      "instance_status": "OUT_OF_SERVICE"  //增加了该节点，可使用该状态来实现蓝绿发布
    }
  },
  "registrationTimeUTC": 1546496763280,
  "serviceType": "DYNAMIC",
  "uriSpec": {
    "parts": [
      {
        "value": "scheme",
        "variable": true
      },
      {
        "value": "://",
        "variable": false
      },
      {
        "value": "address",
        "variable": true
      },
      {
        "value": ":",
        "variable": false
      },
      {
        "value": "port",
        "variable": true
      }
    ]
  }
}
```

可以看到zookeeper对应的数据中增加了`"instance_status": "OUT_OF_SERVICE"`，经验证只有当`instance_status`为空或者为`“UP”`的时候注册中心才会认为该节点可用

* **方式二**：直接反注册

我们要实现反注册，实际上只需要删除Zookeeper中对应节点的注册信息就可以了，Spring Cloud中也提供了相应的实现，我们可以直接调用，代码如下：

```java
@Slf4j
@RestController
public class TestController {

    @Autowired
    private ServiceRegistry serviceRegistry;

    @Autowired(required = false)
    private Registration registration;

    @GetMapping("/test")
    public String test() {
        if (registration != null) {
            serviceRegistry.deregister(registration);
            serviceRegistry.close();
        }
        return "ok";
    }
}
```

测试后发现上面的注册信息已被删除

**MQ停止监听**

通过KILL PID来停止应用时，MQ会自动停止，如下图：

```bash
2019-01-04 15:25:06.519  INFO [-,,,] 10414 --- [Thread-9] o.s.c.support.DefaultLifecycleProcessor  : Stopping beans in phase 2147483647
2019-01-04 15:25:06.523  INFO [-,,,] 10414 --- [Thread-9] o.s.a.r.l.SimpleMessageListenerContainer : Waiting for workers to finish.
2019-01-04 15:25:06.896  INFO [-,,,] 10414 --- [Thread-9] o.s.a.r.l.SimpleMessageListenerContainer : Successfully waited for workers to finish.
2019-01-04 15:25:06.934  INFO [-,,,] 10414 --- [Thread-9] o.s.j.e.a.AnnotationMBeanExporter        : Unregistering JMX-exposed beans on shutdown
2019-01-04 15:25:06.934  INFO [-,,,] 10414 --- [Thread-9] o.s.j.e.a.AnnotationMBeanExporter        : Unregistering JMX-exposed beans
2019-01-04 15:25:06.939  INFO [-,,,] 10414 --- [Thread-9] o.s.a.r.l.SimpleMessageListenerContainer : Shutdown ignored - container is not active already
```

但是默认情况下`DEFAULT_SHUTDOWN_TIMEOUT = 5000;`超时时间为5s，如果某些消息的处理时间是超过5s的，可能就会有问题，所有这个时间需要根据业务情况来重新设定，比如设置为60s：

```java
    /**
     * 关闭mq
     */
    private void shutdownMq() {
        //停止消息监听
        if (rabbitListenerEndpointRegistry != null) {
            for (MessageListenerContainer messageListenerContainer : rabbitListenerEndpointRegistry.getListenerContainers()) {
                if (messageListenerContainer instanceof SimpleMessageListenerContainer) {
                    ((SimpleMessageListenerContainer) messageListenerContainer).setShutdownTimeout(60000);
                }
            }
            rabbitListenerEndpointRegistry.stop();
        }
    }
```

**如何来和Jenkins结合**

为了方便与Jenkins结合，Jenkins最好是只需要执行通用的命令即可，其它的等待时间全部交给代码来实现

* 1、Jenkins执行脚本命令`KILL PID`,发送应用停止信号
* 2、GracefulShutdownTomcat控制等待时间，然后停机

**整合后的代码**

```java
package cn.victorplus.risk.td.support.shutdown;

import lombok.extern.slf4j.Slf4j;
import org.apache.catalina.connector.Connector;
import org.springframework.amqp.rabbit.listener.MessageListenerContainer;
import org.springframework.amqp.rabbit.listener.RabbitListenerEndpointRegistry;
import org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.web.embedded.tomcat.TomcatConnectorCustomizer;
import org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext;
import org.springframework.cloud.client.serviceregistry.Registration;
import org.springframework.cloud.client.serviceregistry.ServiceRegistry;
import org.springframework.context.ApplicationListener;
import org.springframework.context.event.ContextClosedEvent;
import org.springframework.stereotype.Component;
import org.springframework.util.CollectionUtils;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Executor;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * 优雅停机
 *
 * @author liujie
 * @date 2019/12/31
 */
@Slf4j
@Component
public class GracefulShutdownTomcat implements TomcatConnectorCustomizer, ApplicationListener<ContextClosedEvent> {

    private volatile Connector connector;

    private final int waitTime = 30;

    @Autowired(required = false)
    private RabbitListenerEndpointRegistry rabbitListenerEndpointRegistry;

    @Autowired(required = false)
    private Executor asyncExecutor;

    @Autowired(required = false)
    private ServiceRegistry serviceRegistry;

    @Autowired(required = false)
    private Registration registration;

    @Override
    public void customize(Connector connector) {
        this.connector = connector;
    }

    @Override
    public void onApplicationEvent(ContextClosedEvent contextClosedEvent) {
        if (!(contextClosedEvent.getSource() instanceof AnnotationConfigServletWebServerApplicationContext)) {
            return;
        }
        try {
            shutdownMicroService();
            long startTime = System.currentTimeMillis();
            shutdownMq();

            //停止线程池
            Executor httpExecutor = this.connector.getProtocolHandler().getExecutor();
            List<Executor> executors = new ArrayList<>();
            executors.add(httpExecutor);
            if (asyncExecutor != null) {
                executors.add(asyncExecutor);
            }
            long gapTime = System.currentTimeMillis() - startTime;
            // wait 30s(微服务缓存刷新间隔),等微服务不再向该节点转发流量
            if (gapTime < waitTime * 1000) {
                Thread.sleep(waitTime * 1000 - gapTime);
            }
            this.connector.pause();
            shutdownExecutor(executors);
        } catch (Exception e) {
            log.error("停机过程中出现异常:{}", e.getMessage(), e);
        }
    }

    /**
     * 关闭线程池
     *
     * @param executors
     */
    private void shutdownExecutor(List<Executor> executors) {
        if (CollectionUtils.isEmpty(executors)) {
            return;
        }
        for (Executor executor : executors) {
            if (executor instanceof ThreadPoolExecutor) {
                try {
                    ThreadPoolExecutor threadPoolExecutor = (ThreadPoolExecutor) executor;
                    threadPoolExecutor.shutdown();
                    if (!threadPoolExecutor.awaitTermination(waitTime, TimeUnit.SECONDS)) {
                        log.warn("thread pool did not shut down gracefully within " + waitTime + " seconds. Proceeding with forceful shutdown");
                    }
                } catch (InterruptedException ex) {
                    log.error("thread pool did not shut down gracefully", ex);
                }
            }
        }
    }

    /**
     * 关闭微服务
     */
    private void shutdownMicroService() {
        if (serviceRegistry != null && registration != null) {
            serviceRegistry.deregister(registration);
            serviceRegistry.close();
        }
    }

    /**
     * 关闭mq
     */
    private void shutdownMq() {
        //停止消息监听
        if (rabbitListenerEndpointRegistry != null) {
            for (MessageListenerContainer messageListenerContainer : rabbitListenerEndpointRegistry.getListenerContainers()) {
                if (messageListenerContainer instanceof SimpleMessageListenerContainer) {
                    ((SimpleMessageListenerContainer) messageListenerContainer).setShutdownTimeout(60000);
                }
            }
            rabbitListenerEndpointRegistry.stop();
        }
    }
}
```

**总结**

到此我们基本完成了Spring Cloud的优雅停机，但是还是不完善，有待验证，我会在后面持续改进的...


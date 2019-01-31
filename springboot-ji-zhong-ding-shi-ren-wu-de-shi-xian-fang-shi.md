# SpringBoot几种定时任务的实现方式

定时任务实现的几种方式：

* Timer：这是java自带的java.util.Timer类，这个类允许你调度一个java.util.TimerTask任务。使用这种方式可以让你的程序按照某一个频度执行，但不能在指定时间运行。一般用的较少。
* ScheduledExecutorService：也jdk自带的一个类；是基于线程池设计的定时任务类,每个调度任务都会分配到线程池中的一个线程去执行,也就是说,任务是并发执行,互不影响。
* Spring Task：Spring3.0以后自带的task，可以将它看成一个轻量级的Quartz，而且使用起来比Quartz简单许多。
* Quartz：这是一个功能比较强大的的调度器，可以让你的程序在指定时间执行，也可以按照某一个频度执行，配置起来稍显复杂。

## 使用Timer <a id="SpringBoot&#x51E0;&#x79CD;&#x5B9A;&#x65F6;&#x4EFB;&#x52A1;&#x7684;&#x5B9E;&#x73B0;&#x65B9;&#x5F0F;-&#x4F7F;&#x7528;Timer"></a>

这个目前在项目中用的较少，直接贴demo代码。具体的介绍可以查看api

```java
package com.boss.test;
 
import java.util.Date;
import java.util.Timer;
import java.util.TimerTask;
 
/**
 * @author wangwenxiang
 * 2018/12/1.
 * @description:
 */
public class TestTimer {
    public static void main(String[] args) {
        TimerTask timerTask = new TimerTask() {
            @Override
            public void run() {
                System.out.println("task  run:"+ new Date());
            }
        };
        Timer timer = new Timer();
        //安排指定的任务在指定的时间开始进行重复的固定延迟执行。这里是每3秒执行一次
        timer.schedule(timerTask,10,3000);
    }
}
```

## 使用ScheduledExecutorService <a id="SpringBoot&#x51E0;&#x79CD;&#x5B9A;&#x65F6;&#x4EFB;&#x52A1;&#x7684;&#x5B9E;&#x73B0;&#x65B9;&#x5F0F;-&#x4F7F;&#x7528;ScheduledExecutorService"></a>

该方法跟Timer类似，直接看demo：

```java
package com.boss.test;
 
import java.util.Date;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
 
/**
 * @author wangwenxiang
 * 2018/12/1.
 * @description:
 */
public class TestScheduledExecutorService {
    public static void main(String[] args) {
        ScheduledExecutorService service = Executors.newSingleThreadScheduledExecutor();
        // 参数：1、任务体 2、首次执行的延时时间
        //      3、任务执行间隔 4、间隔时间单位
        service.scheduleAtFixedRate(()->System.out.println("task ScheduledExecutorService "+new Date()), 0, 3, TimeUnit.SECONDS);
    }
}
```

## 使用Spring Task <a id="SpringBoot&#x51E0;&#x79CD;&#x5B9A;&#x65F6;&#x4EFB;&#x52A1;&#x7684;&#x5B9E;&#x73B0;&#x65B9;&#x5F0F;-&#x4F7F;&#x7528;SpringTask"></a>

\#\#简单的定时任务

在SpringBoot项目中，我们可以很优雅的使用注解来实现定时任务，首先创建项目，导入依赖：

```markup
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
  </dependency>
  <dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

创建任务类：

```java
package com.boss.test;
 
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
 
/**
 * @author wangwenxiang
 * 2018/12/1.
 * @description:
 */
@Component
public class ScheduledService {
    private Logger log = LoggerFactory.getLogger(ScheduledService.class);
    @Scheduled(cron = "0/5 * * * * *")
    public void scheduled(){
        log.info("=====>>>>>使用cron  {}",System.currentTimeMillis());
    }
    @Scheduled(fixedRate = 5000)
    public void scheduled1() {
        log.info("=====>>>>>使用fixedRate{}", System.currentTimeMillis());
    }
    @Scheduled(fixedDelay = 5000)
    public void scheduled2() {
        log.info("=====>>>>>fixedDelay{}",System.currentTimeMillis());
    }
}
```

### 多线程执行 <a id="SpringBoot&#x51E0;&#x79CD;&#x5B9A;&#x65F6;&#x4EFB;&#x52A1;&#x7684;&#x5B9E;&#x73B0;&#x65B9;&#x5F0F;-&#x591A;&#x7EBF;&#x7A0B;&#x6267;&#x884C;"></a>

在传统的Spring项目中，我们可以在xml配置文件添加task的配置，而在SpringBoot项目中一般使用config配置类的方式添加配置，所以新建一个AsyncConfig类

```java
package com.boss.test;
 
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
 
import java.util.concurrent.Executor;
 
/**
 * @author wangwenxiang
 * 2018/12/1.
 * @description:
 */
@Configuration
@EnableAsync
public class AsyncConfig {
    /*
    此处成员变量应该使用@Value从配置中读取
     */
    private int corePoolSize = 10;
    private int maxPoolSize = 200;
    private int queueCapacity = 10;
    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(corePoolSize);
        executor.setMaxPoolSize(maxPoolSize);
        executor.setQueueCapacity(queueCapacity);
        executor.initialize();
        return executor;
    }
}
```

`@Configuration`：表明该类是一个配置类  
`@EnableAsync`：开启异步事件的支持

然后在定时任务的类或者方法上添加`@Async` 。最后重启项目，每一个任务都是在不同的线程中

### 执行时间的配置 <a id="SpringBoot&#x51E0;&#x79CD;&#x5B9A;&#x65F6;&#x4EFB;&#x52A1;&#x7684;&#x5B9E;&#x73B0;&#x65B9;&#x5F0F;-&#x6267;&#x884C;&#x65F6;&#x95F4;&#x7684;&#x914D;&#x7F6E;"></a>

在上面的定时任务中，我们在方法上使用@Scheduled注解来设置任务的执行时间，并且使用三种属性配置方式：

1. fixedRate：定义一个按一定频率执行的定时任务
2. fixedDelay：定义一个按一定频率执行的定时任务，与上面不同的是，改属性可以配合`initialDelay`， 定义该任务延迟执行时间。
3. cron：通过表达式来配置任务执行时间

### cron表达式详解 <a id="SpringBoot&#x51E0;&#x79CD;&#x5B9A;&#x65F6;&#x4EFB;&#x52A1;&#x7684;&#x5B9E;&#x73B0;&#x65B9;&#x5F0F;-cron&#x8868;&#x8FBE;&#x5F0F;&#x8BE6;&#x89E3;"></a>

一个cron表达式有至少6个（也可能7个）有空格分隔的时间元素。按顺序依次为：

* 秒（0~59）
* 分钟（0~59）
* 3 小时（0~23）
* 4 天（0~31）
* 5 月（0~11）
* 6 星期（1~7 1=SUN 或 SUN，MON，TUE，WED，THU，FRI，SAT）
* 年份（1970－2099）

其中每个元素可以是一个值\(如6\),一个连续区间\(9-12\),一个间隔时间\(8-18/4\)\(/表示每隔4小时\),一个列表\(1,3,5\),通配符。由于”月份中的日期”和”星期中的日期”这两个元素互斥的,必须要对其中一个设置。配置实例：

* 每隔5秒执行一次：_/5_ \* ?
* 每隔1分钟执行一次：0 _/1_ ?
* 0 0 10,14,16 ? 每天上午10点，下午2点，4点
* 0 0/30 9-17 ? 朝九晚五工作时间内每半小时
* 0 0 12 ? \* WED 表示每个星期三中午12点
* “0 0 12 ?” 每天中午12点触发
* “0 15 10 ? “ 每天上午10:15触发
* “0 15 10 ?” 每天上午10:15触发
* “0 15 10 ? \*” 每天上午10:15触发
* “0 15 10 ? 2005” 2005年的每天上午10:15触发
* “0 _14_ \* ?” 在每天下午2点到下午2:59期间的每1分钟触发
* “0 0/5 14 ?” 在每天下午2点到下午2:55期间的每5分钟触发
* “0 0/5 14,18 ?” 在每天下午2点到2:55期间和下午6点到6:55期间的每5分钟触发
* “0 0-5 14 ?” 在每天下午2点到下午2:05期间的每1分钟触发
* “0 10,44 14 ? 3 WED” 每年三月的星期三的下午2:10和2:44触发
* “0 15 10 ? \* MON-FRI” 周一至周五的上午10:15触发
* “0 15 10 15 \* ?” 每月15日上午10:15触发
* “0 15 10 L \* ?” 每月最后一日的上午10:15触发
* “0 15 10 ? \* 6L” 每月的最后一个星期五上午10:15触发
* “0 15 10 ? \* 6L 2002-2005” 2002年至2005年的每月的最后一个星期五上午10:15触发
* “0 15 10 ? \* 6\#3” 每月的第三个星期五上午10:15触发

有些子表达式能包含一些范围或列表

> 例如：子表达式（天（星期））可以为 “MON-FRI”，“MON，WED，FRI”，“MON-WED,SAT”

“\*”字符代表所有可能的值  
“/”字符用来指定数值的增量

> 例如：在子表达式（分钟）里的“0/15”表示从第0分钟开始，每15分钟  
> 在子表达式（分钟）里的“3/20”表示从第3分钟开始，每20分钟（它和“3，23，43”）的含义一样

“？”字符仅被用于天（月）和天（星期）两个子表达式，表示不指定值  
当2个子表达式其中之一被指定了值以后，为了避免冲突，需要将另一个子表达式的值设为“？”

“L” 字符仅被用于天（月）和天（星期）两个子表达式，它是单词“last”的缩写  
如果在“L”前有具体的内容，它就具有其他的含义了。

> 例如：“6L”表示这个月的倒数第６天  
> 注意：在使用“L”参数时，不要指定列表或范围，因为这会导致问题

W 字符代表着平日\(Mon-Fri\)，并且仅能用于日域中。它用来指定离指定日的最近的一个平日。大部分的商业处理都是基于工作周的，所以 W 字符可能是非常重要的。

> 例如，日域中的 15W 意味着 “离该月15号的最近一个平日。” 假如15号是星期六，那么 trigger 会在14号\(星期五\)触发，因为星期四比星期一离15号更近。

C：代表“Calendar”的意思。它的意思是计划所关联的日期，如果日期没有被关联，则相当于日历中所有日期。

> 例如5C在日期字段中就相当于日历5日以后的第一天。1C在星期字段中相当于星期日后的第一天。

| 字段 | 允许值 | 允许的特殊字符 |
| :--- | :--- | :--- |
| 秒 | 0~59 | , - \* / |
| 分 | 0~59 | , - \* / |
| 小时 | 0~23 | , - \* / |
| 日期 | 1-31 | , - \* ? / L W C |
| 月份 | 1~12或者JAN~DEC | , - \* / |
| 星期 | 1~7或者SUN~SAT | , - \* ? / L C \# |
| 年（可选） | 留空，1970~2099 | , - \* / |

在线cron表达式生成：[http://qqe2.com/cron/index](http://qqe2.com/cron/index)

## 整合Quartz <a id="SpringBoot&#x51E0;&#x79CD;&#x5B9A;&#x65F6;&#x4EFB;&#x52A1;&#x7684;&#x5B9E;&#x73B0;&#x65B9;&#x5F0F;-&#x6574;&#x5408;Quartz"></a>

* 添加依赖

如果SpringBoot版本是2.0.0以后的，则在spring-boot-starter中已经包含了quart的依赖，则可以直接使用`spring-boot-starter-quartz`依赖：

```markup
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

如果是1.5.9则要使用以下添加依赖：

```java
<dependency>
  <groupId>org.quartz-scheduler</groupId>
  <artifactId>quartz</artifactId>
  <version>2.3.0</version>
</dependency>
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-context-support</artifactId>
</dependency>
```

这里我使用SpringBoot版本是`2.0.0.BUILD-SNAPSHOT` ，该版本开始集成了Quartz，所以事实现起来很方便。其它好像比较麻烦，这里就不介绍，以后有时间再详细深入了解Quartz。

创建任务类TestQuartz，该类主要是继承了QuartzJobBean

```java

public class TestQuartz extends QuartzJobBean {
    /**
     * 执行定时任务
     * @param jobExecutionContext
     * @throws JobExecutionException
     */
    @Override
    protected void executeInternal(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        System.out.println("quartz task "+new Date());
    }
}
```

创建配置类`QuartzConfig`

```java
package com.boss.test;
 
 
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.springframework.scheduling.quartz.QuartzJobBean;
 
import java.util.Date;
 
 
/**
 * @author wangwenxiang
 * 2018/12/1.
 * @description:
 */
public class TestQuartz extends QuartzJobBean {
    /**
     * 执行定时任务
     * @param jobExecutionContext
     * @throws JobExecutionException
     */
    @Override
    protected void executeInternal(JobExecutionContext jobExecutionContext) throws JobExecutionException {
        System.out.println("quartz task "+new Date());
    }
}
```

##  最后 <a id="SpringBoot&#x51E0;&#x79CD;&#x5B9A;&#x65F6;&#x4EFB;&#x52A1;&#x7684;&#x5B9E;&#x73B0;&#x65B9;&#x5F0F;-&#x6700;&#x540E;"></a>

上面都是简单的介绍了关于SpringBoot定时任务的处理，直接使用SpringTask注解的方式应该是最方便的，而使用Quartz从2.0开始也变得很方便。对于这两种方式，应该说各有长处吧，按需选择。

另外关于Quartz的详细内容可以查看官方文档：[http://www.quartz-scheduler.org/documentation/](http://www.quartz-scheduler.org/documentation/)


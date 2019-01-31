---
description: >-
  有时候当你想批量调取第三方api获取数据，并将返回的信息聚集起来，如果我们用直接简单的循环调用，这个处理时间长不说，一个调用超时就会阻塞整个逻辑的处理。如果采用异步的方式调用，编程的成本就会增大，比如最终结果的收集，异常的处理等；在java8中，有提供更加简便的编程方式实现这种异步处理。
---

# Java8 CompletableFuture: 组合式异步编程

首先，设置一个简单的场景吧：

比如，我们现在有11个经纬度地址，需要获取他们对应的中文名称，下面是我的实验代码：

```java
// 调用第三方api获取地址名称
 private String getAddress(String latitude, String longitude) {
     StopWatch sw = new StopWatch();
     sw.start();
     JSONObject result = restTemplate.getForObject("http://api.map.baidu.com/geocoder/v2/?location={1},{2}&output=json&pois=0&ak=************************",
             JSONObject.class, latitude, longitude);
     sw.stop();
     // 记录执行线程以及耗时时间
     System.out.println(String.format("getAddress: %s -> 耗时: %s", Thread.currentThread(), sw.getTotalTimeMillis()));
     if (result.getInteger("status") == 0) {
         return result.getJSONObject("result").getString("formatted_address");
     }
     return null;
 }
```

```java
// 测试方法
@Test
public void test() {
    String[] cities = {
            "31.2363429624,121.4803295328", "39.9110666857,116.4136103013",
            "30.2799186759,120.1617445782", "31.8880209204,117.3066537271",
            "25.6122215609,100.2742019952", "26.8624428068,100.2335674911",
            "27.6958640000,111.7206640000", "31.2093160000,112.4105620000",
            "39.1731490000,117.2202970000", "34.5113900000,101.5563070000",
            "30.5984670000,114.3115860000"
    };
 
    List<Long> times = new ArrayList<>();
 
    // 执行20次，计算调用11个经纬度耗时
    IntStream.rangeClosed(1, 20).forEach(i -> {
        StopWatch sw = new StopWatch();
        sw.start();
        // 调用实现的方法，更改不同的方法，统计时间
        List<String> r = test1(cities);
        System.out.println(r);
        sw.stop();
        times.add(sw.getTotalTimeMillis());
    });
    System.out.println(times);
    LongSummaryStatistics summaryStatistics = times.stream().mapToLong(Long::longValue).summaryStatistics();
    System.out.println("平均耗时: " + summaryStatistics);
}
```

```java
// 方法一 普通循环调用
private List<String> test1(String[] locations) {
    return Arrays.stream(locations)
            .filter(city -> city.split(",").length == 2)
            .map(city -> {
                String[] v = city.split(",");
                return getAddress(v[0], v[1]);
            })
            .collect(Collectors.toList());
}
 
// java8 Stream parallel并行调用
private List<String> test2(String[] locations) {
    return Arrays.stream(locations).parallel()
            .filter(city -> city.split(",").length == 2)
            .map(city -> {
                String[] v = city.split(",");
                return getAddress(v[0], v[1]);
            })
            .collect(Collectors.toList());
}
 
// java8 CompletableFuture
private List<String> test3(String[] locations) {
    List<CompletableFuture<String>> collect = Arrays.stream(locations)
            .filter(city -> city.split(",").length == 2)
            .map(city -> CompletableFuture.supplyAsync(() -> {
                String[] v = city.split(",");
                return getAddress(v[0], v[1]);
            }))
            .collect(Collectors.toList());
 
    return collect.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
 
}
```

这里，我简单的每个方法调用20次，统计一下时间，进行一下对比：

**方法一**

执行`test1()`时，控制台输出如下：

```text
[662, 564, 645, 552, 577, 562, 534, 1101, 549, 567, 546, 544, 545, 548, 557, 551, 609, 527, 562, 531]
[null, 北京市东城区正义路2号-10号楼-403号, 浙江省杭州市拱墅区莫干山路6, 安徽省合肥市瑶海区众兴路, 云南省大理白族自治州大理市第一大道, 云南省丽江市古城区玉雪大道149, 湖南省娄底市涟源市人民东路, 湖北省荆门市钟祥市, null, null, 湖北省武汉市江岸区沿江大道188号]
[396]
平均耗时: LongSummaryStatistics{count=20, sum=11833, min=527, average=591.650000, max=1101}
```

11个经纬度，平均耗时591.650000毫秒。

**方法二**

执行`test2()`时，控制台输出如下

```bash
...
getAddress: Thread[ForkJoinPool.commonPool-worker-3,5,main] -> 耗时: 32
getAddress: Thread[ForkJoinPool.commonPool-worker-1,5,main] -> 耗时: 29
getAddress: Thread[ForkJoinPool.commonPool-worker-3,5,main] -> 耗时: 25
getAddress: Thread[ForkJoinPool.commonPool-worker-1,5,main] -> 耗时: 25
getAddress: Thread[ForkJoinPool.commonPool-worker-3,5,main] -> 耗时: 25
getAddress: Thread[ForkJoinPool.commonPool-worker-2,5,main] -> 耗时: 233
[277, 252, 143, 116, 160, 164, 119, 125, 125, 121, 125, 127, 121, 132, 128, 128, 118, 124, 113, 236]
[null, 北京市东城区正义路2号-10号楼-403号, 浙江省杭州市拱墅区莫干山路6, 安徽省合肥市瑶海区众兴路, 云南省大理白族自治州大理市第一大道, 云南省丽江市古城区玉雪大道149, 湖南省娄底市涟源市人民东路, 湖北省荆门市钟祥市, null, null, 湖北省武汉市江岸区沿江大道188号]
[396]
平均耗时: LongSummaryStatistics{count=20, sum=2954, min=113, average=147.700000, max=277}
```

可见，多个线程在执行调用，平均耗时147.700000毫秒。

**方法三**

执行`test3()`时，控制台输出如下：

```text
...
getAddress: Thread[ForkJoinPool.commonPool-worker-1,5,main] -> 耗时: 157
getAddress: Thread[ForkJoinPool.commonPool-worker-2,5,main] -> 耗时: 46
getAddress: Thread[main,5,main] -> 耗时: 53
getAddress: Thread[ForkJoinPool.commonPool-worker-3,5,main] -> 耗时: 51
getAddress: Thread[ForkJoinPool.commonPool-worker-1,5,main] -> 耗时: 51
getAddress: Thread[ForkJoinPool.commonPool-worker-2,5,main] -> 耗时: 56
[280, 157, 164, 166, 209, 194, 164, 263, 233, 203, 158, 154, 162, 152, 167, 154, 149, 149, 155, 159]
[null, 北京市东城区正义路2号-10号楼-403号, 浙江省杭州市拱墅区莫干山路6, 安徽省合肥市瑶海区众兴路, 云南省大理白族自治州大理市第一大道, 云南省丽江市古城区玉雪大道149, 湖南省娄底市涟源市人民东路, 湖北省荆门市钟祥市, null, null, 湖北省武汉市江岸区沿江大道188号]
[396]
平均耗时: LongSummaryStatistics{count=20, sum=3592, min=149, average=179.600000, max=280}
```

同样的多个线程在异步执行，平均耗时179.600000毫秒；

其实从效果上还看\(这里运行次数较少，运行时有时候方法二好有时候方法三好\)，方法二和方法三它们看起来不相伯仲，究其原因都一样:它们内部采用的是同样的通用线程池，默认都使用固定数目的线程，具体线程数取决于`Runtime.getRuntime().availableProcessors()`的返回值。尽然这样，方法二更简洁易懂，为什么还需要`CompletableFuture`呢？然而，`CompletableFuture`具有一定的优势，因为它允许你对执行器\(Executor\)进行配置，尤其是线程池的大小，让它以更适合应用需求的方式进行配置，满足程序的要求，而这是并行流API无法提供的。

`CompletableFuture`中4个异步执行任务静态方法：

```java
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
        return asyncSupplyStage(asyncPool, supplier);
    }
 
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,Executor executor) {
    return asyncSupplyStage(screenExecutor(executor), supplier);
}
 
public static CompletableFuture<Void> runAsync(Runnable runnable) {
    return asyncRunStage(asyncPool, runnable);
}
 
public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor) {
    return asyncRunStage(screenExecutor(executor), runnable);
}
```


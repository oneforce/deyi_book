# 搭建spring cloud gateway

#### spring cloud gateway 运行机制 <a id="id-&#x642D;&#x5EFA;springcloudgateway-springcloudgateway&#x8FD0;&#x884C;&#x673A;&#x5236;"></a>

此处借用官方的一张图片：

![](.gitbook/assets/20190131_09.png)

* 服务调用者请求spring cloud gateway
* 根据路由策略找到走哪个路由
* 执行此路由配置的Filter链
* 找到具体被代理的请求地址，然后转发过去
* 返回具体代理请求的响应信息
* 返回调用者

#### 构建我们自己的spring cloud gateway项目 <a id="id-&#x642D;&#x5EFA;springcloudgateway-&#x6784;&#x5EFA;&#x6211;&#x4EEC;&#x81EA;&#x5DF1;&#x7684;springcloudgateway&#x9879;&#x76EE;"></a>

**第一步引入jar包**

```java
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <jdk.version>1.8</jdk.version>
    <lombok.version>1.16.8</lombok.version>
</properties>
 
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.4.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
 
<dependencyManagement>
    <dependencies>
 
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Finchley.SR1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
 
    </dependencies>
</dependencyManagement>
<dependencies>
 
    <dependency>
        <groupId> org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
 
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>
 
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
    </dependency>
 
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-sleuth</artifactId>
    </dependency>
 
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>${lombok.version}</version>
    </dependency>
 
    <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
        <version>3.4.12</version>
    </dependency>
</dependencies>
```

**第二部构建spring boot启动类**

spring cloud gateway是建立在spring boot项目的基础上的，本例还引入了sleuth，使用的zookeeper作为注册中心。

需要再spring boot启动类上加上@EnableDiscoveryClient ，把我们的服务注册到注册中心上。**GatewayBootstrap** 折叠源码

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
 
/**
 * @author jim lin
 * 2018/9/17.
 */
@SpringBootApplication
@EnableDiscoveryClient
public class GatewayBootstrap {
 
    public static void main(String[] args) {
        SpringApplication.run(GatewayBootstrap.class,args);
    }
}
```

**第三步，配置代理策略**

通过以上的代码我们的spring cloud gateway就已经搭建完成了，剩下的都在配置文件中配置。

application.yml文件如下：

```yaml
spring:
  application:
    name: boss-cas-gateway
  cloud:
    gateway:
      routes:
        - id: boss-risk-ap
          uri: lb://boss-risk-ap
          predicates:
          - Path=/boss-risk-ap/**
          filters:
          - StripPrefix=1
          #定义router的唯一ID
        - id: boss-finance-product
          #定义最终路由的地址，lb表示从loadbalance里面获取，此处我们从zookeeper注册中心中获取，获取的instance-id为boss-finance-product
          uri: lb://boss-finance-product
          #定义gateway拦截哪些请求，此处会拦截所有前缀为/api/boss-finance-product
          predicates:
          - Path=/api/boss-finance-product/**
          #此处定义gateway的filter机制
          filters:
          #自定义自己的filter，此filter只对当前的路由生效，即只对router-id为boss-finance-product
          #Tip，在spring cloudgateway中，TokenValidation对应的类为TokenValidation+GatewayFilterFactory的java类。
          #此属性表示对上述拦截的请求截取两段，//中间表示一段，本例即截取/api/boss-finance-product/
          - StripPrefix=2
      discovery:
        locator:
          enabled: true
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins: "*"
            allowedMethods: "*"
            allowedHeaders: "*"
            allowCredentials: true
    zookeeper:
#      connect-string: base.intersh.victorplus.cn:2181
      connect-string: 172.16.45.42:2181
      discovery:
        instance-host: 192.168.60.102
server:
  port: 9999
  tomcat:
    accesslog:
      enabled: true
      rename-on-rotate: true
      pattern: "%{yyyy-MM-dd HH:mm:ss}t %h %{X-FORWARDED-FOR}i %l %u %m %U %H %s %b %q '%{Referer}i' '%{User-Agent}i' %I %T"
      directory: ${log_dir}
 
logging:
  level:
    org.springframework.cloud.gateway: trace
    org.springframework.http.server.reactive: debug
    org.springframework.web.reactive: debug
    reactor.ipc.netty: debug
management:
  endpoints:
    web:
      base-path: /
      exposure:
        include: "*"
        exclude: ["heapdump"]
```

**第四步，自定义全局Filter和私有Filter**

使用spring boot Configuration来注入我们自定义filter**Configuration** 折叠源码

```java
import cn.victorplus.car.gateway.filter.TokenValidationGatewayFilterFactory;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import reactor.core.publisher.Mono;
 
/**
 * @author jim lin
 * 2018/9/17.
 */
@Configuration
@Slf4j
public class GatewayConfiguration {
 
    public static String V_KEY ="vkey";
 
    @Bean
    public GlobalFilter globalFilter(){
        return (exchange,chain)->{
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                log.debug("==================== filter");
            }));
        };
 
    }
 
    @Bean
    public TokenValidationGatewayFilterFactory tokenValidationGatewayFilterFactory(){
        return new TokenValidationGatewayFilterFactory();
    }
}
```

自定义Filter，TokenValidationGatewayFilterFactory，用来解析所有从前端过来的请求的token校验。**TokenValidationGatewayFilterFactory** 

```java
import cn.victorplus.car.gateway.util.SecurityUtil;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang.StringUtils;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import org.springframework.core.io.buffer.DataBuffer;
import org.springframework.http.HttpCookie;
import org.springframework.http.HttpStatus;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.util.CollectionUtils;
import org.springframework.util.MultiValueMap;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
 
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
 
/**
 * @author jim lin
 * 2018/9/18.
 */
@Slf4j
public class TokenValidationGatewayFilterFactory extends AbstractGatewayFilterFactory<TokenValidationGatewayFilterFactory.Config> {
 
    private static String V_KEY ="vkey";
    private static String USER_ID ="x-victor-plus-userId";
    public static final String VALID_KEY = "valid";
 
    public TokenValidationGatewayFilterFactory() {
        super(Config.class);
    }
 
    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList(VALID_KEY);
    }
 
    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            if (!config.isValid()){
                return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                }));
            }
            ServerHttpRequest httpRequest = exchange.getRequest();
            MultiValueMap<String, HttpCookie> cookies = httpRequest.getCookies();
            if (!cookies.containsKey(V_KEY)){
                return failRequest(exchange);
            }
            List<HttpCookie> httpCookie = cookies.get(V_KEY);
            if (CollectionUtils.isEmpty(httpCookie) || httpCookie.size() > 1){
                return failRequest(exchange);
            }
            String token = httpCookie.get(0).getValue();
            if (StringUtils.isBlank(token)){
                return failRequest(exchange);
            }
            try {
                String[] result = SecurityUtil.getDecVal(token, 4);
                if (result == null || result.length < 1){
                    return failRequest(exchange);
                }
                ServerHttpRequest request = exchange.getRequest().mutate()
                        .header(USER_ID,result[0])
                        .build();
                return chain.filter(exchange.mutate().request(request).build()).then(Mono.fromRunnable(() -> {
                    log.warn("==================== filter");
                }));
            } catch (Exception e) {
                log.warn("token校验发生错误，错误信息为：",e);
                return failRequest(exchange);
            }
        };
    }
 
    private Mono<Void> failRequest(ServerWebExchange exchange){
        exchange.getResponse().setStatusCode(HttpStatus.FORBIDDEN);
        DataBuffer buffer = exchange.getResponse().bufferFactory().wrap("".getBytes());
        return exchange.getResponse().writeWith(Flux.just(buffer));
    }
 
    public static class Config {
        private boolean valid = true;
 
        public boolean isValid() {
            return valid;
        }
 
        public void setValid(boolean valid) {
            this.valid = valid;
        }
    }
 
}
```

spring cloud gateway为我们提供了多样的路由策略，多样的Filter机制，使我们可以灵活的实现业务所需。

参考文档 ： spring cloud gateway官方文档 [https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.0.2.RELEASE/single/spring-cloud-gateway.html](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.0.2.RELEASE/single/spring-cloud-gateway.html)


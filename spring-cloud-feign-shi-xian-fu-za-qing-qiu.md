# spring cloud feign 实现复杂请求

### 1 请求头 <a id="springcloudfeign&#x5B9E;&#x73B0;&#x590D;&#x6742;&#x8BF7;&#x6C42;-1&#x8BF7;&#x6C42;&#x5934;"></a>

Content-Type:application/json;charset=utf-8;  
Authorization: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX;

* Authorization是包头验证信息 Authorization的值为 Base64编码\(账户Id +冒号+时间戳\) 其中账户Id可以去[对接数据查询](http://developer.7moor.com/data-query/)里查询。 七陌会提供两个账号，一个是8xxx@xxx，另一个是xxx。对接数据查询用全是字母的账号查询。 例如: N00000000556:20161013113612 base64编码后: TjAwMDAwMDAwNTU2OjIwMTYxMDEzMTEzNjEy
* 冒号为英文冒号
* 时间戳是当前系统时间,格式"yyyyMMddHHmmss",需与sig参数中时间戳相同。

### 2 请求参数（sig） <a id="springcloudfeign&#x5B9E;&#x73B0;&#x590D;&#x6742;&#x8BF7;&#x6C42;-2&#x8BF7;&#x6C42;&#x53C2;&#x6570;&#xFF08;sig&#xFF09;"></a>

例如：

```text
http://apis.7moor.com/v20160818/customer/getTemplate/N00000000556?sig=3E92F146297FCA751F63493877EC9719
```

* URL后必须带有sig参数,例如sig=AAABBBCCCDDDEEEFFFGGG。
* sig的值为 32位大写MD5加密 （帐号Id + 帐号APISecret +时间戳）

例如 N00000000556secret20161013113612

md5加密后为 88996D9907E0EE52C5DAF8EFFCC31CFC

* 时间戳是当前系统时间,格式"yyyyMMddHHmmss"。时间戳是24小时制,如：20140416142030,访问接口时5分钟之内的时间戳有效
* APISecret 可以去[对接数据查询](http://developer.7moor.com/data-query/)里查询

### 3 请求体 <a id="springcloudfeign&#x5B9E;&#x73B0;&#x590D;&#x6742;&#x8BF7;&#x6C42;-3&#x8BF7;&#x6C42;&#x4F53;"></a>

请求体数据类型是JSON,如果接口不需要,可以不传。

例如：{"\_id":"22e25d60-809d-11e6-ad5a-b7e3030127fb"}

### 实现方法 <a id="springcloudfeign&#x5B9E;&#x73B0;&#x590D;&#x6742;&#x8BF7;&#x6C42;-&#x5B9E;&#x73B0;&#x65B9;&#x6CD5;"></a>

使用普通的feign接入不能满足上面的要求，下面介绍使用feign实现上述复杂场景。

**第一步，定义一个feign接口**

**RongLianClient**

```java
import cn.victorplus.base.recording.bean.dto.CallRecordRespDTO;
import cn.victorplus.base.recording.bean.dto.CallRecordReqDTO;
import cn.victorplus.base.recording.bean.dto.RongLianRespDTO;
import cn.victorplus.base.recording.config.RongLianFeignConfiguration;
import feign.Headers;
import feign.Param;
import feign.RequestLine;
import org.springframework.cloud.openfeign.FeignClient;
 
import java.util.List;
 
/**
 * @author jim lin
 * 2018/10/23.
 */
@FeignClient(name = "ronglian",url = "${rong.lian.requestUrl}",configuration = RongLianFeignConfiguration.class)
public interface RongLianClient {
 
//    @PostMapping(produces = MediaType.APPLICATION_JSON_VALUE)
 
    //feign的注解，实现请求URL的可变，在Configuration中开启feign的contract默认模式，才能使用此注解，否则会报错。
    @RequestLine("POST /v20170704/cdr/getCCCdr/{accountId}?sig={sig}")
    //注解head，定义自定义head的变量
    @Headers({
            "Authorization: {Authorization}",
            "Accept: application/json, text/plain, */*"
    })
    //通过feign自带的注解@param来传入变量
    RongLianRespDTO<List<CallRecordRespDTO>> pullRongLianRecordInfo(@Param("accountId") String accountId, @Param("sig") String sig, @Param("Authorization") String authorization, CallRecordReqDTO callRecordReqDTO);
 
}
```

**第二步，单独在此Client调用类中开启Feign默认的Contract模式**

**RongLianFeignConfiguration**

```java
import feign.Contract;
import org.springframework.context.annotation.Bean;
 
/**
 * @author jim lin
 * 2018/9/11.
 */
//@Configuration
//@EnableConfigurationProperties(RongLianSettings.class)
public class RongLianFeignConfiguration {
 
//    @Autowired
//    private RongLianSettings rongLianSettings;
 
    //开启默认的Feign的Contract
    @Bean
    public Contract feignContract() {
        return new feign.Contract.Default();
    }
    //被注解掉的代码是使用Interceptor来实现全局Head生成，本用例因为sig和Authorization 都需要时间这个参数，所以不能使用全局配置
//    @Bean
//    public MyRequestInterceptor requestInterceptor (){
//        return new MyRequestInterceptor();
//    }
 
//    class MyRequestInterceptor implements RequestInterceptor {
//
//        @Override
//        public void apply(RequestTemplate requestTemplate) {
//            String encryptOriginString = rongLianSettings.getAccountId() + ":20181024094130"; //+ new SimpleDateFormat("yyyyMMddHHssmm").format(new Date());
//            String auth = Base64.encodeBase64String(encryptOriginString.getBytes());
//            requestTemplate.header("Authorization",auth);
//        }
//    }
}
```

**第三步：在调用的方法中生成调用的变量值，并请求**

```java
public RespDTO<String> getCallRecord(PullRongLianRecordParam recordParam){
    String time = LocalDateUtil.getCurrentTime();
    String sig = rongLianUtil.buildSig(time);
    String authorization = rongLianUtil.buildAuthorization(time);
    CallRecordReqDTO callRecordReqDTO = new CallRecordReqDTO(LocalDateUtil.defaultParse(recordParam.getBeginTime()),LocalDateUtil.defaultParse(recordParam.getEndTime()));
    RongLianRespDTO<List<CallRecordRespDTO>> respDTO = rongLianClient.pullRongLianRecordInfo(rongLianSettings.getAccountId(),sig,authorization,callRecordReqDTO);
    log.debug("获取电话记录请求参数:{}，响应结果为：{}",recordParam,respDTO);
    if (!respDTO.isSuccess()){
        return RespDTO.fail(respDTO.getMessage());
    }
    respDTO.getData().forEach(record -> rongLianManager.saveRongLianInfo(record));
    return RespDTO.success(StringEnum.SUCCESS.getValue());
}
```

对于一些特殊复杂的场景，使用spring的注解@PostMapping等不能实现业务需求的时候，可以使用Feign自带的注解。


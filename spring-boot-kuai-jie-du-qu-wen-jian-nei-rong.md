# Spring Boot快捷读取文件内容



### 一、示例 <a id="SpringBoot&#x5FEB;&#x6377;&#x8BFB;&#x53D6;&#x6587;&#x4EF6;&#x5185;&#x5BB9;-&#x4E00;&#x3001;&#x793A;&#x4F8B;"></a>

1. yml配置

```yaml
test:
  config:
    a: classpath:release/20181018_v1.1.0.md
    b: file:C:/Users/yc/Desktop/pytest.jks
    c: url:http://www.test.com/test.txt
```

|  |
| :--- |


1. 配置类

```java
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Component;
 
@Component
@ConfigurationProperties(prefix = "test.config")
@Data
public class ConfigTest {
 
    private Resource a;
 
    private Resource b;
 
    private Resource c;
 
}
```

|  |
| :--- |


### 二、原理（基于spring-boot-2.0.6.RELEASE版本） <a id="SpringBoot&#x5FEB;&#x6377;&#x8BFB;&#x53D6;&#x6587;&#x4EF6;&#x5185;&#x5BB9;-&#x4E8C;&#x3001;&#x539F;&#x7406;&#xFF08;&#x57FA;&#x4E8E;spring-boot-2.0.6.RELEASE&#x7248;&#x672C;&#xFF09;"></a>

      当类上使用@ConfigurationProperties 和 `@Component` ，当其他类注入该类时,就会触发该类的加载过程。在加载过程中,会调用`AbstractAutowireCapableBeanFactory`类的`applyBeanPostProcessorsBeforeInitialization`方法。从而触发`ConfigurationPropertiesBindingPostProcessor`类`postProcessBeforeInitialization`方法的调用

* postProcessBeforeInitialization方法代码如下：

```java

@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
   //获得类上的@ConfigurationProperties注解
   ConfigurationProperties annotation = getAnnotation(bean, beanName, ConfigurationProperties.class);
   if (annotation != null) {
      //建立绑定关系
      bind(bean, beanName, annotation);
   }
   return bean;
}
```

|  |
| :--- |


* 跟踪代码会发现有配置文件属性值与配置类的属性类型进行转换的过程，会调用GenericConversionService类的convert方法。convert方法代码如下：

```java
/**
 * 进行配置文件属性值与配置类的属性转换
 *
 * @param source 配置文件属性值，例如上文的“classpath:release/20181018_v1.1.0.md”
 * @param sourceType 配置文件属性类型
 * @param targetType 配置类属性的类型，例如上文的“org.springframework.core.io.Resource”
 */
 
@Override
@Nullable
public Object convert(@Nullable Object source, @Nullable TypeDescriptor sourceType, TypeDescriptor targetType) {
   Assert.notNull(targetType, "Target type to convert to cannot be null");
   if (sourceType == null) {
      Assert.isTrue(source == null, "Source must be [null] if source type == [null]");
      return handleResult(null, targetType, convertNullSource(null, targetType));
   }
   if (source != null && !sourceType.getObjectType().isInstance(source)) {
      throw new IllegalArgumentException("Source to convert from must be an instance of [" +
            sourceType + "]; instead it was a [" + source.getClass().getName() + "]");
   }
   //获取配置文件属性类型与配置类属性的类型的转换器
   GenericConverter converter = getConverter(sourceType, targetType);
   if (converter != null) {
      Object result = ConversionUtils.invokeConverter(converter, source, sourceType, targetType);
      return handleResult(sourceType, targetType, result);
   }
   return handleConverterNotFound(source, sourceType, targetType);
}
```

* 继续debug会发现最终将调用DefaultResourceLoader的getResource方法对配置文件属性值进行解析。DefaultResourceLoader类的getResource方法 代码如下:

```java
@Override
public Resource getResource(String location) {
    Assert.notNull(location, "Location must not be null");
 
    for (ProtocolResolver protocolResolver : this.protocolResolvers) {
        Resource resource = protocolResolver.resolve(location, this);
        if (resource != null) {
            return resource;
        }
    }
 
    if (location.startsWith("/")) {
        return getResourceByPath(location);
    }
    //1、如果location以“classpath:”开头，即转换成ClassPathResource
    else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
        return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
    }
    else {
        try {
            // 2、否则尝试解析成URL，如配置中的“url:http://www.test.com/test.txt”
            URL url = new URL(location);
            return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
        }
        catch (MalformedURLException ex) {
            // 3、不能解析成URL,则以资源路径的方式来转义，如配置中的“file:C:/Users/yc/Desktop/pytest.jks”
            return getResourceByPath(location);
        }
    }
}
```




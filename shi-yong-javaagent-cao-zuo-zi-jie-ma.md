# 使用javaagent操作字节码



### 一、javaagent主要作用 <a id="id-&#x4F7F;&#x7528;javaagent&#x64CD;&#x4F5C;&#x5B57;&#x8282;&#x7801;-&#x4E00;&#x3001;javaagent&#x4E3B;&#x8981;&#x4F5C;&#x7528;"></a>

* 可以在加载java文件之前做拦截把字节码做修改
* 可以在运行期将已经加载的类的字节码做变更
* 获取所有已经被加载过的类
* 获取所有已经被初始化过了的类（执行过了clinit方法，是上面的一个子集）
* 获取某个对象的大小
* 将某个jar加入到bootstrapclasspath里作为高优先级被bootstrapClassloader加载
* 将某个jar加入到classpath里供AppClassload去加载
* 设置某些native方法的前缀，主要在查找native方法的时候做规则匹配

### 二、javaagent简单实例 <a id="id-&#x4F7F;&#x7528;javaagent&#x64CD;&#x4F5C;&#x5B57;&#x8282;&#x7801;-&#x4E8C;&#x3001;javaagent&#x7B80;&#x5355;&#x5B9E;&#x4F8B;"></a>

####       2.1、创建Javaagent工程 <a id="id-&#x4F7F;&#x7528;javaagent&#x64CD;&#x4F5C;&#x5B57;&#x8282;&#x7801;-2.1&#x3001;&#x521B;&#x5EFA;Javaagent&#x5DE5;&#x7A0B;"></a>

* 创建一个类，实现premain方法 折叠源码

```java
public class AgentPreMain {
    //待拦截的类名
    private static final String injectedClassName = "org.test.Target";
 
    //待拦截的方法名
    private static final String injectedMethodName = "doSomething";
 
    public static void premain(String agentArgs, Instrumentation inst){
        System.out.println("start agent premain...");
        inst.addTransformer(((loader, className, classBeingRedefined, protectionDomain, classfileBuffer) -> {
            className = className.replace("/",".");
            if (className.equals(injectedClassName)) {
                try {
                    CtClass ctClass = ClassPool.getDefault().get(className);
                    CtMethod ctMethod = ctClass.getDeclaredMethod(injectedMethodName);
                    ctMethod.insertBefore("System.out.println(\"what a beautiful day...\");");
                    return ctClass.toBytecode();
                }catch (Exception e){
                    e.printStackTrace();
                }
            }
            return null;
        }));
    }
}
```

* 打包成jar包 折叠源码

```markup
<dependencies>
        <!-- https://mvnrepository.com/artifact/org.javassist/javassist -->
        <dependency>
            <groupId>org.javassist</groupId>
            <artifactId>javassist</artifactId>
            <version>3.23.1-GA</version>
        </dependency>
 
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>utf-8</encoding>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.0.0</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <transformers>
                                <transformer
                                        implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <manifestEntries>
                                        <Premain-Class>org.agent.AgentPreMain</Premain-Class>
                                    </manifestEntries>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</dependencies>
```

 Javassit其实就是一个三方包，提供了运行时操作Java字节码的方法。Java代码编译完会生成.class文件，就是一堆字节码。JVM\(准确说是JIT\)会解释执行这些字节码\(转换为机器码并执行\)，由于字节码的解释执行是在运行时进行的，Javassist就提供了一些方便的方法，让我们通过这些方法生成字节码。

           因为需要自动的在MANIFEST.MF文件中添加Premain-Class，所以这里需要在pom中添加插件并且指定premain-class。

#### 2.2创建目标项目 <a id="id-&#x4F7F;&#x7528;javaagent&#x64CD;&#x4F5C;&#x5B57;&#x8282;&#x7801;-2.2&#x521B;&#x5EFA;&#x76EE;&#x6807;&#x9879;&#x76EE;"></a>

1. 建一个类包含实现main方法 折叠源码

```java
public class Target {
    public static void main(String[] args) {
        doSomething();
    }
 
    public static void doSomething(){
        System.out.println("i am going to traveling...");
    }
}
```

* 打包所用maven配置 

```markup
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
                <encoding>utf-8</encoding>
            </configuration>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>3.0.0</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                    <configuration>
                        <transformers>
                            <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <!--指定执行的main函数-->                   
                                    <mainClass>org.test.Target</mainClass>
                            </transformer>
                        </transformers>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### 三、测试 <a id="id-&#x4F7F;&#x7528;javaagent&#x64CD;&#x4F5C;&#x5B57;&#x8282;&#x7801;-&#x4E09;&#x3001;&#x6D4B;&#x8BD5;"></a>

Javaagent工程打包成jar包，javaagent-1.0-SNAPSHOT.jar，目标项目打成jar包：target-1.0-SNAPSHOT.jar

       运行命令： 

```text
java -javaagent:javaagent-1.0-SNAPSHOT.jar -jar test-1.0-SNAPSHOT.jar
start agent premain...
what a beautiful day...
i am going to traveling...
```


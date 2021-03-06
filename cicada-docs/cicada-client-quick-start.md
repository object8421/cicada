# cicada-client使用指南

## 1、说明

          cicada-client有两种classifier：

          一、aspectj：对应静态AOP，采用字节码加强技术，编译时将切面代码直接编译到Java类文件中

          二、spring-aop：对应动态AOP，在运行时将切面代码进行动态织入实现的AOP

        研发人员可以根据需要选择适合自己的classifier。 

## 2、项目中引入cicada-client的步骤

 下面分别介绍 spring-boot/普通Spring引入aspectj/spring-aop的流程，研发人员可以根据自己的情况从中任选一种实现方式 
### 2.1  Spring Boot项目引入classifier:aspectj

#### 2.1.1、引入依赖

```
<!-- 编译切面时需要 -->
<dependency>
             <groupId>org.aspectj</groupId>
             <artifactId>aspectjweaver</artifactId>
</dependency>
<dependency>
             <groupId>com.yirendai.infra</groupId>
             <artifactId>cicada-client</artifactId>
             <classifier>aspectj</classifier>
</dependency>
```


#### 2.1.2、引入aspectj maven插件：用于将切面代码编译到目标class中

```
<plugin>
                 <groupId>org.codehaus.mojo</groupId>
                 <artifactId>aspectj-maven-plugin</artifactId>
                 <version> 1.8 </version>
                 <executions>
                     <execution>
                         <goals>
                             <goal>compile</goal>
                             <goal>test-compile</goal>
                         </goals>
                         <configuration>
                             <complianceLevel> 1.7 </complianceLevel>
                             <source> 1.7 </source>
                             <target> 1.7 </target>
                             <aspectLibraries>
                                 <aspectLibrary>
                                     <groupId>com.yirendai.infra</groupId>
                                     <artifactId>cicada-client</artifactId>
                                     <classifier>aspectj</classifier>
                                 </aspectLibrary>
                             </aspectLibraries>
                         </configuration>
                     </execution>
                 </executions>
</plugin>
```
#### 2.1.3、配置Cicada，例如在application.yml中加上如下所示的配置片段
```
 cicada:
    url: http: //localhost:9080/upload
    sampleRate: 100
    connectTimeout: 100
    soTimeout: 100
    batchSize: 32
    bufferSize: 1024
    tpsLimit: 2048
```

同时在java启动类中加上如下所示注解，启用读取配置的java文件
```
@SpringBootApplication(scanBasePackages={"com.yirendai.infra.cicada"})
//在web工程中需要加此注解
@ServletComponentScan(basePackages={"com.yirendai.infra.cicada"})  
```


### 2.2 Spring Boot项目引入 classifier: spring-aop

#### 2.2.1、引入依赖

```
<!-- 或其他支持spring aop的starter -->  
  <dependency>
             <groupId>org.springframework.boot</groupId>
             <artifactId>spring-boot-starter-aop</artifactId>
</dependency> 
<dependency>
             <groupId>com.yirendai.infra</groupId>
             <artifactId>cicada-client</artifactId>
             <classifier>spring-aop</classifier>
</dependency>

```

#### 2.2.1、加入cicada配置，同上

### 2.3 普通Spring项目引入 classifier:aspectj  或  classifier: spring-aop

      主要步骤同Spring Boot的引入流程，区别在于可以用xml配置bean，如下所示

```
 <bean id= "transferEngine" class = "com.yirendai.infra.cicada.transfer.TransferEngine"
          init-method= "start" >
          <!-- 用于收集日志的nginx地址，必填 -->
          <constructor-arg value= " />  
           
          <!-- 以下为选填，根据需要填写 -->
          <!-- 采样率，整数 ，取值空间： 1 - 100 -->
          <property name= "sampleRate" value= "100" />
          <!-- 连接超时时长，整数，毫秒 -->
          <property name= "connectTimeout" value= "100" />
          <!-- 传送超时时长，整数，毫秒 -->
          <property name= "soTimeout" value= "100" />
          <!-- 批处理大小,消息条数，整数 -->
          <property name= "batchSize" value= "32" />
          <!-- 缓存消息条数，整数 -->
          <property name= "bufferSize" value= "1024" />
          <!-- tps限制，发送频率上限（安全考虑），超过的发送频率的消息将被扔掉 ，整数 -->
          <property name= "tpsLimit" value= "2048" />
 </bean>
    
 <bean id= "springContextUtil " class = "com.yirendai.infra.cicada.utils.SpringContextUtil" />

```

    在具体操作时，需要根据实际情况配置nginx地址， 其余地址保持默认的即可。

    connectTimeout和soTimeout设置的目的是防止nginx挂掉后，发送线程长时间被占用，故时间不宜过长

    批处理大小和缓存大小对应是消息条数，优化发送性能

    TPS的设置是限制每秒消息上限，目的是防止异常情况下，消息发送过快导致内存溢出。可适当增大，但别过大。

## 扩展功能

    为了丰富日志的信息，补充以下几种日志添加方式

### 1、注解拦截

       在任何使用了该注解的地方都可以进行拦截。

       首先开启拦截功能。 在采用asepectj的Aop方式时，该功能默认开启。

                                       在采用Spring Aop的实现方式时，需要在配置文件中添加配置 <bean class= "com.yirendai.infra.cicada.capture.TraceableAop" /> （Spring Boot对应转换实现）

      其次 在拦截的方法上添加注解@Traceable

 

### 2、自定义日志功能

```

Tracer. getInstance ().addBinaryAnntation("hi", "zpc");

Tracer. getInstance ().addBinaryAnntation(className,methodName,throwable);  

```

   

### 3、Mybatis拦截

功效：拦截Sql语句+执行时间

配置： 关键是添加 mybatisTraceInterceptor ， 如

```
 <bean id= "sqlSessionFactory" class = "org.mybatis.spring.SqlSessionFactoryBean" >
      <property name= "configLocation" value= "classpath:mybatis-config.xml" />
      <property name= "dataSource" ref= "dataSource" />
      <property name= "plugins" > 
          <array> 
              <bean id= "mybatisTraceInterceptor" class = "com.yirendai.infra.cicada.capture.MybatisInterceptor" /> 
          </array> 
      </property>     
 </bean>

```

### 4、抓取HttpClient的使用


        对于采用Asepectj的Aop方式，默认拦截

        对于采用Spring Aop的方式，需要手动开启拦截， 即定义拦截AOP <bean class= "com.yirendai.infra.cicada.capture.HttpClientAop" />（Spring Boot需要定义Configuration）
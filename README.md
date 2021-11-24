# eurekazuul工程
springcloud + springboot + eureka + zuul 集群

## 创建三个应用程序（每个应用程序部署2次）
API Gateway - Spring Cloud Netflix Zuul   
Service Registry - Spring Cloud Netflix Eureka   
REST Service - Spring Boot
## 环境
- JDK 1.8
- IDE
- Maven 3.0+ 
> 应用程序使用哪种架构风格并不重要，将相同的应用程序部署在不同区域的数据中心并使用某种技术将请求保持在同一区域内是一个常见的用例。在微服务架构中，也需要实现相同的功能，但需要使用服务注册表设计模式来应用该技术。
> Spring Cloud Netflix 可以轻松实现微服务所需的模式。

## 开始创建应用程序
- 基本依赖项为所有应用程序添加以下依赖项。如果任何特定应用程序有任何差异，将在每个特定线程中提及。
```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.2.5.RELEASE</version>
    <relativePath/>
</parent>
<properties>
    <spring-cloud.version>Hoxton.SR3</spring-cloud.version>
</properties>
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>  
```
### Service Registry
我们将创建的第一个应用程序是使用 Spring Cloud Netflix Eureka 的服务注册中心。   
首先，在 pom.xml 中添加以下依赖项。
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```
其次，Java代码
现在只需创建主 SpringApplication 类并添加 @EnableEurekaServer
```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@EnableEurekaServer
@SpringBootApplication
public class ServiceDiscoveryApplication {
    public static void main(String... args) {
        SpringApplication.run(ServiceDiscoveryApplication.class, args);
    }
}
```
### API Gateway
我们将创建的第二个应用程序是使用 Spring Cloud Netflix Zuul 的 API 网关。   
首先，在 pom.xml 中添加以下依赖项。
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
其次，Java代码
现在只需创建主 SpringApplication 类并添加 @EnableZuulProxy
```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;

@EnableZuulProxy
@SpringBootApplication
public class GatewayApplication {
    public static void main(String... args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```
### Simple REST Service
第三个应用程序仅包含一个 REST 接口，以确保来自每个区域的每个调用都将保留在请求的区域中。对于这个应用程序，在 pom.xml 中添加以下依赖项。
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```
其次，Java代码
现在只需创建主 SpringApplication 
```
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import static org.springframework.http.MediaType.APPLICATION_JSON_UTF8_VALUE;

@SpringBootApplication
public class SimpleServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(SimpleServiceApplication.class, args);
    }

    @RestController
    class SimpleController {
        @Value("${eureka.instance.metadataMap.zone}")
        private String zone;

        @GetMapping(value = "/zone", produces = APPLICATION_JSON_UTF8_VALUE)
        public String zone() {
            return "{\"zone\"=\"" + zone + "\"}";
        }
    }
}
```
## 应用程序的配置文件
如前所述，每个应用程序需要运行两次以模拟两个不同的区域，为了更容易地基于配置文件创建配置，为每个应用程序创建以下三个文件：
>src/main/resources/application.yml
>src/main/resources/application-zone1.yml
>src/main/resources/application-zone2.yml    

文件名中的后缀将用作配置文件名称。
### Service Registry
```
# src/main/resources/application.yml
eureka:
  server:
    enable-self-preservation: true
    eviction-interval-timer-in-ms: 30000
  client:
    register-with-eureka: false
    fetch-registry: false
    region: region-1
    service-url:
      zone1: http://localhost:8761/eureka/
      zone2: http://127.0.0.1:8762/eureka/
    availability-zones:
      region-1: zone1,zone2
spring.profiles.active: zone1
```
```
# src/main/resources/application-zone1.yml
server.port: 8761
eureka:
  instance:
    hostname: localhost
    metadataMap.zone: zone1
```
```
# src/main/resources/application-zone2.yml
server.port: 8762
eureka:
  instance:
    hostname: 127.0.0.1
    metadataMap.zone: zone2
```
对于 -zone1 和 -zone2 配置文件，唯一的区别是 server.port，eureka.metadataMap.zone 中配置的实际区域，在这种情况下是主机名，每个 Eureka Server 需要在不同的主机名中运行；因为我在同一台机器上运行它们，所以我将它命名为 127.0.01 和 localhost。   
如果您在不同的机器上运行，则无需添加主机名。
### Gateway
```
spring:
  profiles.active: zone1
  application.name: gateway

eureka:
  client:
    prefer-same-zone-eureka: true
    region: region-1
    service-url:
      zone1: http://localhost:8761/eureka/
      zone2: http://127.0.0.1:8762/eureka/
    availability-zones:
      region-1: zone1,zone2
    healthcheck:
      enabled: true
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${spring.application.instance_id:${spring.cloud.client.hostname}}:${server.port}
    lease-expiration-duration-in-seconds: 6
    lease-renewal-interval-in-seconds: 3

#开启重连机制
zuul:
  retryable: true
```
这里的主要区别是 eureka.client.prefer-same-zone-eureka 属性，它告诉应用程序，每当它需要调用另一个 EurekaClient 时，它将使用调用者部署的相同区域调用它。如果同一个区域中没有可用的客户端，它将从另一个可用的区域调用。
```
# src/main/resources/application-zone1.yml
server.port: 8080
eureka:
  instance:
    metadataMap.zone: zone1
```
```
# src/main/resources/application-zone2.yml
server.port: 8081
eureka:
  instance:
    metadataMap.zone: zone2
```
### Simple REST Service
服务本身的配置包含与网关相同的配置
```
# src/main/resources/application.yml
eureka:
  client:
    prefer-same-zone-eureka: true
    region: region-1
    service-url:
      zone1: http://localhost:8761/eureka/
      zone2: http://127.0.0.1:8762/eureka/
    availability-zones:
      region-1: zone1,zone2
    healthcheck:
      enabled: true
  instance:
    prefer-ip-address: true
    lease-expiration-duration-in-seconds: 6
    lease-renewal-interval-in-seconds: 3
    instanceId: ${spring.application.name}:${vcap.application.instance_id:${spring.application.instance_id:${spring.cloud.client.hostname}}}:${server.port}
spring:
  profiles.active: zone1
  application.name: simple-service
```
```
# src/main/resources/application-zone1.yml
server.port: 8181
eureka:
  instance:
    metadataMap.zone: zone1
```
```
# src/main/resources/application-zone2.yml
server.port: 8182
eureka:
  instance:
    metadataMap.zone: zone2
```
## 构建与运行
是时候构建应用程序了；如果您使用 maven 构建应用程序（就像我所做的那样），只需构建它们执行：
>$ mvn clean package

之后，只需运行将特定配置文件添加到命令行的每个应用程序，例如：
> $ java -jar target/*.jar --spring.profiles.active=zone1    

请记住，您需要运行每个应用程序两次，每个配置文件一次：zone1 和 zone2。   
为了验证请求网关，我们需要通过每个网关向简单服务发出请求。
> $ curl http://localhost:8080/simple-service/zone
>
>{"zone"="zone1"}

> $ curl http://localhost:8081/simple-service/zone
> 
>{"zone"="zone2"}

>-  这里的区别是 server.port。

故障转移验证  
要验证集群之间的故障转移，您只需要停止其中一个实例并向相反的zone发出请求，例如：
在 zone2 上停止 simple-service，然后通过 zone2 上的网关向 simple-service 发出请求
> $ curl http://localhost:8081/simple-service/zone
> 
> {"zone"="zone1"}  

一旦zone2 的REST服务启动、运行并在 Eureka 服务器中注册，相同的 curl 必须再次响应 {"zone"="zone 2"}。
# 总结：
您刚刚创建并配置了一个 API 网关、服务注册表和一个简单的REST服务，这些服务可以配置为集群，为您的微服务带来更多弹性和 HA。
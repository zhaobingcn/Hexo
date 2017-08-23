---
title: SpringBoot项目部署
date: 2017-07-25 19:36:44
tags:
    - SpringBoot
    - Server
---

## 常见问题
使用Spring Boot构建项目的时候因为默认构建的包为jar包，因此会出现一些部署上的困惑，在这里记录下来Sping Boot程序的部署过程，方便之后回顾。

### 修改打包形式
在pom.xml中设置打包格式<packaging>war</packaging>

### 移除服务器已有的插件
如果构建的是web程序，则需要排除tomcat。

``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- 移除嵌入式tomcat插件 -->
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

### 添加servlet-api的依赖
下面两种方法都可以，任选其一
``` xml
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>3.1.0</version>
    <scope>provided</scope>
</dependency>

```
第二种方法
``` xml
<dependency>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>tomcat-servlet-api</artifactId>
    <version>8.0.36</version>
    <scope>provided</scope>
</dependency>
```

### 修改启动类，并且重写初始化方法
我们常用的是用main方法启动，代码如下：
``` java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```
在这里我们需要使用类似web.xml的配制方法来启动spring上下文了，在Application的同级目录下添加SpringBootStartApplication类：

``` java
/**
 * 修改启动类，继承 SpringBootServletInitializer 并重写 configure 方法
 */
public class SpringBootStartApplication extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        // 注意这里要指向原先用main方法执行的Application启动类
        return builder.sources(Application.class);
    }
}
```
### 打包部署
然后需要使用mvn package命令将项目打包，把打包好的项目移动到tomcat或jetty的webapp目录下。重新启动jetty或者tomcat.

``` 
http://localhost:[端口号]/[项目名]
```

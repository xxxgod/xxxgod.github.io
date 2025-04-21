---
title: Springboot启动流程
date: 2019-03-20 16:41:14
tags: springboot
---

Spring Boot 启动流程主要包含一系列初始化、配置加载和上下文创建等操作，以下为你详细介绍各个阶段：

### 1. 启动入口

Spring Boot 应用启动的入口是 `SpringApplication.run` 方法，一般在 `main` 方法中调用，示例代码如下：

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

### 2. 创建 `SpringApplication` 实例

`SpringApplication.run` 方法内部会创建 `SpringApplication` 实例，并进行一些初始化操作：



- **推断应用类型**：判定应用属于哪种类型，如 Web 应用（包含 Servlet Web 应用、Reactive Web 应用）或者非 Web 应用。
- **加载初始化器和监听器**：从 `META - INF/spring.factories` 文件里加载 `ApplicationContextInitializer` 和 `ApplicationListener` 实例。

### 3. 执行 `SpringApplication.run` 方法

此方法是启动流程的核心，包含以下关键步骤：



- **记录启动时间**：利用 `StopWatch` 记录启动耗时。

- **触发启动事件**：借助 `SpringApplicationRunListeners` 触发 `starting` 事件，通知所有监听器应用开始启动。

- **准备应用环境**：创建并配置 `ConfigurableEnvironment`，加载配置文件、系统属性等。

- **打印启动横幅**：显示自定义或者默认的启动横幅。

- **创建应用上下文**：依据之前推断的应用类型，创建对应的 `ConfigurableApplicationContext` 实例。

- **准备应用上下文**：设置环境、注册监听器、加载主配置源等。

- 刷新应用上下文

  ：这是关键步骤，会触发 Spring 的自动配置机制，具体如下：

  - **加载自动配置类**：从 `META - INF/spring.factories` 文件中读取候选自动配置类。
  - **排除不需要的配置类**：依据 `@EnableAutoConfiguration` 注解的 `exclude` 和 `excludeName` 属性，排除不需要的自动配置类。
  - **条件过滤**：对候选配置类进行条件过滤，只有满足 `@Conditional` 注解条件的配置类才会被加载。
  - **加载配置类到容器**：将经过过滤后的自动配置类加载到 Spring 容器中。

- **触发启动完成事件**：触发 `started` 事件，通知监听器应用已启动。

- **调用运行器**：调用 `CommandLineRunner` 和 `ApplicationRunner` 实例，执行一些启动后的自定义逻辑。

- **触发运行中事件**：触发 `running` 事件，表明应用已成功启动并正在运行。

### 总结

Spring Boot 启动流程从 `SpringApplication.run` 方法开始，经过创建 `SpringApplication` 实例、执行一系列初始化和配置操作，最终完成应用上下文的创建和自动配置类的加载，使应用成功启动并运行。整个流程高度自动化，通过约定大于配置的原则，减少了开发者的配置工作。
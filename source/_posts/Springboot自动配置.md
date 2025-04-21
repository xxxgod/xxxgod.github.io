---
title: Springboot自动配置
date: 2023-04-21 16:23:26
tags: springboot
---

#### 1. 入口注解 `@SpringBootApplication`

`@SpringBootApplication` 是一个组合注解，它包含了 `@SpringBootConfiguration`、`@EnableAutoConfiguration` 和 `@ComponentScan` 三个重要注解。其中，`@EnableAutoConfiguration` 是开启自动配置的关键。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    // ...
}
```

#### 2. `@EnableAutoConfiguration` 注解

该注解通过 `@Import` 导入 `AutoConfigurationImportSelector` 类，这个类负责加载自动配置类。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    // ...
}
```

#### 3. `AutoConfigurationImportSelector` 类

`AutoConfigurationImportSelector` 类实现了 `DeferredImportSelector` 接口，在 `selectImports` 方法中完成自动配置类的加载。

```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return NO_IMPORTS;
    }
    AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
    return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}

protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
    }
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    configurations = removeDuplicates(configurations);
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    checkExcludedClasses(configurations, exclusions);
    configurations.removeAll(exclusions);
    configurations = getConfigurationClassFilter().filter(configurations);
    fireAutoConfigurationImportEvents(configurations, exclusions);
    return new AutoConfigurationEntry(configurations, exclusions);
}
```

#### 4. 获取候选自动配置类

`getCandidateConfigurations` 方法从 `META-INF/spring.factories` 文件中读取所有候选的自动配置类。

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
            getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
            + "are using a custom packaging, make sure that file is correct.");
    return configurations;
}
```

#### 5. 过滤自动配置类

在获取候选自动配置类后，会进行一系列的过滤操作，例如根据 `@Conditional` 注解进行条件判断，只有满足条件的自动配置类才会被加载。

### 步骤总结

1. **启动注解触发**：`@SpringBootApplication` 注解开启自动配置功能，其中 `@EnableAutoConfiguration` 注解导入 `AutoConfigurationImportSelector` 类。
2. **加载候选配置类**：`AutoConfigurationImportSelector` 类从 `META-INF/spring.factories` 文件中读取所有候选的自动配置类。
3. **排除不需要的配置类**：根据 `@EnableAutoConfiguration` 注解的 `exclude` 和 `excludeName` 属性，排除不需要的自动配置类。
4. **条件过滤**：对候选配置类进行条件过滤，只有满足 `@Conditional` 注解条件的配置类才会被加载。
5. **加载自动配置类**：将经过过滤后的自动配置类加载到 Spring 容器中。
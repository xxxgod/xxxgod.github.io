---
title: Spring Bean的生命周期
date: 2018-03-01 12:01:28
tags: spring
---

Spring Bean 的生命周期包含了从创建到销毁的整个过程，主要包括实例化、属性赋值、初始化和销毁等阶段。下面将结合源码对其生命周期进行详细解析。

### 源码解析

#### 1. 实例化 Bean

在 `AbstractAutowireCapableBeanFactory` 类的 `createBeanInstance` 方法中完成 Bean 的实例化。

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
    // 解析 Bean 的类
    Class<?> beanClass = resolveBeanClass(mbd, beanName);

    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
    }

    // 使用构造函数解析器来实例化 Bean
    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
    if (instanceSupplier != null) {
        return obtainFromSupplier(instanceSupplier, beanName);
    }

    if (mbd.getFactoryMethodName() != null) {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    // 尝试使用默认构造函数实例化
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
        synchronized (mbd.constructorArgumentLock) {
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
    if (resolved) {
        if (autowireNecessary) {
            return autowireConstructor(beanName, mbd, null, null);
        }
        else {
            return instantiateBean(beanName, mbd);
        }
    }

    // 自动装配构造函数
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
            mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    // 使用默认构造函数
    return instantiateBean(beanName, mbd);
}
```

#### 2. 属性赋值

在 `populateBean` 方法中进行 Bean 的属性赋值。

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    if (bw == null) {
        if (mbd.hasPropertyValues()) {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
        }
        else {
            // Skip property population phase for null instance.
            return;
        }
    }

    // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
    // state of the bean before properties are set. This can be used, for example,
    // to support styles of field injection.
    boolean continueWithPropertyPopulation = true;

    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    continueWithPropertyPopulation = false;
                    break;
                }
            }
        }
    }

    if (!continueWithPropertyPopulation) {
        return;
    }

    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME || mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

        // Add property values based on autowire by name if applicable.
        if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }

        // Add property values based on autowire by type if applicable.
        if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }

        pvs = newPvs;
    }

    if (hasInstAwareBpps()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
                if (pvsToUse == null) {
                    if (this.defaultMergeable == null) {
                        throw new IllegalStateException("Cannot determine default mergeable settings for " +
                                "bean '" + beanName + "' - " +
                                "inconsistent values in beans of the same type");
                    }
                    pvsToUse = (pvs instanceof MutablePropertyValues ?
                            ((MutablePropertyValues) pvs).mergeIfRequired(this.defaultMergeable) : pvs);
                }
                pvs = pvsToUse;
            }
        }
    }

    if (pvs != null) {
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```

#### 3. 初始化 Bean

在 `initializeBean` 方法中进行 Bean 的初始化操作，包括调用 `BeanNameAware`、`BeanFactoryAware` 等接口的方法，以及执行自定义的初始化方法。

```java
protected Object initializeBean(String beanName, Object bean, @Nullable RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    }
    else {
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                (mbd != null ? mbd.getResourceDescription() : null),
                beanName, "Invocation of init method failed", ex);
    }
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}
```

#### 4. 销毁 Bean

在 `AbstractBeanFactory` 类的 `destroyBean` 方法中进行 Bean 的销毁操作，包括调用 `DisposableBean` 接口的 `destroy` 方法和自定义的销毁方法。

```java
public void destroyBean(String beanName, @Nullable Object beanInstance) {
    // Trigger destruction of dependent beans first...
    Set<String> dependencies = this.dependentBeanMap.remove(beanName);
    if (dependencies != null) {
        if (logger.isTraceEnabled()) {
            logger.trace("Retrieved dependent beans for bean '" + beanName + "': " + dependencies);
        }
        for (String dependentBeanName : dependencies) {
            destroySingleton(dependentBeanName);
        }
    }

    // Actually destroy the bean now...
    if (beanInstance != null) {
        if (hasDestructionAwareBeanPostProcessors()) {
            for (DestructionAwareBeanPostProcessor processor : getDestructionAwareBeanPostProcessors()) {
                processor.postProcessBeforeDestruction(beanInstance, beanName);
            }
        }
        // Actually invoke the destroy method if necessary...
        if (beanInstance instanceof DisposableBean) {
            try {
                ((DisposableBean) beanInstance).destroy();
            }
            catch (Throwable ex) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Invocation of destroy method failed on bean with name '" + beanName + "'", ex);
                }
            }
        }
        if (hasBeanCreationStarted()) {
            removeSingleton(beanName);
        }
    }

    // Remove destroyed bean from other beans' dependencies.
    this.dependentBeanMap.values().removeIf(dependencies -> dependencies.remove(beanName));
    this.dependenciesForBeanMap.remove(beanName);

    // Remove corresponding manual singleton registration.
    if (containsSingleton(beanName)) {
        removeSingleton(beanName);
    }
}
```





### 总结

Spring Bean 的生命周期主要包括以下几个阶段：



1. **实例化**：通过构造函数或工厂方法创建 Bean 的实例。

2. **属性赋值**：为 Bean 的属性注入依赖。

3. 初始化

   ：

   - 调用 `Aware` 接口的方法，如 `BeanNameAware`、`BeanFactoryAware` 等。
   - 调用 `BeanPostProcessor` 的 `postProcessBeforeInitialization` 方法。
   - 调用自定义的初始化方法，如 `@PostConstruct` 注解的方法或 `init-method` 属性指定的方法。
   - 调用 `BeanPostProcessor` 的 `postProcessAfterInitialization` 方法。

4. **使用**：Bean 可以被应用程序使用。

5. 销毁

   ：

   - 调用 `DestructionAwareBeanPostProcessor` 的 `postProcessBeforeDestruction` 方法。
   - 调用 `DisposableBean` 接口的 `destroy` 方法或自定义的销毁方法，如 `@PreDestroy` 注解的方法或 `destroy-method` 属性指定的方法。



通过这些阶段，Spring 框架确保了 Bean 的正确创建、初始化和销毁，同时提供了丰富的扩展点，允许开发者在不同阶段进行自定义操作。
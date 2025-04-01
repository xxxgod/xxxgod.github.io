---
title: Spring源码解析
date: 2021-08-21 22:46:40
categories:
- java
tags: spring
---


一.IOC源码深度剖析
1.一般第一步都是加载xml文件，创建一个ApplicationContext容器
```
public static void main(String[] args) {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
    TestBean testBean = (TestBean) applicationContext.getBean("testBean");
    testBean.print();
}

```

2.进入 new ClassPathXmlApplicationContext("classpath:applicationContext.xml")这个构造方法；
```
public ClassPathXmlApplicationContext(
		String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
		throws BeansException {
	super(parent);
    //根据文件路径，处理配置文件
	setConfigLocations(configLocations);
	if (refresh) {
        //核心方法
		refresh();
	}
}

```
3.进入核心方法refresh的源码
```
@Override
public void refresh() throws BeansException, IllegalStateException {
   // 来个锁，不然 refresh() 还没结束，你又来个启动或销毁容器的操作，那不就乱套了嘛
   synchronized (this.startupShutdownMonitor) {
 
      // 准备工作，记录下容器的启动时间、标记“已启动”状态、处理配置文件中的占位符
      prepareRefresh();
 
      // 这步比较关键，这步完成后，配置文件就会解析成一个个 Bean 定义，注册到 BeanFactory 中，
      // 当然，这里说的 Bean 还没有初始化，只是配置信息都提取出来了，
      // 注册也只是将这些信息都保存到了注册中心(说到底核心是一个 beanName-> beanDefinition 的 map)
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
 
      // 设置 BeanFactory 的类加载器，添加几个 BeanPostProcessor，手动注册几个特殊的 bean
      // 这块待会会展开说
      prepareBeanFactory(beanFactory);
 
      try {
         // 【这里需要知道 BeanFactoryPostProcessor 这个知识点，Bean 如果实现了此接口，
         // 那么在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory 方法。】
 
         // 这里是提供给子类的扩展点，到这里的时候，所有的 Bean 都加载、注册完成了，但是都还没有初始化
         // 具体的子类可以在这步的时候添加一些特殊的 BeanFactoryPostProcessor 的实现类或做点什么事
         postProcessBeanFactory(beanFactory);
         // 调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory(factory) 回调方法
         invokeBeanFactoryPostProcessors(beanFactory);          
 
 
 
         // 注册 BeanPostProcessor 的实现类，注意看和 BeanFactoryPostProcessor 的区别
         // 此接口两个方法: postProcessBeforeInitialization 和 postProcessAfterInitialization
         // 两个方法分别在 Bean 初始化之前和初始化之后得到执行。这里仅仅是注册，之后会看到回调这两方法的时机
         registerBeanPostProcessors(beanFactory);
 
         // 初始化当前 ApplicationContext 的 MessageSource，国际化这里就不展开说了，不然没完没了了
         initMessageSource();
 
         // 初始化当前 ApplicationContext 的事件广播器，这里也不展开了
         initApplicationEventMulticaster();
 
         // 从方法名就可以知道，典型的模板方法(钩子方法)，不展开说
         // 具体的子类可以在这里初始化一些特殊的 Bean（在初始化 singleton beans 之前）
         onRefresh();
 
         // 注册事件监听器，监听器需要实现 ApplicationListener 接口。这也不是我们的重点，过
         registerListeners();
 
         // 重点，重点，重点
         // 初始化所有的 singleton beans
         //（lazy-init 的除外）
         finishBeanFactoryInitialization(beanFactory);
 
         // 最后，广播事件，ApplicationContext 初始化完成，不展开
         finishRefresh();
      }
 
      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }
 
         // Destroy already created singletons to avoid dangling resources.
         // 销毁已经初始化的 singleton 的 Beans，以免有些 bean 会一直占用资源
         destroyBeans();
 
         // Reset 'active' flag.
         cancelRefresh(ex);
 
         // 把异常往外抛
         throw ex;
      }
 
      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         resetCommonCaches();
      }
   }
}
```
（1）【1.准备刷新 】prepareRefresh();
        主要是一些准备工作设置其启动日期和活动标志以及执行一些属性的初始化。（不是很重要的方法）

（2）【2.初始化 BeanFactory 】obtainFreshBeanFactory()；（重点方法）

        <1>如果有旧的BeanFactory就删除并创建新的BeanFactory

        <2>解析所有的Spring配置文件，将配置文件中定义的bean封装成BeanDefinition，加载到BeanFactory中（这里只注册，不会进行Bean的实例化）

（3）【3.bean工厂前置操作】prepareBeanFactory(beanFactory);

        配置 beanFactory 的标准上下文特征，例如上下文的 ClassLoader、后置处理器等。

（4）【4.bean工厂后置操作】postProcessBeanFactory(beanFactory);

        空方法，如果子类需要，自己去实现 

（5）【5.调用bean工厂后置处理器】invokeBeanFactoryPostProcessors(beanFactory);

        实例化和调用所有BeanFactoryPostProcessor，完成类的扫描、解析和注册

        BeanFactoryPostProcessor 接口是 Spring 初始化 BeanFactory 时对外暴露的扩展点，Spring IoC 容器允许 BeanFactoryPostProcessor 在容器实例化任何 bean 之前读取 bean 的定义，并可以修改它。

（6）【6.注册bean后置处理器】registerBeanPostProcessors(beanFactory);

        将所有实现了 BeanPostProcessor 接口的类注册到 BeanFactory 中。

（7）【7.初始化消息源】initMessageSource();

        初始化MessageSource组件（做国际化功能；消息绑定，消息解析）

（8）【8.初始化事件广播器】initApplicationEventMulticaster();

        初始化应用的事件广播器 ApplicationEventMulticaster。

（9）【9.刷新:拓展方法】onRefresh();

        空方法，模板设计模式;子类重写该方法并在容器刷新的时候自定义逻辑；

        例：springBoot在onRefresh() 完成内置Tomcat的创建及启动

（10）【10.注册监听器】registerListeners();

        向事件分发器注册硬编码设置的applicationListener，向事件分发器注册一个IOC中的事件监听器（并不实例化）

（11）【11.实例化所有非懒加载的单例bean】finishBeanFactoryInitialization(beanFactory);

        是整个Spring IoC核心中的核心

（12）【12.结束refresh操作】finishRefresh();

        发布事件，ApplicationContext 初始化完成

4.核心方法源码解析
【2.初始化 BeanFactory 】obtainFreshBeanFactory()；

        这一步上面简单介绍过了，作用是把配置文件解析成一个个BeanBeanDefinition，并且注册到BeanFactory中，点进去源码：
```
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
	refreshBeanFactory();
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (logger.isDebugEnabled()) {
		logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
	}
	return beanFactory;
}
        我们进入refreshBeanFactory()方法

@Override
protected final void refreshBeanFactory() throws BeansException {
    //如果存在旧的BeanFactory就删除
	if (hasBeanFactory()) {
		destroyBeans();
		closeBeanFactory();
	}
	try {
        //创建新的BeanFactory
		DefaultListableBeanFactory beanFactory = createBeanFactory();
        //序列化
		beanFactory.setSerializationId(getId());
        //是否允许 Bean 覆盖、是否允许循环引用
		customizeBeanFactory(beanFactory);
        //根据配置，加载各个Bean，然后放到 BeanFactory 中
		loadBeanDefinitions(beanFactory);
		synchronized (this.beanFactoryMonitor) {
			this.beanFactory = beanFactory;
		}
	}
	catch (IOException ex) {
		throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	}
}

        这个方法的作用是关闭旧的 BeanFactory (如果有)，创建新的 BeanFactory，加载 Bean 定义、注册 Bean 等。
```
【11.实例化所有非懒加载的单例bean】finishBeanFactoryInitialization(beanFactory);

在这个方法中，我们主要做了以下操作：
（1）将之前解析的 BeanDefinition 进一步处理，将有父 BeanDefinition 的进行合并，获得MergedBeanDefinition
（2）尝试从缓存获取 bean 实例
（3）处理特殊的 bean —— FactoryBean 的创建
（4）创建 bean 实例
（5）循环引用的处理[spring的三级缓存就是在这一步，后面详述]
（6）bean 实例属性填充
（7）bean 实例的初始化
（8）BeanPostProcessor 的各种扩展应用
这个方法解析的结束，也标志着Spring IoC重要内容基本都已经解析完毕
```
        finishBeanFactoryInitialization(beanFactory)点进去看源码

protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
	// 5.实例化所有剩余（非懒加载）单例对象
	beanFactory.preInstantiateSingletons();
}
        preInstantiateSingletons();点进去看源码

@Override
public void preInstantiateSingletons() throws BeansException {
	List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
	for (String beanName : beanNames) {
		RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
		if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            //判断beanName对应的bean是否为FactoryBean
			if (isFactoryBean(beanName)) {
				Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
				if (bean instanceof FactoryBean) {
					final FactoryBean<?> factory = (FactoryBean<?>) bean;
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
										((SmartFactoryBean<?>) factory)::isEagerInit,
								getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						//如果希望急切的初始化，则通过beanName获取bean实例
						getBean(beanName);
					}
				}
			}
			else {
				//如果beanName对应的bean不是FactoryBean，只是普通Bean，通过beanName获取bean实例
				getBean(beanName);
			}
		}
	}
	//遍历beanNames，触发所有SmartInitializingSingleton的后初始化回调
	for (String beanName : beanNames) {
		Object singletonInstance = getSingleton(beanName);
		if (singletonInstance instanceof SmartInitializingSingleton) {
			final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
			if (System.getSecurityManager() != null) {
				AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
					smartSingleton.afterSingletonsInstantiated();
					return null;
				}, getAccessControlContext());
			}
			else {
				smartSingleton.afterSingletonsInstantiated();
			}
		}
	}
}

             getBean();点进去看源码

@Override
public Object getBean(String name) throws BeansException {
    // 获取name对应的bean实例，如果不存在，则创建一个
    return doGetBean(name, null, null, false);
}
              doGetBean();点进去看源码

protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
 
	// 1.解析beanName，主要是解析别名、去掉FactoryBean的前缀“&”		
	final String beanName = transformedBeanName(name);
	Object bean;
	// 2.尝试从缓存中获取beanName对应的实例
	Object sharedInstance = getSingleton(beanName);
	// 3.如果beanName的实例存在于缓存中
	if (sharedInstance != null && args == null) {
		if (logger.isDebugEnabled()) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
						"' that is not fully initialized yet - a consequence of a circular reference");
			}
			else {
				logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
			}
		}
		// 3.1 返回beanName对应的实例对象（主要用于FactoryBean的特殊处理，普通Bean会直接返回sharedInstance本身）
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}
 
	else {
		// 4.scope为prototype的循环依赖校验：如果beanName已经正在创建Bean实例中，而此时我们又要再一次创建beanName的实例，则代表出现了循环依赖，需要抛出异常。
		if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}
 
		// 5.获取parentBeanFactory
		BeanFactory parentBeanFactory = getParentBeanFactory();
		// 5.1 如果parentBeanFactory存在，并且beanName在当前BeanFactory不存在Bean定义，则尝试从parentBeanFactory中获取bean实例
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// 5.2 将别名解析成真正的beanName
			String nameToLookup = originalBeanName(name);
			if (parentBeanFactory instanceof AbstractBeanFactory) {
				return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
						nameToLookup, requiredType, args, typeCheckOnly);
			}
			// 5.3 尝试在parentBeanFactory中获取bean对象实例
			else if (args != null) {
				// Delegation to parent with explicit args.
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			}
			else {
				// No args -> delegate to standard getBean method.
				return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
		}
		// 6.如果不是仅仅做类型检测，而是创建bean实例，这里要将beanName放到alreadyCreated缓存
		if (!typeCheckOnly) {
			markBeanAsCreated(beanName);
		}
 
		try {
			// 7.根据beanName重新获取MergedBeanDefinition（步骤6将MergedBeanDefinition删除了，这边获取一个新的）
			final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			checkMergedBeanDefinition(mbd, beanName, args);
 
			// 8.拿到当前bean依赖的bean名称集合，在实例化自己之前，需要先实例化自己依赖的bean
			String[] dependsOn = mbd.getDependsOn();
			if (dependsOn != null) {
				for (String dep : dependsOn) {
					// 8.2 检查dep是否依赖于beanName，即检查是否存在循环依赖
					if (isDependent(beanName, dep)) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
					}
					// 8.4 将dep和beanName的依赖关系注册到缓存中
					registerDependentBean(dep, beanName);
					try {
						// 8.5 获取dep对应的bean实例，如果dep还没有创建bean实例，则创建dep的bean实例
						getBean(dep);
					}
					catch (NoSuchBeanDefinitionException ex) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
					}
				}
			}
 
			// 9.针对不同的scope进行bean的创建
			if (mbd.isSingleton()) {
				sharedInstance = getSingleton(beanName, () -> {
					try {
						// 9.1.1 创建Bean实例
						return createBean(beanName, mbd, args);
					}
					catch (BeansException ex) {
						destroySingleton(beanName);
						throw ex;
					}
				});
				bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
			}
 
			else if (mbd.isPrototype()) {
				// 9.2 scope为prototype的bean创建
				Object prototypeInstance = null;
				try {
					beforePrototypeCreation(beanName);
					prototypeInstance = createBean(beanName, mbd, args);
				}
				finally {
					afterPrototypeCreation(beanName);
				}
				bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
			}
	// 10.返回创建出来的bean实例对象
	return (T) bean;
}

        getSingleton(beanName); 点进去看源码

protected Object getSingleton(String beanName, boolean allowEarlyReference) {
	// 1.从单例对象缓存中获取beanName对应的单例对象
	Object singletonObject = this.singletonObjects.get(beanName);
	// 2.如果单例对象缓存中没有，并且该beanName对应的单例bean正在创建中
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		// 3.加锁进行操作
		synchronized (this.singletonObjects) {
			// 4.从早期单例对象缓存中获取单例对象（之所称成为早期单例对象，是因为
			earlySingletonObjects里
			// 的对象的都是通过提前曝光的ObjectFactory创建出来的，还未进行属性填充等操作）
			singletonObject = this.earlySingletonObjects.get(beanName);
			// 5.如果在早期单例对象缓存中也没有，并且允许创建早期单例对象引用
			if (singletonObject == null && allowEarlyReference) {
				// 6.从单例工厂缓存中获取beanName的单例工厂
				ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
				if (singletonFactory != null) {
					// 7.如果存在单例对象工厂，则通过工厂创建一个单例对象
					singletonObject = singletonFactory.getObject();
					// 8.将通过单例对象工厂创建的单例对象，放到早期单例对象缓存中
					this.earlySingletonObjects.put(beanName, singletonObject);
					// 9.移除该beanName对应的单例对象工厂，因为该单例工厂已经创建了一个实
					例对象，并且放到earlySingletonObjects缓存了，
					// 因此，后续获取beanName的单例对象，可以通过earlySingletonObjects
					缓存拿到，不需要在用到该单例工厂
					this.singletonFactories.remove(beanName);
				}
			}
		}
	}
	// 10.返回单例对象
	return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
public boolean isSingletonCurrentlyInCreation(String beanName) {
	return this.singletonsCurrentlyInCreation.contains(beanName);
}

        这段代码很重要，在正常情况下，该代码很普通，只是正常的检查下我们要拿的 bean 实例是否存在于缓存中，如果有就返回缓存中的 bean 实例，否则就返回 null。
        这段代码之所以重要，是因为该段代码是 Spring 解决循环引用的核心代码。解决循环引用逻辑：使用构造函数创建一个 “不完整” 的 bean 实例（之所以说不完整，是因为此时该bean 实例还未初始化），并且提前曝光该 bean 实例的 ObjectFactory（提前曝光就是将
ObjectFactory 放到 singletonFactories 缓存），通过 ObjectFactory 我们可以拿到该 bean 实例的引用，如果出现循环引用，我们可以通过缓存中的 ObjectFactory 来拿到 bean 实例，从而避免出现循环引用导致的死循环。这边通过缓存中的 ObjectFactory 拿到的 bean 实例虽然拿到的是 “不完整” 的bean 实例，但是由于是单例，所以后续初始化完成后，该 bean 实例的引用地址并不会变，所以最终我们看到的还是完整 bean 实例。

另外这个代码块中引进了4个重要缓存：
（1）singletonObjects 缓存：beanName -> 单例 bean 对象。
（2）earlySingletonObjects 缓存：beanName -> 单例 bean 对象，该缓存存放的是早期单例 bean 对象，可以理解成还未进行属性填充、初始化。
（3）singletonFactories 缓存：beanName -> ObjectFactory。
【singletonObjects、earlySingletonObjects、singletonFactories 在这边构成了一个类似于 “一、二、三级缓存” 的概念】
（4）singletonsCurrentlyInCreation 缓存：当前正在创建单例 bean 对象的 beanName 集合。

         createBean(beanName, mbd, args);点进去看源码

@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
    ...
	try {
        //创建Bean实例（真正创建Bean的方法）
		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
		if (logger.isDebugEnabled()) {
			logger.debug("Finished creating instance of bean '" + beanName + "'");
		}
		return beanInstance;
	}
    ...
}
        doCreateBean(beanName, mbdToUse, args);点进去看源码

protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {
	......
	Object exposedObject = bean;
	try {
		//对bean进行属性填充
		populateBean(beanName, mbd, instanceWrapper);
		//对bean进行初始化，方法中激活了Aware
		exposedObject = initializeBean(beanName, exposedObject, mbd);
	}
	......
	//完成创建并返回
	return exposedObject;
}
```



二.Spring的循环依赖
1.Spring解决循环依赖
        Spring的循环依赖的理论依据其实是基于Java的引用传递，当我们获取到对象的引用时，对象的field或属性是可以延后设置的(但是构造器必须是在获取引用之前)。

Spring为了解决单例的循环依赖问题，使用了三级缓存（三个map）。

（1）一级缓存singletonObjects：spring容器，用于存放完整的bean实例

private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);
（2）二级缓存earlySingletonObjects：判断bean是否存在AOP，不存在AOP保存半成品的bean（属性暂未填充），存在AOP保存代理bean的beanProxy（目标bean还是半成品的）

private final Map<String, Object> earlySingletonObjects = new HashMap<String,Object>(16);
（3）三级缓存singletonFactories：存放ObjectFactory，传入的是一个匿名内部类，如果bean被代理返回代理对象，如果bean未被代理返回原bean实例

private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);
2.Spring解决循环依赖流程图


3.循环依赖的经典面试题
（1）【Spring 为何需要三级缓存解决循环依赖，而不是二级缓存？】

        答：只要两个缓存确实可以做到解决循环依赖的问题，但是有一个前提这个bean没被AOP进行切面代理，如果这个bean被AOP进行了切面代理，那么只使用两个缓存是无法解决问题。

（2）【三级缓存中为什么要添加 ObjectFactory 对象，而不是直接保存实例对象？】

        答：假如想对添加到三级缓存中的实例对象进行增强，直接用实例对象是行不通的。

（3）【构造器注入注入的循环依赖为什么无法解决？】

        答：因为我们要先用构造函数创建一个 “不完整” 的 bean 实例，如果构造器出现循环依赖，我们连不完整的 bean 实例都构建不出来。

三.AOP源码深度剖析
1.AOP的作用
        功能增强，比如日志管理、事务管理

2.Spring AOP底层实现机制
        Spring AOP 底层实现机制目前有两种：JDK 动态代理、CGLIB动态字节码生成。

（1）JDK 动态代理
```
@Test
public void test(){
    IPerson target = new ManPerson();
    IPerson proxy = (IPerson)Proxy.newProxyInstance(
        target.getClass().getClassLoader(), 
        target.getClass().getInterfaces(), 
        new PersonInvocationHandler(target));
    proxy.eat();
    proxy.sleep();
}
 
// 代理对象
class PersonInvocationHandler implements InvocationHandler{
    private Object target;
	
    public PersonInvocationHandler(Object target){
        this.target = target;
    }
	
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable{
        System.out.println("start");
	Object result = method.invoke(target, args);
	System.out.println("end");
		
	return result;
    }
}
 
// 目标对象
class ManPerson implements IPerson{
    @Override
    public void eat(){
	System.out.println("吃饭中......");
    }
	
    @Override
    public void sleep(){
	System.out.println("睡觉中......");
    }
}
 
// 目标对象接口
interface IPerson{
    void eat();
	
    void sleep();
}
```
（2）CGLIB 动态代理
```
@Test
public void test(){
    Person proxy = (Person)Enhancer.create(Person.class, new PersonMethodInterceptor());
    proxy.eat();
    proxy.sleep();
}
 
// 代理对象
class PersonMethodInterceptor implements MethodInterceptor{
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable{
        System.out.println("start");
	Object result = methodProxy.invokeSuper(o, objects);
	System.out.println("end");
		
	return result;
    }
}
//目标对象
public class Person{
    public void eat(){
	System.out.println("吃饭中......");
    }
 
    public void sleep(){
	System.out.println("睡觉中......");
    }
}
```
（3）JDK 动态代理与CGLIB 动态代理的区别

        <1>JDK动态代理是实现了被代理对象的接口，Cglib是继承了被代理对象。

        <2>JDK调用代理方法，是通过反射机制调用，Cglib是通过FastClass机制直接调用方法，Cglib执行效率更高。

3.Spring AOP使用
```
//UserService接口
public interface UserService {
    public void findAll();
}
//UserService实现类
@Service
public class UserServiceImpl implements UserService{
    public void findAll(){
        System.out.println("findAll...");
    }
}
//切面类
@Component
@Aspect
public class AopAspect {
    @Pointcut("execution(* com.itheima.service..*.*(..))")
    public void pointcut() {
    }
    @Before("pointcut()")
    public void before() {
        System.out.println("before");
    }
 
    @After("pointcut()")
    public void after() {
        System.out.println("after advice");
    }
    @Around("pointcut()")
    public Object around(ProceedingJoinPoint proceedingJoinPoint) throws InterruptedException {
        System.out.println("around advice start");
        try {
            Object result = proceedingJoinPoint.proceed();
            System.out.println("result: " + result);
            System.out.println("around advice end");
            return result;
        } catch (Throwable throwable) {
            throwable.printStackTrace();
            return null;
        }
    }
}
```


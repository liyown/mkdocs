---
title: springcloud[boot]配置中心的自动刷新机制
date: 2024-07-07
authors: [刘耀文]
categories:
  - 源码阅读

---

# springcloud[boot]配置中心的自动刷新机制


## 实现机制：

### bean属性的自动刷新原理：

在spring2的时候，新增了自定义作用域，也就是除了单例和原型，新增了scope注解和接口，以便提高bean的储存生命周期，与它相关的接口和类为
<!-- more -->
```java
ConfigurableBeanFactory.registerScope, 
CustomScopeConfigurer, 
org.springframework.aop.scope.ScopedProxyFactoryBean, org.springframework.web.context.request.RequestScope,
org.springframework.web.context.request.SessionScope
```
实现的基本原理为
- 包扫描的时候，识别到@scope接口后将beandefination修改为ScopedProxyFactoryBean，具体在
```java
	protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
		for (String basePackage : basePackages) {
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				if (candidate instanceof AbstractBeanDefinition abstractBeanDefinition) {
					postProcessBeanDefinition(abstractBeanDefinition, beanName);
				}
				if (candidate instanceof AnnotatedBeanDefinition annotatedBeanDefinition) {
					AnnotationConfigUtils.processCommonDefinitionAnnotations(annotatedBeanDefinition);
				}
                // 也就是在这里，找到所有scope注解的类，将定义更换为ScopedProxyFactoryBean
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					definitionHolder =
							AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
```
跟进去代码发现
```java
    public static BeanDefinitionHolder createScopedProxy(BeanDefinitionHolder definition, BeanDefinitionRegistry registry, boolean proxyTargetClass) {
        String originalBeanName = definition.getBeanName();
        BeanDefinition targetDefinition = definition.getBeanDefinition();
        String targetBeanName = getTargetBeanName(originalBeanName);
        // 可以发现，在这里创建了ScopedProxyFactoryBean的bean定义
        RootBeanDefinition proxyDefinition = new RootBeanDefinition(ScopedProxyFactoryBean.class);
        // 将原始类设置进去，也就是被scope注解的类
        proxyDefinition.setDecoratedDefinition(new BeanDefinitionHolder(targetDefinition, targetBeanName));
        proxyDefinition.setOriginatingBeanDefinition(targetDefinition);
        proxyDefinition.setSource(definition.getSource());
        proxyDefinition.setRole(targetDefinition.getRole());
        proxyDefinition.getPropertyValues().add("targetBeanName", targetBeanName);
        if (proxyTargetClass) {
            targetDefinition.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
        } else {
            proxyDefinition.getPropertyValues().add("proxyTargetClass", Boolean.FALSE);
        }

        proxyDefinition.setAutowireCandidate(targetDefinition.isAutowireCandidate());
        proxyDefinition.setPrimary(targetDefinition.isPrimary());
        if (targetDefinition instanceof AbstractBeanDefinition abd) {
            proxyDefinition.copyQualifiersFrom(abd);
        }

        targetDefinition.setAutowireCandidate(false);
        targetDefinition.setPrimary(false);
        registry.registerBeanDefinition(targetBeanName, targetDefinition);
        return new BeanDefinitionHolder(proxyDefinition, originalBeanName, definition.getAliases());
    }

```
那么，ScopedProxyFactoryBean做了什么呢？为什么要修改为这个类，我们看看这个类的结构
```java
// 可以看到，实现了FactoryBean（肯定的，不然怎么获取bean，只是将原始bean作了进一步包装），还有BeanFactoryAware，这个是为了获取到BeanFactory从而获取到实例bean，AopInfrastructureBean这个接口表明这个类可以实现aop的逻辑，标记不会被aop自己包装，实现这个接口的还有非常重要的InfrastructureAdvisorAutoProxyCreator和AspectJAwareAdvisorAutoProxyCreator，一个是spring自带的aop，一个是启用AspectJ的aop，这部分内容也很有趣，有时间再更新一篇关于springaop的文章
public class ScopedProxyFactoryBean extends ProxyConfig implements FactoryBean<Object>, BeanFactoryAware, AopInfrastructureBean

```
既然实现了FactoryBean，我们看看这里获取的bean做了什么处理，很简单，首先，在设置BeanFactory的时候生成代理
```java
    public void setBeanFactory(BeanFactory beanFactory) {
        if (beanFactory instanceof ConfigurableBeanFactory cbf) {
            this.scopedTargetSource.setBeanFactory(beanFactory);
            ProxyFactory pf = new ProxyFactory();
            pf.copyFrom(this);
            pf.setTargetSource(this.scopedTargetSource);
            Assert.notNull(this.targetBeanName, "Property 'targetBeanName' is required");
            Class beanType = beanFactory.getType(this.targetBeanName);
            if (beanType == null) {
                throw new IllegalStateException("Cannot create scoped proxy for bean '" + this.targetBeanName + "': Target type could not be determined at the time of proxy creation.");
            } else {
                if (!this.isProxyTargetClass() || beanType.isInterface() || Modifier.isPrivate(beanType.getModifiers())) {
                    pf.setInterfaces(ClassUtils.getAllInterfacesForClass(beanType, cbf.getBeanClassLoader()));
                }

                ScopedObject scopedObject = new DefaultScopedObject(cbf, this.scopedTargetSource.getTargetBeanName());
                pf.addAdvice(new DelegatingIntroductionInterceptor(scopedObject));
                pf.addInterface(AopInfrastructureBean.class);
                this.proxy = pf.getProxy(cbf.getBeanClassLoader());
            }
        } else {
            throw new IllegalStateException("Not running in a ConfigurableBeanFactory: " + beanFactory);
        }
    }
```
然后获取bean的时候返回代理
```java
    public Object getObject() {
        if (this.proxy == null) {
            throw new FactoryBeanNotInitializedException();
        } else {
            return this.proxy;
        }
    }
```
生成的代理有什么用呢？进一步看看代理类的逻辑，其实就是在每次调用方法的时候，会先进入aop回调方法，去获取原始对象
```java

		public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
			Object oldProxy = null;
			boolean setProxyContext = false;
			Object target = null;
			TargetSource targetSource = this.advised.getTargetSource();
			try {
				if (this.advised.exposeProxy) {
					// Make invocation available if necessary.
					oldProxy = AopContext.setCurrentProxy(proxy);
					setProxyContext = true;
				}
                //也就是这里，获取beanfactory获取原始对象
				target = targetSource.getTarget();
				Class<?> targetClass = (target != null ? target.getClass() : null);
				List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
				Object retVal;

				if (chain.isEmpty()) {

					Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
					retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
				}
				else {
					// We need to create a method invocation...
					retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
				}
				return processReturnType(proxy, target, method, args, retVal);
			}
			finally {
				if (target != null && !targetSource.isStatic()) {
					targetSource.releaseTarget(target);
				}
				if (setProxyContext) {
					// Restore old proxy.
					AopContext.setCurrentProxy(oldProxy);
				}
			}
		}
```
那么问题回到开始，哪里起到了动态刷新呢？其实如果在配置中心的配置改变时候，将bean销毁掉，那么下次调用的时候去获取不是就可以获取最新的bean吗，确实，springcoud就是这个做的，springcloud使用refreshscope注解配合RefreshScope类，refreshscope注解将包装为ScopedProxyFactoryBean，RefreshScope类负责处理bean的生命周期，也就是说，获取的bean不在是去原始的beanfacroty中获取，而是到RefreshScope中获取，源码如下
```java
if (mbd.isSingleton()) {
    sharedInstance = getSingleton(beanName, () -> {
        try {
            return createBean(beanName, mbd, args);
        }
        catch (BeansException ex) {
            // Explicitly remove instance from singleton cache: It might have been put there
            // eagerly by the creation process, to allow for circular reference resolution.
            // Also remove any beans that received a temporary reference to the bean.
            destroySingleton(beanName);
            throw ex;
        }
    });
    beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}

else if (mbd.isPrototype()) {
    // It's a prototype -> create a new instance.
    Object prototypeInstance = null;
    try {
        beforePrototypeCreation(beanName);
        prototypeInstance = createBean(beanName, mbd, args);
    }
    finally {
        afterPrototypeCreation(beanName);
    }
    beanInstance = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
}

else {
    
    String scopeName = mbd.getScope();
    if (!StringUtils.hasLength(scopeName)) {
        throw new IllegalStateException("No scope name defined for bean '" + beanName + "'");
    }
    Scope scope = this.scopes.get(scopeName);
    if (scope == null) {
        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
    }
    try {
        // 也就是这里，当不是单例和原型的时候，去scope获取
        Object scopedInstance = scope.get(beanName, () -> {
            beforePrototypeCreation(beanName);
            try {
                return createBean(beanName, mbd, args);
            }
            finally {
                afterPrototypeCreation(beanName);
            }
        });
        beanInstance = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
    }
    catch (IllegalStateException ex) {
        throw new ScopeNotActiveException(beanName, scopeName, ex);
    }
}
```
scope是在RefreshScope中被注册的，因为实现了BeanFactoryPostProcessor, BeanDefinitionRegistryPostProcessor
```java
@Override
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    this.beanFactory = beanFactory;
    // 将RefreshScope注册进去，以便上面方便获取和调用Scope scope = this.scopes.get(scopeName);
    beanFactory.registerScope(this.name, this);
    setSerializationId(beanFactory);
}
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
    for (String name : registry.getBeanDefinitionNames()) {
        BeanDefinition definition = registry.getBeanDefinition(name);
        if (definition instanceof RootBeanDefinition root) {
            if (root.getDecoratedDefinition() != null && root.hasBeanClass()
                    && root.getBeanClass() == ScopedProxyFactoryBean.class) {
                if (getName().equals(root.getDecoratedDefinition().getBeanDefinition().getScope())) {
                    // 这里其实是把包扫描进去的ScopedProxyFactoryBean进一步更换为LockedScopedProxyFactoryBean，当然的是refresh情况下，为了加下锁，这里逻辑不重要
                    root.setBeanClass(LockedScopedProxyFactoryBean.class);
                    root.getConstructorArgumentValues().addGenericArgumentValue(this);
                    // surprising that a scoped proxy bean definition is not already
                    // marked as synthetic?
                    root.setSynthetic(true);
                }
            }
        }
    }
}
```
okok，终于到这里了，休息一下，我们回顾一下上面做了什么？
- 将标注了@refreshscope的bean包装为LockedScopedProxyFactoryBean，产生每次调用方法都去RefreshScope中获取bean对象
- RefreshScope是springcloud注册进去的，在springcoud-context中的自动配置类中，注册到beanfactory中
那么问题来了，为什么每次获取的时候都是最新对象呢，我们自然而然想到的是，每次刷新配置的时候，将RefreshScope保存的bean销毁，然后下次调用方法的时候就会获取到最新的bean了，这里也是这么做的，每次刷新的时候都会通知RefreshScope进行销毁，源码如下，
```java
public void refreshAll() {
    super.destroy();
    this.context.publishEvent(new RefreshScopeRefreshedEvent());
}
@Override
public void destroy() {
    List<Throwable> errors = new ArrayList<>();
    // 调用refreshAll后会清理所有缓存，所以下次获取的时候是最新的
    Collection<BeanLifecycleWrapper> wrappers = this.cache.clear();
    for (BeanLifecycleWrapper wrapper : wrappers) {
        try {
            Lock lock = this.locks.get(wrapper.getName()).writeLock();
            lock.lock();
            try {
                wrapper.destroy();
            }
            finally {
                lock.unlock();
            }
        }
        catch (RuntimeException e) {
            errors.add(e);
        }
    }
    if (!errors.isEmpty()) {
        throw wrapIfNecessary(errors.get(0));
    }
    this.errors.clear();
}
```
那么什么时候调用的呢？在ConfigDataContextRefresher中可以看到调用
```java
public synchronized Set<String> refresh() {
    Set<String> keys = refreshEnvironment();
    this.scope.refreshAll();
    return keys;
}
// springcloud将RefreshScope注册进来
@Bean
@ConditionalOnMissingBean
@ConditionalOnBootstrapDisabled
public ConfigDataContextRefresher configDataContextRefresher(ConfigurableApplicationContext context,
        RefreshScope scope, RefreshProperties properties) {
    return new ConfigDataContextRefresher(context, scope, properties);
}
```
ConfigDataContextRefresher又是谁调用的呢？在RefreshEventListener中可以看到
```java
@Override
public void onApplicationEvent(ApplicationEvent event) {
    if (event instanceof ApplicationReadyEvent) {
        handle((ApplicationReadyEvent) event);
    }
    // 接收到RefreshEvent后刷新
    else if (event instanceof RefreshEvent) {
        handle((RefreshEvent) event);
    }
}

public void handle(ApplicationReadyEvent event) {
    this.ready.compareAndSet(false, true);
}

public void handle(RefreshEvent event) {
    if (this.ready.get()) { // don't handle events before app is ready
        log.debug("Event received " + event.getEventDesc());
        Set<String> keys = this.refresh.refresh();
        log.info("Refresh keys changed: " + keys);
    }
}
//springcloud 将上面的ConfigDataContextRefresher注入进来
@Bean
public RefreshEventListener refreshEventListener(ContextRefresher contextRefresher) {
    return new RefreshEventListener(contextRefresher);
}
```
最后RefreshEvent事件是谁发布的呢？在我们引入nacos-config依赖之后，会注入一个bean，看起来有关，我们进去看看
```java
@Bean
public NacosContextRefresher nacosContextRefresher(
        NacosConfigManager nacosConfigManager,
        NacosRefreshHistory nacosRefreshHistory) {
    // Consider that it is not necessary to be compatible with the previous
    // configuration
    // and use the new configuration if necessary.
    return new NacosContextRefresher(nacosConfigManager, nacosRefreshHistory);
}

private void registerNacosListener(final String groupKey, final String dataKey) {
    String key = NacosPropertySourceRepository.getMapKey(dataKey, groupKey);
    Listener listener = listenerMap.computeIfAbsent(key,
            lst -> new AbstractSharedListener() {
                @Override
                public void innerReceive(String dataId, String group,
                        String configInfo) {
                    refreshCountIncrement();
                    nacosRefreshHistory.addRefreshRecord(dataId, group, configInfo);
                    NacosSnapshotConfigManager.putConfigSnapshot(dataId, group,
                            configInfo);
                    // 可以看到，每次配置更新的时候会发布RefreshEvent事件
                    applicationContext.publishEvent(
                            new RefreshEvent(this, null, "Refresh Nacos config"));
                    if (log.isDebugEnabled()) {
                        log.debug(String.format(
                                "Refresh Nacos config group=%s,dataId=%s,configInfo=%s",
                                group, dataId, configInfo));
                    }
                }
            });
    try {
        configService.addListener(dataKey, groupKey, listener);
        log.info("[Nacos Config] Listening config: dataId={}, group={}", dataKey,
                groupKey);
    }
    catch (NacosException e) {
        log.warn(String.format(
                "register fail for nacos listener ,dataId=[%s],group=[%s]", dataKey,
                groupKey), e);
    }
}

```
ook，终于到头了，也就是每次nacos更新配置的时候，都会发布RefreshEvent事件，然后RefreshEventListener接收事件调用ConfigDataContextRefresher中的refresh，进一步调用RefreshScope中的refresh，然后就将缓存清空了，下次获取就是最新的了

### 还有一件事，刷新过程我们看看
```java
public synchronized Set<String> refresh() {
    Set<String> keys = refreshEnvironment();
    this.scope.refreshAll();
    return keys;
}
// 这里刷新环境的时候还会发出EnvironmentChangeEvent事件，这是nacos中ConfigurationPropertiesRebinder的事件源，会从新设置所有ConfigurationPropertiesBeans
public synchronized Set<String> refreshEnvironment() {
    Map<String, Object> before = extract(this.context.getEnvironment().getPropertySources());
    updateEnvironment();
    Set<String> keys = changes(before, extract(this.context.getEnvironment().getPropertySources())).keySet();
    this.context.publishEvent(new EnvironmentChangeEvent(this.context, keys));
    return keys;
}

//也就是这里
@Bean
@ConditionalOnMissingBean(search = SearchStrategy.CURRENT)
@ConditionalOnNonDefaultBehavior
public ConfigurationPropertiesRebinder smartConfigurationPropertiesRebinder(
        ConfigurationPropertiesBeans beans) {
    // If using default behavior, not use SmartConfigurationPropertiesRebinder.
    // Minimize te possibility of making mistakes.
    return new SmartConfigurationPropertiesRebinder(beans);
}
内部为：它会收集所有的ConfigurationPropertiesBeans
private boolean rebind(String name, ApplicationContext appContext) {
    try {
        Object bean = appContext.getBean(name);
        if (AopUtils.isAopProxy(bean)) {
            bean = ProxyUtils.getTargetObject(bean);
        }
        if (bean != null) {
            // TODO: determine a more general approach to fix this.
            // see
            // https://github.com/spring-cloud/spring-cloud-commons/issues/571
            if (getNeverRefreshable().contains(bean.getClass().getName())) {
                return false; // ignore
            }
            appContext.getAutowireCapableBeanFactory().destroyBean(bean);
            appContext.getAutowireCapableBeanFactory().initializeBean(bean, name);
            return true;
        }
    }
    catch (RuntimeException e) {
        this.errors.put(name, e);
        throw e;
    }
    catch (Exception e) {
        this.errors.put(name, e);
        throw new IllegalStateException("Cannot rebind to " + name, e);
    }
    return false;
}
```
### 结尾
上面就是refreshscope自动刷新的流程了，其实还有一点，nacos是如何监听配置刷新和发布事件的呢，这里面就涉及到netty了，具体来说，nacoa会有一个定时任务去查看是否由配置的更改
```java
@Override
public void startInternal() {
    executor.schedule(() -> {
        while (!executor.isShutdown() && !executor.isTerminated()) {
            try {
                listenExecutebell.poll(5L, TimeUnit.SECONDS);
                if (executor.isShutdown() || executor.isTerminated()) {
                    continue;
                }
                //这里
                executeConfigListen();
            } catch (Throwable e) {
                LOGGER.error("[rpc listen execute] [rpc listen] exception", e);
                try {
                    Thread.sleep(50L);
                } catch (InterruptedException interruptedException) {
                    //ignore
                }
                notifyListenConfig();
            }
        }
    }, 0L, TimeUnit.MILLISECONDS);
    
}
```
netty还没学，下次再进一步看看，当然还有bootstrap中如何将远程配置拉去，以及EnvironmentPostProcessorApplicationListener中获取配置也要写写，和配置拉去也有关，因为springboot2.4？之后将bootstrap取消，提出了EnvironmentPostProcessorApplicationListener，更方便的配置导入
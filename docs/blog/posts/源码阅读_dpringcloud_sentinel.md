---
title: springcloud sentinel
date: 2024-07-07
authors: [刘耀文]
categories:
  - 源码阅读

---
# springcloud sentinel
## 1.源码解读分为三部分，初始化和运行过程以及扩展点

### 1.1 sentinel自身初始化

初始化为sentinel在springboot启动时候，做了什么？
在引入依赖后，对spring boot产生了什么副作用

<!-- more --> 
```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    <version>2023.0.1.0</version>
</dependency>
```
我们看看依赖中包含什么？

![image-20240709114212024](https://raw.githubusercontent.com/liyown/pic-go/master/blog/image-20240709114212024.png)

有很多，可以看到经典的几个部分，我们先看看本体spring-cloud-starter-alibaba-sentinel中的内容，一般看三个部分，一是自动配置类，二是SPI接口，再是springboot中的扩展点spring.factories

我们先看看有什么

![image-20240709114953153](https://raw.githubusercontent.com/liyown/pic-go/master/blog/image-20240709114953153.png)

可以看到只是导入了自动配置类，看看里面导入了什么配置类

```java
com.alibaba.cloud.sentinel.SentinelWebAutoConfiguration
com.alibaba.cloud.sentinel.SentinelWebFluxAutoConfiguration
com.alibaba.cloud.sentinel.endpoint.SentinelEndpointAutoConfiguration
com.alibaba.cloud.sentinel.custom.SentinelAutoConfiguration
com.alibaba.cloud.sentinel.feign.SentinelFeignAutoConfiguration
```

第一第二个是对springweb框架的适配，第三个是sentinel提供对外的访问端口，第三个是初始化和定制sentinel，第五个是对feign的支持，我们一个一个看，首先看sentinel自身的初始化

### 1.2 sentinel自身的初始化，属性初始化，数据源初始化，切面初始化，restTemplate初始化，这部分适配了spring项目，在没有使用mvc的情况下

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnProperty(name = "spring.cloud.sentinel.enabled", matchIfMissing = true)
@EnableConfigurationProperties(SentinelProperties.class)
public class SentinelAutoConfiguration {

	@Value("${project.name:${spring.application.name:}}")
	private String projectName;

	@Autowired
	private SentinelProperties properties;
    // 属性初始化
	@PostConstruct
	public void init() {
		if (StringUtils.isEmpty(System.getProperty(LogBase.LOG_DIR))
				&& StringUtils.isNotBlank(properties.getLog().getDir())) {
            // 日志
			System.setProperty(LogBase.LOG_DIR, properties.getLog().getDir());
		}
		if (StringUtils.isEmpty(System.getProperty(LogBase.LOG_NAME_USE_PID))
				&& properties.getLog().isSwitchPid()) {
            // 日志
			System.setProperty(LogBase.LOG_NAME_USE_PID,
					String.valueOf(properties.getLog().isSwitchPid()));
		}
		if (StringUtils.isEmpty(System.getProperty(SentinelConfig.APP_NAME_PROP_KEY))
				&& StringUtils.isNotBlank(projectName)) {
            // 设置工程名或者spring应用名字，一般是应用名${spring.application.name:}
			System.setProperty(SentinelConfig.APP_NAME_PROP_KEY, projectName);
		}
		if (StringUtils.isEmpty(System.getProperty(TransportConfig.SERVER_PORT))
				&& StringUtils.isNotBlank(properties.getTransport().getPort())) {
             // sentinel 对外的控制端口，像是dashboard就是同个这个端口进行访问的，默认为public static final String API_PORT = "8719";
			System.setProperty(TransportConfig.SERVER_PORT,
					properties.getTransport().getPort());
		}
		if (StringUtils.isEmpty(System.getProperty(TransportConfig.CONSOLE_SERVER))
				&& StringUtils.isNotBlank(properties.getTransport().getDashboard())) {
           // dashboard的端口地址	
			System.setProperty(TransportConfig.CONSOLE_SERVER,
					properties.getTransport().getDashboard());
		}
		if (StringUtils.isEmpty(System.getProperty(TransportConfig.HEARTBEAT_INTERVAL_MS))
				&& StringUtils
						.isNotBlank(properties.getTransport().getHeartbeatIntervalMs())) {
            // 心跳包的时间间隔，默认为private static final long DEFAULT_INTERVAL = 1000 * 10;
			System.setProperty(TransportConfig.HEARTBEAT_INTERVAL_MS,
					properties.getTransport().getHeartbeatIntervalMs());
		}
		if (StringUtils.isEmpty(System.getProperty(TransportConfig.HEARTBEAT_CLIENT_IP))
				&& StringUtils.isNotBlank(properties.getTransport().getClientIp())) {
            // 心跳包的客户端ip，默认为本机ip
			System.setProperty(TransportConfig.HEARTBEAT_CLIENT_IP,
					properties.getTransport().getClientIp());
		}
		if (StringUtils.isEmpty(System.getProperty(SentinelConfig.CHARSET))
				&& StringUtils.isNotBlank(properties.getMetric().getCharset())) {
			System.setProperty(SentinelConfig.CHARSET,
					properties.getMetric().getCharset());
		}
		if (StringUtils
				.isEmpty(System.getProperty(SentinelConfig.SINGLE_METRIC_FILE_SIZE))
				&& StringUtils.isNotBlank(properties.getMetric().getFileSingleSize())) {
			System.setProperty(SentinelConfig.SINGLE_METRIC_FILE_SIZE,
					properties.getMetric().getFileSingleSize());
		}
		if (StringUtils
				.isEmpty(System.getProperty(SentinelConfig.TOTAL_METRIC_FILE_COUNT))
				&& StringUtils.isNotBlank(properties.getMetric().getFileTotalCount())) {
			System.setProperty(SentinelConfig.TOTAL_METRIC_FILE_COUNT,
					properties.getMetric().getFileTotalCount());
		}
		if (StringUtils.isEmpty(System.getProperty(SentinelConfig.COLD_FACTOR))
				&& StringUtils.isNotBlank(properties.getFlow().getColdFactor())) {
			System.setProperty(SentinelConfig.COLD_FACTOR,
					properties.getFlow().getColdFactor());
		}
		if (StringUtils.isNotBlank(properties.getBlockPage())) {
			setConfig(BLOCK_PAGE_URL_CONF_KEY, properties.getBlockPage());
		}

		// earlier initialize
        //是否一开始就初始化，默认为false，而是等到第一次调用的时候初始化
		if (properties.isEager()) {
			InitExecutor.doInit();
		}

	}
	
    // 这个是支持SentinelResource注解的切面类初始化
	@Bean
	@ConditionalOnMissingBean
	public SentinelResourceAspect sentinelResourceAspect() {
		return new SentinelResourceAspect();
	}
	
    // 对SentinelRestTemplate的初始化，初始化一个后置处理器，给他添加拦截器
	@Bean
	@ConditionalOnMissingBean
	@ConditionalOnClass(name = "org.springframework.web.client.RestTemplate")
	@ConditionalOnProperty(name = "resttemplate.sentinel.enabled", havingValue = "true",
			matchIfMissing = true)
	public static SentinelBeanPostProcessor sentinelBeanPostProcessor(
			ApplicationContext applicationContext) {
		return new SentinelBeanPostProcessor(applicationContext);
	}
	// 外置属性源的处理，初始化
    // 再所有单例bean初始化后
        /*
        public void postRegister(AbstractDataSource dataSource) {
        switch (this.getRuleType()) {
            case FLOW -> FlowRuleManager.register2Property(dataSource.getProperty());
            case DEGRADE -> DegradeRuleManager.register2Property(dataSource.getProperty());
            case PARAM_FLOW -> ParamFlowRuleManager.register2Property(dataSource.getProperty());
            case SYSTEM -> SystemRuleManager.register2Property(dataSource.getProperty());
            case AUTHORITY -> AuthorityRuleManager.register2Property(dataSource.getProperty());
            case GW_FLOW -> GatewayRuleManager.register2Property(dataSource.getProperty());
            case GW_API_GROUP -> GatewayApiDefinitionManager.register2Property(dataSource.getProperty());
        }

    }
    */
	@Bean
	@ConditionalOnMissingBean
	public SentinelDataSourceHandler sentinelDataSourceHandler(
			DefaultListableBeanFactory beanFactory, SentinelProperties sentinelProperties,
			Environment env) {
		return new SentinelDataSourceHandler(beanFactory, sentinelProperties, env);
	}
	
    // 一些转换器，例如将配置外部属性源的时候设置转换器
	@ConditionalOnClass(ObjectMapper.class)
	@Configuration(proxyBeanMethods = false)
	protected static class SentinelConverterConfiguration {
		
        // json
		@Configuration(proxyBeanMethods = false)
		protected static class SentinelJsonConfiguration {

			private ObjectMapper objectMapper = new ObjectMapper();

			public SentinelJsonConfiguration() {
				objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES,
						false);
			}

			@Bean("sentinel-json-flow-converter")
			public JsonConverter jsonFlowConverter() {
				return new JsonConverter(objectMapper, FlowRule.class);
			}

			@Bean("sentinel-json-degrade-converter")
			public JsonConverter jsonDegradeConverter() {
				return new JsonConverter(objectMapper, DegradeRule.class);
			}

			@Bean("sentinel-json-system-converter")
			public JsonConverter jsonSystemConverter() {
				return new JsonConverter(objectMapper, SystemRule.class);
			}

			@Bean("sentinel-json-authority-converter")
			public JsonConverter jsonAuthorityConverter() {
				return new JsonConverter(objectMapper, AuthorityRule.class);
			}

			@Bean("sentinel-json-param-flow-converter")
			public JsonConverter jsonParamFlowConverter() {
				return new JsonConverter(objectMapper, ParamFlowRule.class);
			}

		}
		// xml
		@ConditionalOnClass(XmlMapper.class)
		@Configuration(proxyBeanMethods = false)
		protected static class SentinelXmlConfiguration {

			private XmlMapper xmlMapper = new XmlMapper();

			public SentinelXmlConfiguration() {
				xmlMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES,
						false);
			}

			@Bean("sentinel-xml-flow-converter")
			public XmlConverter xmlFlowConverter() {
				return new XmlConverter(xmlMapper, FlowRule.class);
			}

			@Bean("sentinel-xml-degrade-converter")
			public XmlConverter xmlDegradeConverter() {
				return new XmlConverter(xmlMapper, DegradeRule.class);
			}

			@Bean("sentinel-xml-system-converter")
			public XmlConverter xmlSystemConverter() {
				return new XmlConverter(xmlMapper, SystemRule.class);
			}

			@Bean("sentinel-xml-authority-converter")
			public XmlConverter xmlAuthorityConverter() {
				return new XmlConverter(xmlMapper, AuthorityRule.class);
			}

			@Bean("sentinel-xml-param-flow-converter")
			public XmlConverter xmlParamFlowConverter() {
				return new XmlConverter(xmlMapper, ParamFlowRule.class);
			}

		}

	}

}
```

### 1.3 对springMVC的适配初始化

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnProperty(name = "spring.cloud.sentinel.enabled", matchIfMissing = true)
@ConditionalOnClass(SentinelWebInterceptor.class)
@EnableConfigurationProperties(SentinelProperties.class)
public class SentinelWebAutoConfiguration implements WebMvcConfigurer {

    private static final Logger log = LoggerFactory
          .getLogger(SentinelWebAutoConfiguration.class);

    @Autowired
    private SentinelProperties properties;

    @Autowired
    private Optional<UrlCleaner> urlCleanerOptional;

    @Autowired
    private Optional<BlockExceptionHandler> blockExceptionHandlerOptional;

    @Autowired
    private Optional<RequestOriginParser> requestOriginParserOptional;

    // 这里初始化了一个拦截器，全局拦截器
    @Bean
    @ConditionalOnProperty(name = "spring.cloud.sentinel.filter.enabled",
          matchIfMissing = true)
    public SentinelWebInterceptor sentinelWebInterceptor(
          SentinelWebMvcConfig sentinelWebMvcConfig) {
       return new SentinelWebInterceptor(sentinelWebMvcConfig);
    }
	// 上面拦截器的一些配置
    @Bean
    @ConditionalOnProperty(name = "spring.cloud.sentinel.filter.enabled",
          matchIfMissing = true)
    public SentinelWebMvcConfig sentinelWebMvcConfig() {
       SentinelWebMvcConfig sentinelWebMvcConfig = new SentinelWebMvcConfig();
       // 是否将请求方法加入resource name
       sentinelWebMvcConfig.setHttpMethodSpecify(properties.getHttpMethodSpecify());
       sentinelWebMvcConfig.setWebContextUnify(properties.getWebContextUnify());
	   // 限流后的一下异常处理
       if (blockExceptionHandlerOptional.isPresent()) {
          blockExceptionHandlerOptional
                .ifPresent(sentinelWebMvcConfig::setBlockExceptionHandler);
       }
       else {
          if (StringUtils.hasText(properties.getBlockPage())) {
             sentinelWebMvcConfig.setBlockExceptionHandler(((request, response,
                   e) -> response.sendRedirect(properties.getBlockPage())));
          }
          else {
              // 限流后的一下异常处理，默认值
             sentinelWebMvcConfig
                   .setBlockExceptionHandler(new DefaultBlockExceptionHandler());
          }
       }

       urlCleanerOptional.ifPresent(sentinelWebMvcConfig::setUrlCleaner);
        // 源名字解析
       requestOriginParserOptional.ifPresent(sentinelWebMvcConfig::setOriginParser);
       return sentinelWebMvcConfig;
    }
	// 注册拦截器
    @Bean
    @ConditionalOnProperty(name = "spring.cloud.sentinel.filter.enabled",
          matchIfMissing = true)
    public SentinelWebMvcConfigurer sentinelWebMvcConfigurer() {
       return new SentinelWebMvcConfigurer();
    }

}
```

## 2 运行过程

### 2.1 springMVC拦截器运行逻辑

拦截器一般有两个方法，一个是请求前的方法，一个是请求后的方法，我们先看请求前的方法

```java
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
        throws Exception {
        try {
            // 首先获取到资源名字
            String resourceName = getResourceName(request);

            if (StringUtil.isEmpty(resourceName)) {
                return true;
            }
            // 如果requests属性中有$$sentinel_spring_web_entry_attr-rc，计数后放行
            if (increaseReferece(request, this.baseWebMvcConfig.getRequestRefName(), 1) != 1) {
                return true;
            }
            
            // Parse the request origin using registered origin parser.
            // 根据HTTP生成源，默认为空，在StatisticSlot会用到
            String origin = parseOrigin(request);
            // 获取监控容器的名字，这里默认为sentinel_spring_web_context
            String contextName = getContextName(request);
            
        	// 关键代码，初始化调用上下文
            ContextUtil.enter(contextName, origin);
            // 进入资源，进入slot插件模块，创建为
            
            
            Entry entry = SphU.entry(resourceName, ResourceTypeConstants.COMMON_WEB, EntryType.IN);
            request.setAttribute(baseWebMvcConfig.getRequestAttributeName(), entry);
            return true;
        } catch (BlockException e) {
            try {
                handleBlockException(request, response, e);
            } finally {
                ContextUtil.exit();
            }
            return false;
        }
    }
```

```java
//slot插件模块创建过程
public static ProcessorSlotChain newSlotChain() {
        if (slotChainBuilder != null) {
            return slotChainBuilder.build();
        }

        // Resolve the slot chain builder SPI.
    	//Sentinel default ProcessorSlots
        //com.alibaba.csp.sentinel.slots.nodeselector.NodeSelectorSlot
        //com.alibaba.csp.sentinel.slots.clusterbuilder.ClusterBuilderSlot
        //com.alibaba.csp.sentinel.slots.logger.LogSlot
        //com.alibaba.csp.sentinel.slots.statistic.StatisticSlot
        //com.alibaba.csp.sentinel.slots.block.authority.AuthoritySlot
        //com.alibaba.csp.sentinel.slots.system.SystemSlot
    	//com.alibaba.csp.sentinel.slots.block.flow.param.ParamFlowSlot
        //com.alibaba.csp.sentinel.slots.block.flow.FlowSlot
        //com.alibaba.csp.sentinel.slots.block.degrade.DegradeSlot
    	// 所有的默认slot，执行循序也是上面的顺序
        slotChainBuilder = SpiLoader.of(SlotChainBuilder.class).loadFirstInstanceOrDefault();

        if (slotChainBuilder == null) {
            // Should not go through here.
            RecordLog.warn("[SlotChainProvider] Wrong state when resolving slot chain builder, using default");
            slotChainBuilder = new DefaultSlotChainBuilder();
        } else {
            RecordLog.info("[SlotChainProvider] Global slot chain builder resolved: {}",
                slotChainBuilder.getClass().getCanonicalName());
        }
        return slotChainBuilder.build();
    }
```

### 2.2 进入chain.entry(context, resourceWrapper, null, count, prioritized, args)看看实际的运行过程

我们先看看slot接口，方面理解，

```java
public interface ProcessorSlot<T> {
	
	// 本slot的运行逻辑
    void entry(Context context, ResourceWrapper resourceWrapper, T param, int count, boolean prioritized,
               Object... args) throws Throwable;
	// 本slot的运行完成后，需要干什么? 抽象类的实现是将参数强转后进入下一个slot，他们的关系的单向链表
    void fireEntry(Context context, ResourceWrapper resourceWrapper, Object obj, int count, boolean prioritized,
                   Object... args) throws Throwable;
	// 同理
    void exit(Context context, ResourceWrapper resourceWrapper, int count, Object... args);

    void fireExit(Context context, ResourceWrapper resourceWrapper, int count, Object... args);
}
```

看看抽象类

```java

public abstract class AbstractLinkedProcessorSlot<T> implements ProcessorSlot<T> {
	
	// 保存下一个该执行的slot
    private AbstractLinkedProcessorSlot<?> next = null;
	
	//抽象类的实现是将参数强转后进入下一个slot，他们的关系的单向列表
    @Override
    public void fireEntry(Context context, ResourceWrapper resourceWrapper, Object obj, int count, boolean prioritized, Object... args)
        throws Throwable {
        if (next != null) {
            next.transformEntry(context, resourceWrapper, obj, count, prioritized, args);
        }
    }

    @SuppressWarnings("unchecked")
    void transformEntry(Context context, ResourceWrapper resourceWrapper, Object o, int count, boolean prioritized, Object... args)
        throws Throwable {
        T t = (T)o;
        entry(context, resourceWrapper, t, count, prioritized, args);
    }

    @Override
    public void fireExit(Context context, ResourceWrapper resourceWrapper, int count, Object... args) {
        if (next != null) {
            next.exit(context, resourceWrapper, count, args);
        }
    }

    public AbstractLinkedProcessorSlot<?> getNext() {
        return next;
    }

    public void setNext(AbstractLinkedProcessorSlot<?> next) {
        this.next = next;
    }

}
```

### ok,现在正是进入到调用slot的过程中，我们按照上面的循序一个一个看

```java
// Resolve the slot chain builder SPI.
//Sentinel default ProcessorSlots
//com.alibaba.csp.sentinel.slots.nodeselector.NodeSelectorSlot
//com.alibaba.csp.sentinel.slots.clusterbuilder.ClusterBuilderSlot
//com.alibaba.csp.sentinel.slots.logger.LogSlot
//com.alibaba.csp.sentinel.slots.statistic.StatisticSlot
//com.alibaba.csp.sentinel.slots.block.authority.AuthoritySlot
//com.alibaba.csp.sentinel.slots.system.SystemSlot
//com.alibaba.csp.sentinel.slots.block.flow.param.ParamFlowSlot
//com.alibaba.csp.sentinel.slots.block.flow.FlowSlot
//com.alibaba.csp.sentinel.slots.block.degrade.DegradeSlot
// 所有的默认slot，执行循序也是上面的顺序
```

#### NodeSelectorSlot，资源node选择器

我们必须先明确一点，node是与slotchain绑定的，每一个唯一资源都有唯一一个slotchain，而slotchain里面保存了node(defaultNode)，并且保存了不同调用上下文的不同node(defaultNode)，官网有解说

![image-20240709141403982](https://raw.githubusercontent.com/liyown/pic-go/master/blog/image-20240709141403982.png)

资源都有唯一一个slotchain；

```java
ProcessorSlot<Object> lookProcessChain(ResourceWrapper resourceWrapper) {
    ProcessorSlotChain chain = chainMap.get(resourceWrapper);
    if (chain == null) {
        synchronized (LOCK) {
            chain = chainMap.get(resourceWrapper);
            if (chain == null) {
                // Entry size limit.
                if (chainMap.size() >= Constants.MAX_SLOT_CHAIN_SIZE) {
                    return null;
                }

                chain = SlotChainProvider.newSlotChain();
                Map<ResourceWrapper, ProcessorSlotChain> newMap = new HashMap<ResourceWrapper, ProcessorSlotChain>(
                    chainMap.size() + 1);
                newMap.putAll(chainMap);
                newMap.put(resourceWrapper, chain);
                chainMap = newMap;
            }
        }
    }
    return chain;
}
```

我们看看NodeSelectorSlot是怎么实现的

```java
DefaultNode node = map.get(context.getName());
// 可以看到，以context.getName()调用上下文的名字为key去查找node，也就是说，一个资源可能有多个node，但是其实只会统计一次，在下面的ClusterBuilderSlot调用可以看到
if (node == null) {
    synchronized (this) {
        node = map.get(context.getName());
        if (node == null) {
            node = new DefaultNode(resourceWrapper, null);
            HashMap<String, DefaultNode> cacheMap = new HashMap<String, DefaultNode>(map.size());
            cacheMap.putAll(map);
            cacheMap.put(context.getName(), node);
            map = cacheMap;
            // Build invocation tree
            // 设置调用链，因为可能嵌套进入不同的资源，
            // context保存着入口node和调用链（双向链表），这里是把当前调用的node添加到上一个节点的CTNode中的child,当然如果是第一此进入，就加入入口node中
            ((DefaultNode) context.getLastNode()).addChild(node);
        }

    }
}
// 设置当前调用，设置到当前调用CTNODE中的CurNode，加入了两次，一次是加入到上级调用上下文，一次是加入当前
context.setCurNode(node);
```

#### ClusterBuilderSlot，资源统计节点的构建，也就是ClusterNode

```java
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                  boolean prioritized, Object... args)
    throws Throwable {
    //先检查当当前slotchain有没有
    if (clusterNode == null) {
        synchronized (lock) {
            if (clusterNode == null) {
                // Create the cluster node.
                clusterNode = new ClusterNode(resourceWrapper.getName(), resourceWrapper.getResourceType());
                HashMap<ResourceWrapper, ClusterNode> newMap = new HashMap<>(Math.max(clusterNodeMap.size(), 16));
                newMap.putAll(clusterNodeMap);
                newMap.put(node.getId(), clusterNode);

                clusterNodeMap = newMap;
            }
        }
    }
    //可以看到，每个slotchain只会生成一个clusterNode，共享给不同contextname的node
    node.setClusterNode(clusterNode);

    /*
     * if context origin is set, we should get or create a new {@link Node} of
     * the specific origin.
     */
    //设置源，默认没有，有过有，额外在clusterNode维护一个map

    if (!"".equals(context.getOrigin())) {
        Node originNode = node.getClusterNode().getOrCreateOriginNode(context.getOrigin());
        context.getCurEntry().setOriginNode(originNode);
    }

    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```

ok，这里逻辑很简单，创建一个clusterNode统计进入信息

#### 接着进入LogSlot

这里面逻辑也比较简单，记录BlockException

```java
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode obj, int count, boolean prioritized, Object... args)
    throws Throwable {
    try {
        fireEntry(context, resourceWrapper, obj, count, prioritized, args);
    } catch (BlockException e) {
        EagleEyeLogUtil.log(resourceWrapper.getName(), e.getClass().getSimpleName(), e.getRuleLimitApp(),
            context.getOrigin(), e.getRule().getId(), count);
        throw e;
    } catch (Throwable e) {
        RecordLog.warn("Unexpected entry exception", e);
    }

}
```

#### ok ，到了核心的StatisticSlot

 首先判断是否可以进入，这里先去执行AuthoritySlot、SystemSlot，ParamFlowSlot，FlowSlot，所以我们先去看看其他几个slot的逻辑，先跳过这一节

```java
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                  boolean prioritized, Object... args) throws Throwable {
    try {
        // 首先判断是否可以进入，这里先去执行AuthoritySlot、SystemSlot，ParamFlowSlot，FlowSlot
        fireEntry(context, resourceWrapper, node, count, prioritized, args);

        // Request passed, add thread count and pass count.
        node.increaseThreadNum();
        node.addPassRequest(count);

        if (context.getCurEntry().getOriginNode() != null) {
            // Add count for origin node.
            context.getCurEntry().getOriginNode().increaseThreadNum();
            context.getCurEntry().getOriginNode().addPassRequest(count);
        }

        if (resourceWrapper.getEntryType() == EntryType.IN) {
            // Add count for global inbound entry node for global statistics.
            Constants.ENTRY_NODE.increaseThreadNum();
            Constants.ENTRY_NODE.addPassRequest(count);
        }

        // Handle pass event with registered entry callback handlers.
        for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
            handler.onPass(context, resourceWrapper, node, count, args);
        }
    } catch (PriorityWaitException ex) {
        node.increaseThreadNum();
        if (context.getCurEntry().getOriginNode() != null) {
            // Add count for origin node.
            context.getCurEntry().getOriginNode().increaseThreadNum();
        }

        if (resourceWrapper.getEntryType() == EntryType.IN) {
            // Add count for global inbound entry node for global statistics.
            Constants.ENTRY_NODE.increaseThreadNum();
        }
        // Handle pass event with registered entry callback handlers.
        for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
            handler.onPass(context, resourceWrapper, node, count, args);
        }
    } catch (BlockException e) {
        // Blocked, set block exception to current entry.
        context.getCurEntry().setBlockError(e);

        // Add block count.
        node.increaseBlockQps(count);
        if (context.getCurEntry().getOriginNode() != null) {
            context.getCurEntry().getOriginNode().increaseBlockQps(count);
        }

        if (resourceWrapper.getEntryType() == EntryType.IN) {
            // Add count for global inbound entry node for global statistics.
            Constants.ENTRY_NODE.increaseBlockQps(count);
        }

        // Handle block event with registered entry callback handlers.
        for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
            handler.onBlocked(e, context, resourceWrapper, node, count, args);
        }

        throw e;
    } catch (Throwable e) {
        // Unexpected internal error, set error to current entry.
        context.getCurEntry().setError(e);

        throw e;
    }
}
```

#### AuthoritySlot

```java
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count, boolean prioritized, Object... args)
    throws Throwable {
    // 检查鉴权信息
    checkBlackWhiteAuthority(resourceWrapper, context);
    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
// 就是去拿到实现设置好的鉴权规则看看是否可以通过
void checkBlackWhiteAuthority(ResourceWrapper resource, Context context) throws AuthorityException {
    Map<String, Set<AuthorityRule>> authorityRules = AuthorityRuleManager.getAuthorityRules();

    if (authorityRules == null) {
        return;
    }

    Set<AuthorityRule> rules = authorityRules.get(resource.getName());
    if (rules == null) {
        return;
    }

    for (AuthorityRule rule : rules) {
        if (!AuthorityRuleChecker.passCheck(rule, context)) {
            throw new AuthorityException(context.getOrigin(), rule);
        }
    }
}
```

#### SystemSlot

```java
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                  boolean prioritized, Object... args) throws Throwable {
    // 检查系统健康状态
    // 这里就不进去看了，具体就是根据系统的指标调整通过的请求，官网解释https://sentinelguard.io/zh-cn/docs/system-adaptive-protection.html
    SystemRuleManager.checkSystem(resourceWrapper, count);
    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```

#### ParamFlowSlot

```java
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                  boolean prioritized, Object... args) throws Throwable {
    // 查看是否有规则
    if (!ParamFlowRuleManager.hasRules(resourceWrapper.getName())) {
        fireEntry(context, resourceWrapper, node, count, prioritized, args);
        return;
    }
	// 根据参数的位置，限制访问https://sentinelguard.io/zh-cn/docs/parameter-flow-control.html
    checkFlow(resourceWrapper, count, args);
    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```

#### FlowSlot

与ParamFlowSlot类似，根据FlowRule决定是否通过[flow-control | Sentinel (sentinelguard.io)](https://sentinelguard.io/zh-cn/docs/flow-control.html)

```
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                  boolean prioritized, Object... args) throws Throwable {
    checkFlow(resourceWrapper, context, node, count, prioritized);

    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```

#### DegradeSlot

- 慢调用比例 (`SLOW_REQUEST_RATIO`)：选择以慢调用比例作为阈值，需要设置允许的慢调用 RT（即最大的响应时间），请求的响应时间大于该值则统计为慢调用。当单位统计时长（`statIntervalMs`）内请求数目大于设置的最小请求数目，并且慢调用的比例大于阈值，则接下来的熔断时长内请求会自动被熔断。经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求响应时间小于设置的慢调用 RT 则结束熔断，若大于设置的慢调用 RT 则会再次被熔断。
- 异常比例 (`ERROR_RATIO`)：当单位统计时长（`statIntervalMs`）内请求数目大于设置的最小请求数目，并且异常的比例大于阈值，则接下来的熔断时长内请求会自动被熔断。经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求成功完成（没有错误）则结束熔断，否则会再次被熔断。异常比率的阈值范围是 `[0.0, 1.0]`，代表 0% - 100%。
- 异常数 (`ERROR_COUNT`)：当单位统计时长内的异常数目超过阈值之后会自动进行熔断。经过熔断时长后熔断器会进入探测恢复状态（HALF-OPEN 状态），若接下来的一个请求成功完成（没有错误）则结束熔断，否则会再次被熔断。
- [circuit-breaking | Sentinel (sentinelguard.io)](https://sentinelguard.io/zh-cn/docs/circuit-breaking.html)

```java
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                  boolean prioritized, Object... args) throws Throwable {
    performChecking(context, resourceWrapper);

    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```

如果前面的都通过了然后回到StatisticSlot

```java
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node, int count,
                  boolean prioritized, Object... args) throws Throwable {
    try {
        // Do some checking.
        fireEntry(context, resourceWrapper, node, count, prioritized, args);

        // 现在执行到这里了
        // 增加线程数，里面会有两种，一种是瞬时的，一种是集群的
        node.increaseThreadNum();
        // 增加访问次数
        node.addPassRequest(count);
		// 增加源访问次数
        if (context.getCurEntry().getOriginNode() != null) {
            // Add count for origin node.
            context.getCurEntry().getOriginNode().increaseThreadNum();
            context.getCurEntry().getOriginNode().addPassRequest(count);
        }
		
        // 统计入口node的访问次数和线程
        if (resourceWrapper.getEntryType() == EntryType.IN) {
            // Add count for global inbound entry node for global statistics.
            Constants.ENTRY_NODE.increaseThreadNum();
            Constants.ENTRY_NODE.addPassRequest(count);
        }
		
        // 通过了限流，的回调函数
        for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
            handler.onPass(context, resourceWrapper, node, count, args);
        }
    } catch (PriorityWaitException ex) {
        // 这里表示默认处理时，等待一会后可以进去，还是放行
        node.increaseThreadNum();
        if (context.getCurEntry().getOriginNode() != null) {
            // Add count for origin node.
            context.getCurEntry().getOriginNode().increaseThreadNum();
        }

        if (resourceWrapper.getEntryType() == EntryType.IN) {
            // 一样统计入口node的访问次数和线程
            Constants.ENTRY_NODE.increaseThreadNum();
        }
        // 通过了限流，的回调函数
        for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
            handler.onPass(context, resourceWrapper, node, count, args);
        }
    } catch (BlockException e) {
        // Blocked, set block exception to current entry.
        context.getCurEntry().setBlockError(e);

        // Add block count.
        node.increaseBlockQps(count);
        if (context.getCurEntry().getOriginNode() != null) {
            context.getCurEntry().getOriginNode().increaseBlockQps(count);
        }

        if (resourceWrapper.getEntryType() == EntryType.IN) {
            // Add count for global inbound entry node for global statistics.
            Constants.ENTRY_NODE.increaseBlockQps(count);
        }

        // 没有通过了限流，的回调函数
        for (ProcessorSlotEntryCallback<DefaultNode> handler : StatisticSlotCallbackRegistry.getEntryCallbacks()) {
            handler.onBlocked(e, context, resourceWrapper, node, count, args);
        }

        throw e;
    } catch (Throwable e) {
        // Unexpected internal error, set error to current entry.
        context.getCurEntry().setError(e);

        throw e;
    }
}
```

## 3. 扩展点

sentinel有很多扩展点

### 3.1 初始化过程扩展Initexector

执行时机，第一次调用enty，或者不是Earler模式

```java
public static void doInit() {
    if (!initialized.compareAndSet(false, true)) {
        return;
    }
    try {
        // 找到所有的InitFunc
        List<InitFunc> initFuncs = SpiLoader.of(InitFunc.class).loadInstanceListSorted();
        List<OrderWrapper> initList = new ArrayList<OrderWrapper>();
        for (InitFunc initFunc : initFuncs) {
            RecordLog.info("[InitExecutor] Found init func: {}", initFunc.getClass().getCanonicalName());
            insertSorted(initList, initFunc);
        }
        for (OrderWrapper w : initList) {
            w.func.init();
            RecordLog.info("[InitExecutor] Executing {} with order {}",
                w.func.getClass().getCanonicalName(), w.order);
        }
    } catch (Exception ex) {
        RecordLog.warn("[InitExecutor] WARN: Initialization failed", ex);
        ex.printStackTrace();
    } catch (Error error) {
        RecordLog.warn("[InitExecutor] ERROR: Initialization failed with fatal error", error);
        error.printStackTrace();
    }
}
```

默认情况下，会找到集群模式下的client和server初始化以及metric回调函数初始化

```java
public class MetricCallbackInit implements InitFunc {
    @Override
    public void init() throws Exception {
        StatisticSlotCallbackRegistry.addEntryCallback(MetricEntryCallback.class.getCanonicalName(),
            new MetricEntryCallback());
        StatisticSlotCallbackRegistry.addExitCallback(MetricExitCallback.class.getCanonicalName(),
            new MetricExitCallback());
    }
}
```

### 3.2 Slot/Slot Chain扩展

调用时机：没有找到资源匹配的slotchain

```java
public final class SlotChainProvider {

    private static volatile SlotChainBuilder slotChainBuilder = null;

    /**
     * The load and pick process is not thread-safe, but it's okay since the method should be only invoked
     * via {@code lookProcessChain} in {@link com.alibaba.csp.sentinel.CtSph} under lock.
     *
     * @return new created slot chain
     */
    public static ProcessorSlotChain newSlotChain() {
        if (slotChainBuilder != null) {
            return slotChainBuilder.build();
        }

        // Resolve the slot chain builder SPI.
        slotChainBuilder = SpiLoader.of(SlotChainBuilder.class).loadFirstInstanceOrDefault();

        if (slotChainBuilder == null) {
            // Should not go through here.
            RecordLog.warn("[SlotChainProvider] Wrong state when resolving slot chain builder, using default");
            slotChainBuilder = new DefaultSlotChainBuilder();
        } else {
            RecordLog.info("[SlotChainProvider] Global slot chain builder resolved: {}",
                slotChainBuilder.getClass().getCanonicalName());
        }
        return slotChainBuilder.build();
    }

    private SlotChainProvider() {}
}
```

### 3.3 Transport 扩展

其实就是客户端对外暴露的接口，默认也会暴露一些接口，方面查看客户端的情况

首先有一个API中心负责接收外界信息

```java

public class SimpleHttpCommandCenter implements CommandCenter {

    private static final int PORT_UNINITIALIZED = -1;

    private static final int DEFAULT_SERVER_SO_TIMEOUT = 3000;
    private static final int DEFAULT_PORT = 8719;

    @SuppressWarnings("rawtypes")
    private static final Map<String, CommandHandler> handlerMap = new ConcurrentHashMap<String, CommandHandler>();

    @SuppressWarnings("PMD.ThreadPoolCreationRule")
    private ExecutorService executor = Executors.newSingleThreadExecutor(
        new NamedThreadFactory("sentinel-command-center-executor", true));
    private ExecutorService bizExecutor;

    private ServerSocket socketReference;

    @Override
    @SuppressWarnings("rawtypes")
    public void beforeStart() throws Exception {
        // Register handlers
        Map<String, CommandHandler> handlers = CommandHandlerProvider.getInstance().namedHandlers();
        registerCommands(handlers);
    }

    @Override
    public void start() throws Exception {
        int nThreads = Runtime.getRuntime().availableProcessors();
        this.bizExecutor = new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS,
            new ArrayBlockingQueue<Runnable>(10),
            new NamedThreadFactory("sentinel-command-center-service-executor", true),
            new RejectedExecutionHandler() {
                @Override
                public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
                    CommandCenterLog.info("EventTask rejected");
                    throw new RejectedExecutionException();
                }
            });

        Runnable serverInitTask = new Runnable() {
            int port;

            {
                try {
                    port = Integer.parseInt(TransportConfig.getPort());
                } catch (Exception e) {
                    port = DEFAULT_PORT;
                }
            }

            @Override
            public void run() {
                boolean success = false;
                ServerSocket serverSocket = getServerSocketFromBasePort(port);

                if (serverSocket != null) {
                    CommandCenterLog.info("[CommandCenter] Begin listening at port " + serverSocket.getLocalPort());
                    socketReference = serverSocket;
                    executor.submit(new ServerThread(serverSocket));
                    success = true;
                    port = serverSocket.getLocalPort();
                } else {
                    CommandCenterLog.info("[CommandCenter] chooses port fail, http command center will not work");
                }

                if (!success) {
                    port = PORT_UNINITIALIZED;
                }

                TransportConfig.setRuntimePort(port);
                executor.shutdown();
            }

        };

        new Thread(serverInitTask).start();
    }

    /**
     * Get a server socket from an available port from a base port.<br>
     * Increasing on port number will occur when the port has already been used.
     *
     * @param basePort base port to start
     * @return new socket with available port
     */
    private static ServerSocket getServerSocketFromBasePort(int basePort) {
        int tryCount = 0;
        while (true) {
            try {
                ServerSocket server = new ServerSocket(basePort + tryCount / 3, 100);
                server.setReuseAddress(true);
                return server;
            } catch (IOException e) {
                tryCount++;
                try {
                    TimeUnit.MILLISECONDS.sleep(30);
                } catch (InterruptedException e1) {
                    break;
                }
            }
        }
        return null;
    }

    @Override
    public void stop() throws Exception {
        if (socketReference != null) {
            try {
                socketReference.close();
            } catch (IOException e) {
                CommandCenterLog.warn("Error when releasing the server socket", e);
            }
        }

        if (bizExecutor != null) {
            bizExecutor.shutdownNow();
        }
        executor.shutdownNow();
        TransportConfig.setRuntimePort(PORT_UNINITIALIZED);
        handlerMap.clear();
    }

    /**
     * Get the name set of all registered commands.
     */
    public static Set<String> getCommands() {
        return handlerMap.keySet();
    }

    class ServerThread extends Thread {

        private ServerSocket serverSocket;

        ServerThread(ServerSocket s) {
            this.serverSocket = s;
            setName("sentinel-courier-server-accept-thread");
        }

        @Override
        public void run() {
            while (true) {
                Socket socket = null;
                try {
                    socket = this.serverSocket.accept();
                    setSocketSoTimeout(socket);
                    HttpEventTask eventTask = new HttpEventTask(socket);
                    bizExecutor.submit(eventTask);
                } catch (Exception e) {
                    CommandCenterLog.info("Server error", e);
                    if (socket != null) {
                        try {
                            socket.close();
                        } catch (Exception e1) {
                            CommandCenterLog.info("Error when closing an opened socket", e1);
                        }
                    }
                    try {
                        // In case of infinite log.
                        Thread.sleep(10);
                    } catch (InterruptedException e1) {
                        // Indicates the task should stop.
                        break;
                    }
                }
            }
        }
    }

}
```

接收到之后找到适配的handler

```java
public void run() {
    if (socket == null) {
        return;
    }

    PrintWriter printWriter = null;
    InputStream inputStream = null;
    try {
        long start = System.currentTimeMillis();
        inputStream = new BufferedInputStream(socket.getInputStream());
        OutputStream outputStream = socket.getOutputStream();

        printWriter = new PrintWriter(
            new OutputStreamWriter(outputStream, Charset.forName(SentinelConfig.charset())));

        String firstLine = readLine(inputStream);
        CommandCenterLog.info("[SimpleHttpCommandCenter] Socket income: " + firstLine
            + ", addr: " + socket.getInetAddress());
        CommandRequest request = processQueryString(firstLine);

        if (firstLine.length() > 4 && StringUtil.equalsIgnoreCase("POST", firstLine.substring(0, 4))) {
            // Deal with post method
            processPostRequest(inputStream, request);
        }

        // Validate the target command.
        String commandName = HttpCommandUtils.getTarget(request);
        if (StringUtil.isBlank(commandName)) {
            writeResponse(printWriter, StatusCode.BAD_REQUEST, INVALID_COMMAND_MESSAGE);
            return;
        }

        // Find the matching command handler.
        CommandHandler<?> commandHandler = SimpleHttpCommandCenter.getHandler(commandName);
        if (commandHandler != null) {
            CommandResponse<?> response = commandHandler.handle(request);
            handleResponse(response, printWriter);
        } else {
            // No matching command handler.
            writeResponse(printWriter, StatusCode.BAD_REQUEST, "Unknown command `" + commandName + '`');
        }

        long cost = System.currentTimeMillis() - start;
        CommandCenterLog.info("[SimpleHttpCommandCenter] Deal a socket task: " + firstLine
            + ", address: " + socket.getInetAddress() + ", time cost: " + cost + " ms");
    } catch (RequestException e) {
        writeResponse(printWriter, e.getStatusCode(), e.getMessage());
    } catch (Throwable e) {
        CommandCenterLog.warn("[SimpleHttpCommandCenter] CommandCenter error", e);
        try {
            if (printWriter != null) {
                String errorMessage = SERVER_ERROR_MESSAGE;
                e.printStackTrace();
                if (!writtenHead) {
                    writeResponse(printWriter, StatusCode.INTERNAL_SERVER_ERROR, errorMessage);
                } else {
                    printWriter.println(errorMessage);
                }
                printWriter.flush();
            }
        } catch (Exception e1) {
            CommandCenterLog.warn("Failed to write error response", e1);
        }
    } finally {
        closeResource(inputStream);
        closeResource(printWriter);
        closeResource(socket);
    }
}
```

默认会定义一些handle

```java
com.alibaba.csp.sentinel.cluster.server.command.handler.ModifyClusterServerFlowConfigHandler
com.alibaba.csp.sentinel.cluster.server.command.handler.FetchClusterFlowRulesCommandHandler
com.alibaba.csp.sentinel.cluster.server.command.handler.FetchClusterParamFlowRulesCommandHandler
com.alibaba.csp.sentinel.cluster.server.command.handler.FetchClusterServerConfigHandler
com.alibaba.csp.sentinel.cluster.server.command.handler.ModifyClusterServerTransportConfigHandler
com.alibaba.csp.sentinel.cluster.server.command.handler.ModifyServerNamespaceSetHandler
com.alibaba.csp.sentinel.cluster.server.command.handler.ModifyClusterFlowRulesCommandHandler
com.alibaba.csp.sentinel.cluster.server.command.handler.ModifyClusterParamFlowRulesCommandHandler
com.alibaba.csp.sentinel.cluster.server.command.handler.FetchClusterServerInfoCommandHandler
com.alibaba.csp.sentinel.cluster.server.command.handler.FetchClusterMetricCommandHandler

com.alibaba.csp.sentinel.command.handler.GetParamFlowRulesCommandHandler
com.alibaba.csp.sentinel.command.handler.ModifyParamFlowRulesCommandHandler

com.alibaba.csp.sentinel.command.handler.ModifyClusterClientConfigHandler
com.alibaba.csp.sentinel.command.handler.FetchClusterClientConfigHandler
```

### 3.4 集群流控扩展

[集群流控 · alibaba/Sentinel Wiki (github.com)](https://github.com/alibaba/Sentinel/wiki/集群流控#扩展接口设计)

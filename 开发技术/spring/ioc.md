版本：5.3.27



ioc的实现：ioc容器



容器对象：

1. AnnotationConfigApplicationContext：基于注解配置管理bean。
2. ClassPathXmlApplicationContext：基于xml文件配置管理bean。



AnnotationConfigApplicationContext

源码解析

1. 下列代码为容器创建方式之一，以此为例分析。

   ```java
   AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(Tyre.class);
   ```

2. AnnotationConfigApplicationContext是一个上下文，继承关系如下：

   ```java
   public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry {
   ```

   [继承图](../../img/AnnotationConfigApplicationContext继承图.png)

   它是BeanFactory、AnnotationConfigRegistry的实现，即它既是一个bean工厂也是一个注册表。注册表提供注册和扫描功能。

   

   它有两个域reader、scanner。scanner能够扫描类路径，识别bean定义，并将其注册。scanner能够识别@Component、@Repository、@Service、@Controller、@ManagedBean（Java EE 6's）、@Named（JSR-330's）注解。reader是scanner的平替，能解析相同的注解，但只能用作显式的注册。

   ```java
   private final AnnotatedBeanDefinitionReader reader;
   
   private final ClassPathBeanDefinitionScanner scanner;
   ```

3. 从1进入AnnotationConfigApplicationContext。

   ```java
   public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
   	this();
       // 注册
   	register(componentClasses);
       // 刷新
   	refresh();
   }
   ```

4. 进入this。该方法创建了reader和scanner实例。

   ```java
   public AnnotationConfigApplicationContext() {
   	StartupStep createAnnotatedBeanDefReader = this.getApplicationStartup().start("spring.context.annotated-bean-reader.create");
   	// 将上下文作为注册表传入
   	this.reader = new AnnotatedBeanDefinitionReader(this);
   	createAnnotatedBeanDefReader.end();
   	// 将上下文作为注册表传入
   	this.scanner = new ClassPathBeanDefinitionScanner(this);
   }
   ```

5. 进入AnnotatedBeanDefinitionReader。该方法将传入的上下文指定为了注册表。它的核心代码是最后一行代码，其作用是注册注解相关的所有后置处理器。

   ```java
   public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
   	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
   	Assert.notNull(environment, "Environment must not be null");
   	// 指定注册表
   	this.registry = registry;
   
       this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
   	// 注册注解相关的后置处理器
   	AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
   }
   ```

6. 进入registerAnnotationConfigProcessors。

   ```java
   public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
   		BeanDefinitionRegistry registry, @Nullable Object source) {
   
   	// 将注册表解析为DefaultListableBeanFactory
   	DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
   	if (beanFactory != null) {
   		// 设置排序规则 (关联: Ordered接口以及@Order, @Priority注解)
   		if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
   			beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
   		}
   		// 设置自动装配候选人的解析器, 其作用是判断bean定义是否能够作为自动装配的候选人.
   		if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
   			beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
   		}
   	}
   
   	// bean定义集合
   	Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);
   
   	// ------- 注册处理器 -------
   
   	// @Configuration处理器
   	if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
   		RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
   		def.setSource(source);
   		// 注册处理器, 后面的同此.
   		beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
   	}
   
   	// @Autowired处理器
   	if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
   		RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
   		def.setSource(source);
   		beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
   	}
   
   	// 对于JSR-250注解的支持
   	// Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
   	if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
   		RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
   		def.setSource(source);
   		beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
   	}
   
   	// 对于JPA注解的支持
   	// Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
   	if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
   		RootBeanDefinition def = new RootBeanDefinition();
   		try {
   			def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
   					AnnotationConfigUtils.class.getClassLoader()));
   		}
   		catch (ClassNotFoundException ex) {
   			throw new IllegalStateException(
   					"Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
   		}
   		def.setSource(source);
   		beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
   	}
   
   	// @EventListener处理器
   	if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
   		RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
   		def.setSource(source);
   		beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
   	}
   
   	// 事件监听工厂
   	if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
   		RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
   		def.setSource(source);
   		beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
   	}
   
   	return beanDefs;
   }
   ```

   

   进入unwrapDefaultListableBeanFactory。上下文继承的是GenericApplicationContext，所以可知实际使用的bean工厂是GenericApplicationContext中的beanFactory。

   ```java
   private static DefaultListableBeanFactory unwrapDefaultListableBeanFactory(BeanDefinitionRegistry registry) {
   	if (registry instanceof DefaultListableBeanFactory) {
   		return (DefaultListableBeanFactory) registry;
   	}
       // 实际执行位置
   	else if (registry instanceof GenericApplicationContext) {
   		return ((GenericApplicationContext) registry).getDefaultListableBeanFactory();
   	}
   	else {
   		return null;
   	}
   }
   ```

   ```java
   // GenericApplicationContext
   private final DefaultListableBeanFactory beanFactory;
   ```

   

   进入registerPostProcessor。可以看到处理器实际是注册到了bean工厂的beanDefinitionMap中。

   ```java
   private static BeanDefinitionHolder registerPostProcessor(
   		BeanDefinitionRegistry registry, RootBeanDefinition definition, String beanName) {
   
   	definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
   	registry.registerBeanDefinition(beanName, definition);
   	return new BeanDefinitionHolder(definition, beanName);
   }
   ```

   ```java
   @Override
   public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
   		throws BeanDefinitionStoreException {
   
   	this.beanFactory.registerBeanDefinition(beanName, beanDefinition);
   }
   ```

   ```java
   @Override
   public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
   		throws BeanDefinitionStoreException {
   
   	Assert.hasText(beanName, "Bean name must not be empty");
   	Assert.notNull(beanDefinition, "BeanDefinition must not be null");
   
   	if (beanDefinition instanceof AbstractBeanDefinition) {
   		try {
   			((AbstractBeanDefinition) beanDefinition).validate();
   		}
   		catch (BeanDefinitionValidationException ex) {
   			throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
   					"Validation of bean definition failed", ex);
   		}
   	}
   
   	BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
   	if (existingDefinition != null) {
   		if (!isAllowBeanDefinitionOverriding()) {
   			throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
   		}
   		else if (existingDefinition.getRole() < beanDefinition.getRole()) {
   			// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
   			if (logger.isInfoEnabled()) {
   				logger.info("Overriding user-defined bean definition for bean '" + beanName +
   						"' with a framework-generated bean definition: replacing [" +
   						existingDefinition + "] with [" + beanDefinition + "]");
   			}
   		}
   		else if (!beanDefinition.equals(existingDefinition)) {
   			if (logger.isDebugEnabled()) {
   				logger.debug("Overriding bean definition for bean '" + beanName +
   						"' with a different definition: replacing [" + existingDefinition +
   						"] with [" + beanDefinition + "]");
   			}
   		}
   		else {
   			if (logger.isTraceEnabled()) {
   				logger.trace("Overriding bean definition for bean '" + beanName +
   						"' with an equivalent definition: replacing [" + existingDefinition +
   						"] with [" + beanDefinition + "]");
   			}
   		}
   		this.beanDefinitionMap.put(beanName, beanDefinition);
   	}
   	else {
   		if (hasBeanCreationStarted()) {
   			// Cannot modify startup-time collection elements anymore (for stable iteration)
   			synchronized (this.beanDefinitionMap) {
   				this.beanDefinitionMap.put(beanName, beanDefinition);
   				List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
   				updatedDefinitions.addAll(this.beanDefinitionNames);
   				updatedDefinitions.add(beanName);
   				this.beanDefinitionNames = updatedDefinitions;
   				removeManualSingletonName(beanName);
   			}
   		}
   		else {
   			// Still in startup registration phase
   			this.beanDefinitionMap.put(beanName, beanDefinition);
   			this.beanDefinitionNames.add(beanName);
   			removeManualSingletonName(beanName);
   		}
   		this.frozenBeanDefinitionNames = null;
   	}
   
   	if (existingDefinition != null || containsSingleton(beanName)) {
   		resetBeanDefinition(beanName);
   	}
   	else if (isConfigurationFrozen()) {
   		clearByTypeCache();
   	}
   }
   ```

   ```java
   // DefaultListableBeanFactory
   /** Map of bean definition objects, keyed by bean name. */
   private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
   ```

7. 从4进入ClassPathBeanDefinitionScanner。

   ```java
   public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
   		Environment environment, @Nullable ResourceLoader resourceLoader) {
   
   	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
   	// 指定注册表
   	this.registry = registry;
   
   	// 注册默认拦截器, 用于拦截注解.
   	if (useDefaultFilters) {
   		registerDefaultFilters();
   	}
   	// 设置环境变量: 配置, 属性
   	setEnvironment(environment);
   	// 设置资源加载器, 用于将给定的配置文件路径解析为资源对象.
   	setResourceLoader(resourceLoader);
   }
   ```

8. 从3进入register。该方法显式将目标类解析为bean定义并注册。

   ```java
   public void register(Class<?>... componentClasses) {
   	Assert.notEmpty(componentClasses, "At least one component class must be specified");
   	StartupStep registerComponentClass = this.getApplicationStartup().start("spring.context.component-classes.register")
   			.tag("classes", () -> Arrays.toString(componentClasses));
   	// 使用reader进行注册
   	this.reader.register(componentClasses);
   	registerComponentClass.end();
   }
   ```

9. 进入register。

   ```java
   private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
   		@Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
   		@Nullable BeanDefinitionCustomizer[] customizers) {
   
   	// 将目标类解析为bean定义, 期间会从类的注解中获取元数据.
   	AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
   	if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
   		return;
   	}
   
   	// 设置创建bean实例的回调
   	abd.setInstanceSupplier(supplier);
   	// 设置作用域
   	ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
   	abd.setScope(scopeMetadata.getScopeName());
   
   	String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));
   
   	AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
   	// 处理限定类注解
   	if (qualifiers != null) {
   		for (Class<? extends Annotation> qualifier : qualifiers) {
   			if (Primary.class == qualifier) {
   				abd.setPrimary(true);
   			}
   			else if (Lazy.class == qualifier) {
   				abd.setLazyInit(true);
   			}
   			else {
   				abd.addQualifier(new AutowireCandidateQualifier(qualifier));
   			}
   		}
   	}
   	// 设置bean定义的回调
   	if (customizers != null) {
   		for (BeanDefinitionCustomizer customizer : customizers) {
   			customizer.customize(abd);
   		}
   	}
   
   	BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
   	// 设置作用域代理
   	definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
   	// 注册bean定义, 注册到了beanDefinitionMap中.
   	BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
   }
   ```

10. 从3进入refresh。该方法加载或刷新配置的持久化表示，简单说就是执行bean的实例化并刷新上下文。该方法是启动方法，如果失败，则需要销毁所有已经创建的单例bean，即单例bean要么全部实例化，要么全部都没实例化。

   ```java
   public void refresh() throws BeansException, IllegalStateException {
   	synchronized (this.startupShutdownMonitor) {
   		StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");
   
   		// 使上下文做好刷新准备
   		// Prepare this context for refreshing.
   		prepareRefresh();
   
   		// 通知子类刷新内置bean工厂
   		// Tell the subclass to refresh the internal bean factory.
   		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
   
   		// 使bean工厂做好在上下文中使用的准备
   		// Prepare the bean factory for use in this context.
   		prepareBeanFactory(beanFactory);
   
   		try {
   			// 允许在上下文子类中对bean工厂做后置处理
   			// Allows post-processing of the bean factory in context subclasses.
   			postProcessBeanFactory(beanFactory);
   
   			StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
   			// 调用在上下文中被作为bean注册的工厂处理器
   			// Invoke factory processors registered as beans in the context.
   			invokeBeanFactoryPostProcessors(beanFactory);
   
   			// 注册拦截bean创建的bean处理器
   			// Register bean processors that intercept bean creation.
   			registerBeanPostProcessors(beanFactory);
   			beanPostProcess.end();
   
   			// 初始化上下文消息源
   			// Initialize message source for this context.
   			initMessageSource();
   
   			// 初始化事件组播器
   			// Initialize event multicaster for this context.
   			initApplicationEventMulticaster();
   
   			// 在特定上下文子类中初始化其他特殊的bean
   			// Initialize other special beans in specific context subclasses.
   			onRefresh();
   
   			// 检查监听器bean并注册它们
   			// Check for listener beans and register them.
   			registerListeners();
   
   			// 实例化所有剩下的非懒加载的单例bean
   			// Instantiate all remaining (non-lazy-init) singletons.
   			finishBeanFactoryInitialization(beanFactory);
   
   			// 发布相应事件
   			// Last step: publish corresponding event.
   			finishRefresh();
   		}
   
   		catch (BeansException ex) {
   			if (logger.isWarnEnabled()) {
   				logger.warn("Exception encountered during context initialization - " +
   						"cancelling refresh attempt: " + ex);
   			}
   
   			// 销毁已经创建的单例bean, 避免悬挂资源.
   			// Destroy already created singletons to avoid dangling resources.
   			destroyBeans();
   
   			// 重置“激活”标志
   			// Reset 'active' flag.
   			cancelRefresh(ex);
   
   			// Propagate exception to caller.
   			throw ex;
   		}
   
   		finally {
   			// Reset common introspection caches in Spring's core, since we
   			// might not ever need metadata for singleton beans anymore...
   			resetCommonCaches();
   			contextRefresh.end();
   		}
   	}
   }
   ```

11. 返回1，构造函数传入路径，这是另一种容器创建方式。

    ```java
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext("com.test");
    ```

12. 进入AnnotationConfigApplicationContext。与之前相似，只是使用了不一样的注册方法。

    ```java
    public AnnotationConfigApplicationContext(String... basePackages) {
    	this();
        // 扫描
    	scan(basePackages);
    	refresh();
    }
    ```

13. 进入scan。该方法使用scanner处理目标路径。

    ```java
    public void scan(String... basePackages) {
    	Assert.notEmpty(basePackages, "At least one base package must be specified");
    	StartupStep scanPackages = this.getApplicationStartup().start("spring.context.base-packages.scan")
    			.tag("packages", () -> Arrays.toString(basePackages));
    	// 扫描
        this.scanner.scan(basePackages);
    	scanPackages.end();
    }
    ```

14. 进入scan。

    ```java
    public int scan(String... basePackages) {
    	int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
        
        // 扫描
    	doScan(basePackages);
    
        // 注册注解相关的后置处理器
    	// Register annotation config processors, if necessary.
    	if (this.includeAnnotationConfig) {
    		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    	}
    
    	return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
    }
    ```

15. 进入doScan。

    ```java
    protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    	Assert.notEmpty(basePackages, "At least one base package must be specified");
    	Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
    	for (String basePackage : basePackages) {
    		// 扫描路径下的候选类并解析为定义
    		Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
    		for (BeanDefinition candidate : candidates) {
    
    			// 获取元数据
    			ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
    			candidate.setScope(scopeMetadata.getScopeName());
    			String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
    
    			if (candidate instanceof AbstractBeanDefinition) {
    				postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
    			}
    			if (candidate instanceof AnnotatedBeanDefinition) {
    				AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
    			}
    			if (checkCandidate(beanName, candidate)) {
    				BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
    				// 设置作用域代理
    				definitionHolder =
    						AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    				beanDefinitions.add(definitionHolder);
    				// 注册bean定义
    				registerBeanDefinition(definitionHolder, this.registry);
    			}
    		}
    	}
    	return beanDefinitions;
    }
    ```





核心内容

1. BeanFactory：bena工厂，提供了一种高级配置机制，能够管理任何对象。
2. ApplicationContext：应用上下文，bean工厂的扩展，提供更多企业支持，支持特定环境的上下文。
3. Reader、Scanner：用来读取/扫描和注册bean定义。
4. BeanDefinitionMap：bean定义集合，用于存储bean定义，存在于bean工厂中。





流程

![](../../img/AnnotationConfigApplicationContext流程.svg)

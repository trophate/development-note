版本：5.3.27

<br>

IOC的实现：IOC容器

<br>

容器对象：

1. AnnotationConfigApplicationContext：基于注解配置管理bean。
2. ClassPathXmlApplicationContext：基于xml文件配置管理bean。

<br>

AnnotationConfigApplicationContext

源码解析

1. 下列代码为容器创建方式之一，以此为例分析。

   ```java
   AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(A.class, B.class, Planet.class);
   ```

2. AnnotationConfigApplicationContext是一个上下文，继承关系如下：

   ```java
   public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry
   ```

   [继承图](../img/AnnotationConfigApplicationContext继承图.png)

   它是BeanFactory、AnnotationConfigRegistry的实现，即它既是一个bean工厂也是一个注册表。注册表提供注册和扫描功能。

   

   它有两个属性reader、scanner。scanner能够扫描类路径，识别bean定义，并将其注册。scanner能够识别@Component、@Repository、@Service、@Controller、@ManagedBean（Java EE 6's）、@Named（JSR-330's）注解。reader是scanner的平替，能解析相同的注解，但只能用作显式的注册。

   ```java
   private final AnnotatedBeanDefinitionReader reader;
   
   private final ClassPathBeanDefinitionScanner scanner;
   ```

3. 从1进入AnnotationConfigApplicationContext。

   ```java
   public AnnotationConfigApplicationContext(String... basePackages) {
   	this();
   	// 扫描
   	scan(basePackages);
   	// 刷新
   	refresh();
   }
   ```

4. 进入this。

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

5. 进入AnnotatedBeanDefinitionReader。

   ```java
   public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
   	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
   	Assert.notNull(environment, "Environment must not be null");
   	// 设置注册表
   	this.registry = registry;
   	this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
   	// 注册所有注解相关的后置处理器
   	AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
   }
   ```

6. 进入registerAnnotationConfigProcessors。

   ```java
   public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
   		BeanDefinitionRegistry registry, @Nullable Object source) {
   
   	// 从注册表中获取DefaultListableBeanFactory对象.
   	DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
   	if (beanFactory != null) {
   		// 设置依赖比较器, 用于确定bean的执行顺序. 其产生的优先级不会影响bean的加载顺序. 相关内容: Order接口, @Order, @Priority
   		if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
   			beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
   		}
   		// 设置自动装配候选者解析器, 用于判断bean定义是否能够作为自动装配的候选者.
   		if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
   			beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
   		}
   	}
   
   	Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);
   
   	// ------- 注册后置处理器 -------
   
   	// @Configuration后置处理器
   	if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
   		RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
   		def.setSource(source);
   		// 注册后置处理器
   		beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
   	}
   
   	// @Autowired后置处理器
   	if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
   		RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
   		def.setSource(source);
   		// 注册后置处理器
   		beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
   	}
   
   	// 对于JSR-250注解的支持
   	// Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
   	if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
   		RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
   		def.setSource(source);
   		// 注册后置处理器
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
   		// 注册后置处理器
   		beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
   	}
   
   	// @EventListener后置处理器
   	if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
   		RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
   		def.setSource(source);
   		// 注册后置处理器
   		beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
   	}
   
   	// 事件监听工厂
   	if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
   		RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
   		def.setSource(source);
   		// 注册后置处理器
   		beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
   	}
   
   	return beanDefs;
   }
   ```

   <br>

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

   <br>

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
   	// 设置注册表
   	this.registry = registry;
   
   	// 注册默认拦截器, 用于拦截注解.
   	if (useDefaultFilters) {
   		registerDefaultFilters();
   	}
   	// 设置环境变量, 包含: 配置, 属性.
   	setEnvironment(environment);
   	// 设置资源加载器, 用于将给定的配置文件路径解析为资源对象.
   	setResourceLoader(resourceLoader);
   }
   ```

8. 从3进入register。该方法显式将目标类解析为bean定义并注册。

   ```java
   @Override
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
   
   	// 解析类信息, 生成bean定义. 解析过程中会从注解中提取元数据.
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
   	// 注册bean定义, 定义被注册到了beanDefinitionMap中.
   	BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
   }
   ```

10. 从3进入refresh。该方法加载或刷新配置的持久化表示，简单说就是执行bean的实例化并刷新上下文。该方法是启动方法，如果失败，则需要销毁所有已经创建的单例bean，即单例bean要么全部实例化，要么全部都没实例化。

    ```java
    @Override
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
    			// 调用在上下文中被作为bean被注册的bean工厂后置处理器
    			// Invoke factory processors registered as beans in the context.
    			invokeBeanFactoryPostProcessors(beanFactory);

    			// 注册拦截bean创建的bean后置处理器
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

    			// 实例化所有剩余的非懒加载的单例bean
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

12. 进入beanFactory.preInstantiateSingletons。

    ```java
    @Override
    public void preInstantiateSingletons() throws BeansException {
    	if (logger.isTraceEnabled()) {
    		logger.trace("Pre-instantiating singletons in " + this);
    	}
    
    	// Iterate over a copy to allow for init methods which in turn register new bean definitions.
    	// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
    	List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
    
    	// 初始化
    	// Trigger initialization of all non-lazy singleton beans...
    	for (String beanName : beanNames) {
    		// 合并bean定义并返回RootBeanDefinition实例, RootBeanDefinition是在运行中实际使用的"统一"bean定义视图.
    		RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
    		// 抽象bean/单例/懒加载
    		if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
    			// 是否是工厂bean
    			if (isFactoryBean(beanName)) {
    				Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
    				if (bean instanceof FactoryBean) {
    					FactoryBean<?> factory = (FactoryBean<?>) bean;
    					boolean isEagerInit;
    					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
    						isEagerInit = AccessController.doPrivileged(
    								(PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
    								getAccessControlContext());
    					}
    					else {
    						isEagerInit = (factory instanceof SmartFactoryBean &&
    								((SmartFactoryBean<?>) factory).isEagerInit());
    					}
    					if (isEagerInit) {
    						// 获取bean实例, 如果不存在则创建.
    						getBean(beanName);
    					}
    				}
    			}
    			else {
    				// 获取bean实例, 如果不存在则创建.
    				getBean(beanName);
    			}
    		}
    	}
    
    	// 初始化后回调
    	// Trigger post-initialization callback for all applicable beans...
    	for (String beanName : beanNames) {
    		Object singletonInstance = getSingleton(beanName);
    		if (singletonInstance instanceof SmartInitializingSingleton) {
    			StartupStep smartInitialize = this.getApplicationStartup().start("spring.beans.smart-initialize")
    					.tag("beanName", beanName);
    			SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
    			if (System.getSecurityManager() != null) {
    				AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
    					smartSingleton.afterSingletonsInstantiated();
    					return null;
    				}, getAccessControlContext());
    			}
    			else {
    				smartSingleton.afterSingletonsInstantiated();
    			}
    			smartInitialize.end();
    		}
    	}
    }
    ```

13. 进入getBean。

    ```java
    @SuppressWarnings("unchecked")
    protected <T> T doGetBean(
    		String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
    		throws BeansException {
    
    	String beanName = transformedBeanName(name);
    	Object beanInstance;
    
    	// 在缓存中查找目标bean实例
    	// Eagerly check singleton cache for manually registered singletons.
    	Object sharedInstance = getSingleton(beanName);
    	if (sharedInstance != null && args == null) {
    		if (logger.isTraceEnabled()) {
    			if (isSingletonCurrentlyInCreation(beanName)) {
    				logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
    						"' that is not fully initialized yet - a consequence of a circular reference");
    			}
    			else {
    				logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
    			}
    		}
    		beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    	}
    
    	// 目标bean实例不存在
    	else {
    		// Fail if we're already creating this bean instance:
    		// We're assumably within a circular reference.
    		if (isPrototypeCurrentlyInCreation(beanName)) {
    			throw new BeanCurrentlyInCreationException(beanName);
    		}
    
    		// Check if bean definition exists in this factory.
    		BeanFactory parentBeanFactory = getParentBeanFactory();
    		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
    			// Not found -> check parent.
    			String nameToLookup = originalBeanName(name);
    			if (parentBeanFactory instanceof AbstractBeanFactory) {
    				return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
    						nameToLookup, requiredType, args, typeCheckOnly);
    			}
    			else if (args != null) {
    				// Delegation to parent with explicit args.
    				return (T) parentBeanFactory.getBean(nameToLookup, args);
    			}
    			else if (requiredType != null) {
    				// No args -> delegate to standard getBean method.
    				return parentBeanFactory.getBean(nameToLookup, requiredType);
    			}
    			else {
    				return (T) parentBeanFactory.getBean(nameToLookup);
    			}
    		}
    
    		if (!typeCheckOnly) {
    			markBeanAsCreated(beanName);
    		}
    
    		StartupStep beanCreation = this.applicationStartup.start("spring.beans.instantiate")
    				.tag("beanName", name);
    		try {
    			if (requiredType != null) {
    				beanCreation.tag("beanType", requiredType::toString);
    			}
    			RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
    			checkMergedBeanDefinition(mbd, beanName, args);
    
    			// Guarantee initialization of beans that the current bean depends on.
    			String[] dependsOn = mbd.getDependsOn();
    			if (dependsOn != null) {
    				for (String dep : dependsOn) {
    					if (isDependent(beanName, dep)) {
    						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
    								"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
    					}
    					// 注册依赖bean
    					registerDependentBean(dep, beanName);
    					try {
    						getBean(dep);
    					}
    					catch (NoSuchBeanDefinitionException ex) {
    						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
    								"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
    					}
    				}
    			}
    
    			// 创建bean实例
    			// Create bean instance.
    			if (mbd.isSingleton()) {
    				sharedInstance = getSingleton(beanName, () -> {
    					try {
    						// 创建bean实例, 包含创建bean实例, 填充bean实例, 调用后置处理器等.
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
    		}
    		catch (BeansException ex) {
    			beanCreation.tag("exception", ex.getClass().toString());
    			beanCreation.tag("message", String.valueOf(ex.getMessage()));
    			cleanupAfterBeanCreationFailure(beanName);
    			throw ex;
    		}
    		finally {
    			beanCreation.end();
    		}
    	}
    
    	return adaptBeanInstance(name, beanInstance, requiredType);
    }
    ```

14. 进入createBean。

    ```java
    @Override
    protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    		throws BeanCreationException {
    
    	if (logger.isTraceEnabled()) {
    		logger.trace("Creating instance of bean '" + beanName + "'");
    	}
    	RootBeanDefinition mbdToUse = mbd;
    
    	// Make sure bean class is actually resolved at this point, and
    	// clone the bean definition in case of a dynamically resolved Class
    	// which cannot be stored in the shared merged bean definition.
    	Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    	if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
    		mbdToUse = new RootBeanDefinition(mbd);
    		mbdToUse.setBeanClass(resolvedClass);
    	}
    
    	// Prepare method overrides.
    	try {
    		mbdToUse.prepareMethodOverrides();
    	}
    	catch (BeanDefinitionValidationException ex) {
    		throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
    				beanName, "Validation of method overrides failed", ex);
    	}
    
    	// 后置处理器代理bean实例, 并返回一个代理对象.
    	try {
    		// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
    		Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    		if (bean != null) {
    			return bean;
    		}
    	}
    	catch (Throwable ex) {
    		throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
    				"BeanPostProcessor before instantiation of bean failed", ex);
    	}
    
    	// 实际创建bean实例
    	try {
    		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
    		if (logger.isTraceEnabled()) {
    			logger.trace("Finished creating instance of bean '" + beanName + "'");
    		}
    		return beanInstance;
    	}
    	catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
    		// A previously detected exception with proper bean creation context already,
    		// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
    		throw ex;
    	}
    	catch (Throwable ex) {
    		throw new BeanCreationException(
    				mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
    	}
    }
    ```

15. 进入doCreateBean。

    ```java
    protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    			throws BeanCreationException {
    
    		// 实例化bean
    		// Instantiate the bean.
    		BeanWrapper instanceWrapper = null;
    		if (mbd.isSingleton()) {
    			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    		}
    		if (instanceWrapper == null) {
    			// 使用合适的策略(工厂方法/构造函数/简单实例化)创建bean实例
    			instanceWrapper = createBeanInstance(beanName, mbd, args);
    		}
    		Object bean = instanceWrapper.getWrappedInstance();
    		Class<?> beanType = instanceWrapper.getWrappedClass();
    		if (beanType != NullBean.class) {
    			mbd.resolvedTargetType = beanType;
    		}
    
    		// 允许后置处理器修改合并的bean定义
    		// Allow post-processors to modify the merged bean definition.
    		synchronized (mbd.postProcessingLock) {
    			if (!mbd.postProcessed) {
    				try {
    					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
    				}
    				catch (Throwable ex) {
    					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
    							"Post-processing of merged bean definition failed", ex);
    				}
    				mbd.postProcessed = true;
    			}
    		}
    
    		// 快速缓存bean实例, 用来解决循环引用问题.
    		// Eagerly cache singletons to be able to resolve circular references
    		// even when triggered by lifecycle interfaces like BeanFactoryAware.
    		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
    				isSingletonCurrentlyInCreation(beanName));
    		if (earlySingletonExposure) {
    			if (logger.isTraceEnabled()) {
    				logger.trace("Eagerly caching bean '" + beanName +
    						"' to allow for resolving potential circular references");
    			}
    			// 单例工厂, 用于必要时快速创建实例. 工厂与bean一一对应.
    			addSingletonFactory(beanName,
    					// 匿名类
    					() ->
    							// 获目标bean早期实例的引用
    							getEarlyBeanReference(beanName, mbd, bean));
    		}
    
    		// 初始化bean实例
    		// Initialize the bean instance.
    		Object exposedObject = bean;
    		try {
    			// bean实例属性赋值
    			populateBean(beanName, mbd, instanceWrapper);
    			// 使用bean工厂回调, 初始方法, 后置处理器初始化bean实例.
    			exposedObject = initializeBean(beanName, exposedObject, mbd);
    		}
    		catch (Throwable ex) {
    			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
    				throw (BeanCreationException) ex;
    			}
    			else {
    				throw new BeanCreationException(
    						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
    			}
    		}
    
    		if (earlySingletonExposure) {
    			Object earlySingletonReference = getSingleton(beanName, false);
    			if (earlySingletonReference != null) {
    				if (exposedObject == bean) {
    					exposedObject = earlySingletonReference;
    				}
    				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
    					String[] dependentBeans = getDependentBeans(beanName);
    					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
    					for (String dependentBean : dependentBeans) {
    						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
    							actualDependentBeans.add(dependentBean);
    						}
    					}
    					if (!actualDependentBeans.isEmpty()) {
    						throw new BeanCurrentlyInCreationException(beanName,
    								"Bean with name '" + beanName + "' has been injected into other beans [" +
    								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
    								"] in its raw version as part of a circular reference, but has eventually been " +
    								"wrapped. This means that said other beans do not use the final version of the " +
    								"bean. This is often the result of over-eager type matching - consider using " +
    								"'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");
    					}
    				}
    			}
    		}
    
    		// 将bean注册为一次性的
    		// Register bean as disposable.
    		try {
    			// 将bean添加到bean工厂的一次性bean列表中, 在销毁时使用.
    			registerDisposableBeanIfNecessary(beanName, bean, mbd);
    		}
    		catch (BeanDefinitionValidationException ex) {
    			throw new BeanCreationException(
    					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    		}
    
    		return exposedObject;
    	}
    ```

16. 进入initializeBean。

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
    
    	// 在初始化前调用后置处理器, 自定义后置处理器也是在此处执行.
    	Object wrappedBean = bean;
    	if (mbd == null || !mbd.isSynthetic()) {
    		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    	}
    
    	// 调用初始化方法
    	try {
    		// 如果bean实现了InitializingBean或设置了自定义初始化方法
    		invokeInitMethods(beanName, wrappedBean, mbd);
    	}
    	catch (Throwable ex) {
    		throw new BeanCreationException(
    				(mbd != null ? mbd.getResourceDescription() : null),
    				beanName, "Invocation of init method failed", ex);
    	}
    
    	// 在初始化后调用后置处理器, 自定义后置处理器也是在此处执行.
    	if (mbd == null || !mbd.isSynthetic()) {
    		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    	}
    
    	return wrappedBean;
    }
    ```

17. 返回1，构造函数传入包路径，这是另一种容器创建方式。

    ```java
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext("com.test");
    ```

18. 进入AnnotationConfigApplicationContext。与之前相似，只是使用了不一样的注册方法。

    ```java
    public AnnotationConfigApplicationContext(String... basePackages) {
        this();
        // 扫描
        scan(basePackages);
        // 刷新
        refresh();
    }
    ```

19. 进入scan。该方法使用scanner处理目标路径。

    ```java
    @Override
    public void scan(String... basePackages) {
    	Assert.notEmpty(basePackages, "At least one base package must be specified");
    	StartupStep scanPackages = this.getApplicationStartup().start("spring.context.base-packages.scan")
    			.tag("packages", () -> Arrays.toString(basePackages));
    	// 扫描
    	this.scanner.scan(basePackages);
    	scanPackages.end();
    }
    ```

20. 进入scan。

    ```java
    public int scan(String... basePackages) {
    	int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
    
    	// 扫描
    	doScan(basePackages);
    
    	// 注册所有注解相关的后置处理器
    	// Register annotation config processors, if necessary.
    	if (this.includeAnnotationConfig) {
    		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    	}
    
    	return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
    }
    ```

21. 进入doScan。

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

<br>核心内容

1. BeanFactory：bena工厂，提供了一种高级配置机制，能够管理任何对象。
2. ApplicationContext：应用上下文，bean工厂的扩展，提供更多企业支持，支持特定环境的上下文。
3. reader、scanner：用来读取/扫描和注册bean定义。
4. beanDefinitionMap：bean定义集合，用于存储bean定义，存在于bean工厂中。

<br>

流程

1. 整体

   ![](../img/AnnotationConfigApplicationContext流程.svg)

2. bean实例化

   ![](../img/bean实例化流程.svg)

<br>

循环依赖及解决

定义：多个bean之间相互依赖，依赖成环，导致bean创建失败。

设有A、B两个bean，两者相互依赖，现在实例化它们。

```java
@Component
public class A {
    
	@Autowired
	private B b;
}

@Component
public class B {
    
	@Autowired
	private A a;
}
```

创建A实例 → 需要B实例 → 创建B实例 → 需要A实例

Spring提供了特定场景下循环依赖的解决方案，方案如下：

bean实例被存储在三级缓存中。

```java
// DefaultSingletonBeanRegistry

// bean实例列表, 完全走完流程的bean实例. 一级缓存
/** Cache of singleton objects: bean name to bean instance. */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

// 单例工厂, 用于快速创建bean实例, 与bean一一对应. 主要用于快速获取bean实例的代理对象. 三级缓存
/** Cache of singleton factories: bean name to ObjectFactory. */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

// bean早期实例, 没有走完全部流程的bean实例. 二级缓存
/** Cache of early singleton objects: bean name to bean instance. */
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

// 已经实例化bean列表
/** Set of registered singletons, containing the bean names in registration order. */
private final Set<String> registeredSingletons = new LinkedHashSet<>(256);
```

创建A实例，缓存A的早期实例（不完全实例，没有初始化属性），然后设置属性值。此时需要B实例，但没有B实例，于是创建B实例。在创建B实例时，需要A实例，虽然没有A的完整实例，但缓存了A的早期实例，于是B完成实例化。然后A实例获取到了B实例，完成了A的实例化。

![](../img/循环依赖解决方案流程.svg)

因为AOP等操作，可能导致A早期实例与A最终实例不是一个对象。这这种情况下创建的B实例是不对的，因为B实例关联的是A的早期实例，而需要的是A最终实例。为了解决这个问题，Spring缓存了单例工厂。这个工厂的作用是用来获取bean的早期实例，如果bean将被代理，则获取一个bean的早期代理实例。该工厂保证了早期实例始终是“正确的”那个。

过程可以看出能够被解决的循环依赖需要满足依赖注入不能与实例创建形成不可分割操作。如构造器注入就不能解决，setter注入就可以解决。

代码：

1. 从缓存中获取bean实例，从源码12进入getSingleton。

   ```java
   @Nullable
   protected Object getSingleton(String beanName, boolean allowEarlyReference) {
   	// 在singletonObjects中查询(一级缓存)
   	// Quick check for existing instance without full singleton lock
   	Object singletonObject = this.singletonObjects.get(beanName);
   	// 在earlySingletonObjects中查询(二级缓存)
   	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
   		singletonObject = this.earlySingletonObjects.get(beanName);
   		// 创建bean早期实例, 并将其存入二级缓存.(三级缓存)
   		if (singletonObject == null && allowEarlyReference) {
   			synchronized (this.singletonObjects) {
   				// Consistent creation of early reference within full singleton lock
   				singletonObject = this.singletonObjects.get(beanName);
   				if (singletonObject == null) {
   					singletonObject = this.earlySingletonObjects.get(beanName);
   					if (singletonObject == null) {
   						ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
   						if (singletonFactory != null) {
   							singletonObject = singletonFactory.getObject();
   							this.earlySingletonObjects.put(beanName, singletonObject);
   							this.singletonFactories.remove(beanName);
   						}
   					}
   				}
   			}
   		}
   	}
   	return singletonObject;
   }
   ```

2. bean实例缓存，从源码14进入addSingletonFactory。

   ```java
   protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
   	Assert.notNull(singletonFactory, "Singleton factory must not be null");
   	synchronized (this.singletonObjects) {
   		if (!this.singletonObjects.containsKey(beanName)) {
   			// 添加单例工厂
   			this.singletonFactories.put(beanName, singletonFactory);
   			// 移除bean早期实例
   			this.earlySingletonObjects.remove(beanName);
   			// 注册到已注册实例表中
   			this.registeredSingletons.add(beanName);
   		}
   	}
   }
   ```

   


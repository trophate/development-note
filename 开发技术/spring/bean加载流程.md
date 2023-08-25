版本：5.3.27



bean定义：注解、xml

spring容器：AnnotationConfigApplicationContext（注解）、ClassPathXmlApplicationContext（xml）



AnnotationConfigApplicationContext

创建方式

1. 将类信息传入构造函数：

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(A.class, B.class);
Object a = context.getBean("a");
```

2. 添加bean注解，并扫描路径：

```java
@Component
public class A {}
```

```java
@ComponentScan(basePackages = "com.test")
public class BeanConfig {
    
}
```

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(BeanConfig.class);
Object a = context.getBean("a");
```

3. 定义配置类：

```java
public class BeanConfig {
    
    @Bean
    public A a() {
        return new A();
    }
}
```

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(BeanConfig.class);
Object a = context.getBean("a");
```

解析

1. 从构造函数点入

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(A.class);
```

2. 先调用了无参构造函数，然后调用了一个注册方法和一个刷新方法。

```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
    this();
    register(componentClasses);
    refresh();
}
```

3. 进入this方法，其中涉及两个重要对象：reader和scanner。scanner用以扫描类路径上的bean定义并将其注册到指定注册表中，scanner可识别的注解有：@Component、@Repository、@Service、@Controller、@ManagedBean（Java EE 6's）、@Named（JSR-330's）。

```java
public AnnotationConfigApplicationContext() {
    // 步骤记录ApplicationStartup期间特定阶段或行为的度量
    StartupStep createAnnotatedBeanDefReader = this.getApplicationStartup().start("spring.context.annotated-bean-reader.create");
    this.reader = new AnnotatedBeanDefinitionReader(this);
    createAnnotatedBeanDefReader.end();
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

4. 进入reader的构造函数，

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
    this(registry, getOrCreateEnvironment(registry));
}
```

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
   Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
   Assert.notNull(environment, "Environment must not be null");
   this.registry = registry;
   this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
   AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}
```

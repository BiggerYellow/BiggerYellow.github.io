---
layout: default
springboot: true
modal-id: 20000004
date: 2021-09-07
img: pexels-anastasia-pavlova-8692301.jpg
alt: SpringBoot4
project-date: Septemper 2021
client: SpringBoot
category: SpringBoot
subtitle: SpringBoot启动源码解析(三)
description: SpringBootApplication Run方法解析(二):createApplicationContext
---
### Run方法入口:createApplicationContext
- - -
``` java
/**
 * Run the Spring application, creating and refreshing a new
 * {@link ApplicationContext}.
 * 运行Spring应用，创建和刷新一个新的应用上下文
 * @param args the application arguments (usually passed from a Java main method)
 * @return a running {@link ApplicationContext}
 */
public ConfigurableApplicationContext run(String... args) {
    //1.创建启动计时器
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    //创建启动上下文并初始化相应的Bootstrapper 目前暂未使用
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    //设置系统的配置模式，一般用于服务端开发 默认为true
    configureHeadlessProperty();
    //获取spring上下文启动监听者  从META-INF/spring.factories中获取SpringApplicationRunListener的实现类并加载
    SpringApplicationRunListeners listeners = getRunListeners(args);
    //发布启动中事件
    listeners.starting();
    try {
        //2.创建默认上下文参数
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        //3.准备环境，将资源属性添加到环境中并与上下文绑定
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        //设置spring.beaninfo.ignore中过滤的bean
        configureIgnoreBeanInfo(environment);
        //打印spring标记
        Banner printedBanner = printBanner(environment);
        //4.根据web应用类型创建指定的应用上下文
        //注意:这里反射创建AnnotationConfigServletWebServerApplicationContext时,构造中创建AnnotatedBeanDefinitionReader中的AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry)方法很重要
        //它将几个常见的BeanPostProcessor和BeanFactoryPostProcessor注册到注册表中  最重要的是internalConfigurationAnnotationProcessor来解析Configuration的配置类
        context = createApplicationContext();
        exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                new Class[] { ConfigurableApplicationContext.class }, context);
        //5.准备上下文
        //主要做了两件事:
        // 1.调用前面实例化的ApplicationContextInitializer的initialize方式初始化上下文
        // 2.在其中的load方法中将启动主函数bean定义注册到注册表中 beanDefinitionMap
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        //6.刷新上下文
        refreshContext(context);
        afterRefresh(context, applicationArguments);
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        //监听者发布已启动事件
        listeners.started(context);
        //7.调用所有的ApplicationRunner和CommandLineRunner的实现类
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        //监听者发布运行中事件
        listeners.running(context);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```
本章我们主要看下createApplicationContext:
第四步就是通过反射创建 __ApplicationContext__ ,相关源码如下:

``` java
/**
 * The class name of application context that will be used by default for non-web
 * environments.
 * 默认情况下将用于非web环境的应用程序上下文的类名
 */
public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
        + "annotation.AnnotationConfigApplicationContext";

/**
 * The class name of application context that will be used by default for web
 * environments.
 * 默认情况下将用于web环境的应用程序上下文的类名
 */
public static final String DEFAULT_SERVLET_WEB_CONTEXT_CLASS = "org.springframework.boot."
        + "web.servlet.context.AnnotationConfigServletWebServerApplicationContext";

/**
 * The class name of application context that will be used by default for reactive web
 * environments.
 * 默认情况下将用于reactive web环境的应用程序上下文的类名
 */
public static final String DEFAULT_REACTIVE_WEB_CONTEXT_CLASS = "org.springframework."
        + "boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext";

/**
 * Strategy method used to create the {@link ApplicationContext}. By default this
 * method will respect any explicitly set application context or application context
 * class before falling back to a suitable default.
 * 创建应用上下文使用的策略方法.默认情况下,此方法将在回退到合适的默认值之前尊重任何显示设置的应用程序上下文或类.
 */
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            switch (this.webApplicationType) {
            //通常我们的上下文默认是Servlet类型,实例化AnnotationConfigServletWebServerApplicationContext
            case SERVLET:
                contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
                break;
            case REACTIVE:
                contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                break;
            default:
                contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
            }
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                    "Unable create a default ApplicationContext, please specify an ApplicationContextClass", ex);
        }
    }
    //真正的实例化位置
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```
因为我们的应用类型为 __SERVLET__ ,所以应该初始化 __AnnotationConfigServletWebServerApplicationContext__ , 下面我们继续看下实例化 __AnnotationConfigServletWebServerApplicationContext__ 的源码:
``` java
/**
 * Create a new {@link AnnotationConfigServletWebServerApplicationContext} that needs
 * to be populated through {@link #register} calls and then manually
 * {@linkplain #refresh refreshed}.
 * 创建新的AnnotationConfigServletWebServerApplicationContext, 他需要通过通过register调用填充然后手动refreshed
 */
public AnnotationConfigServletWebServerApplicationContext() {
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```
我们首先来看下 __AnnotationConfigServletWebServerApplicationContext__ 的作用:接受带有 __@Configuration__ 、 __@Component__ 和 __inject__ 注解的类,可以指定类进行注册同样可以根据类路径扫描来注册.  
在它的无参构造中,它又创建了 __AnnotatedBeanDefinitionReader__ 和 __ClassPathBeanDefinitionScanner__ .  
其中 __AnnotatedBeanDefinitionReader__ 的作用是以编程方式注册Bean类; __ClassPathBeanDefinitionScanner__ 的作用是扫描类路径下所有以 __@Component、@Repository、@Service、@Controller、ManagedBean、Named__ 等注解修饰的类,并注册相关的bean定义.  
  
4.1 首先我们先看下 __AnnotatedBeanDefinitionReader__ 的源码:  

``` java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
    this(registry, getOrCreateEnvironment(registry));
}

/**
 * Create a new {@code AnnotatedBeanDefinitionReader} for the given registry,
 * using the given {@link Environment}.
 * 使用给定的注册表和环境创建AnnotatedBeanDefinitionReader
 */
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    Assert.notNull(environment, "Environment must not be null");
    this.registry = registry;
    //创建用于处理@Conditional注解的内部类
    this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
    //这里是最重要的地方,注册了多个后置处理器,供后面创建bean的时候调用
    AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}
```
可见 __AnnotatedBeanDefinitionReader__ 主构造中最重要的就是 __AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry)__ 这个方法,我们继续往下看看:
``` java
/**
 * The bean name of the internally managed Configuration annotation processor.
 * 内部管理的配置注解处理器的bean名称
 */
public static final String CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME =
        "org.springframework.context.annotation.internalConfigurationAnnotationProcessor";

/**
 * The bean name of the internally managed Autowired annotation processor.
 * 内部管理的自动注入注解处理器的bean名称
 */
public static final String AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME =
        "org.springframework.context.annotation.internalAutowiredAnnotationProcessor";

/**
 * The bean name of the internally managed JSR-250 annotation processor.
 * 内部管理的JSR-250注解处理器的bean名称
 */
public static final String COMMON_ANNOTATION_PROCESSOR_BEAN_NAME =
        "org.springframework.context.annotation.internalCommonAnnotationProcessor";

/**
 * The bean name of the internally managed JPA annotation processor.
 * 内部处理的JPA注解处理器的bean名称
 */
public static final String PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME =
        "org.springframework.context.annotation.internalPersistenceAnnotationProcessor";

/**
 * The bean name of the internally managed @EventListener annotation processor.
 * 内部处理的@EventListener注解处理器的bean名称
 */
public static final String EVENT_LISTENER_PROCESSOR_BEAN_NAME =
        "org.springframework.context.event.internalEventListenerProcessor";

/**
 * The bean name of the internally managed EventListenerFactory.
 * 内部处理的EventListenerFactory的bean名称
 */
public static final String EVENT_LISTENER_FACTORY_BEAN_NAME =
        "org.springframework.context.event.internalEventListenerFactory";

/**
 * Register all relevant annotation post processors in the given registry.
 * 在给定的注册表中注册所有相关注解的后置处理器
 */
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
        BeanDefinitionRegistry registry, @Nullable Object source) {
    //获取最原始的bean工厂
    DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
    if (beanFactory != null) {
        //设置AnnotationAwareOrderComparator,主要用于 @Order和@Priority的排序功能
        if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
            beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
        }
        //设置ContextAnnotationAutowireCandidateResolver,主要提供对限定符注解以及@Lazy注解的懒加载
        if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
            beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
        }
    }

    Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);
    //创建ConfigurationClassPostProcessor后置处理器bean定义,并注册到bean工厂中
    //ConfigurationClassPostProcessor的主要作用就是在启动时处理@Configuration修饰的类,其实就是我们的主函数对应的类
    if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
    }
    //创建AutowiredAnnotationBeanPostProcessor后置处理器Bean定义,并注册到bean工厂中
    //AutowiredAnnotationBeanPostProcessor的主要作用是自动装配注解的字段、set方法和任何配置方法,默认通过@Autowired和@Value
    if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
    //检查是否支持JSR-250,支持则创建CommonAnnotationBeanPostProcessor后置处理器并注册到bean工厂中
    //CommonAnnotationBeanPostProcessor的主要作用是支持开箱即用的java注解,主要是annotation中的注解,一般是@PostConstruct、@PreDestroy,最主要是@Resources
    if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
    //检查是否支持JPA,支持则创建PersistenceAnnotationBeanPostProcessor后置处理器并注册到bean工厂中
    //PersistenceAnnotationBeanPostProcessor的主要作用是为了注入相关EntityManagerFactory和EntityManager资源处理PersistenceUnit和PersistenceContext注解
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

    //创建EventListenerMethodProcessor后置处理器并注册到bean工厂中
    //EventListenerMethodProcessor的主要作用是作为单独的ApplicationListener实例,主要用于早期检索,避免对其处理器bean和他的EventListenerFactory委托者进行aop检查
    if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
    }
    
    //创建DefaultEventListenerFactory bean定义并注册到bean工厂
    //DefaultEventListenerFactory主要是为了常规的EventListener注解
    if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
    }

    return beanDefs;
}
```
可见 __AnnotatedBeanDefinitionReader__ 的作用就是注册了多个后置处理器.我们来看这几个后置处理器,先看他们的类图:  
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/AnnotationConfigProcessors.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="AnnotationConfigProcessors"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/AnnotationConfigProcessors.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">AnnotationConfigProcessors</div>
    </a>
</center>

其中最重要的就是 __ConfigurationClassPostProcessor__ 从类图中也可以看到它继承的父类更多,其中最重要的父类就是 __BeanDefinitionRegistryPostProcessor__ ,它就是处理主函数的核心,具体后面再介绍.  
我们一一介绍下这几个类的大概作用:  
- ConfigurationClassPostProcessor:最为重要,在启动时处理 __@Configuration__ 修饰的类,主要就是我们启动时使用 __@SpringBootApplication__ 修饰的主函数
- AutowiredAnnotationBeanPostProcessor:主要作用是通过 __@Autowired、@Value__ 等注解自动装配字段、set方法和任何配置方法
- CommonAnnotationBeanPostProcessor:主要作用是支持开箱即用的方法,例如 __@PostConstruct、@PreDestroy、@Resources__ 等
- PersistenceAnnotationBeanPostProcessor:主要作用是为了注入相关EntityManagerFactory和EntityManager资源处理PersistenceUnit和PersistenceContext注解
- EventListenerMethodProcessor:主要作用是作为单独的ApplicationListener实例,主要用于早期检索,避免对其处理器bean和他的EventListenerFactory委托者进行aop检查
- DefaultEventListenerFactory:主要是为了常规的EventListener注解

>综上所述: __AnnotatedBeanDefinitionReader__ 的主要作用就是注册多个后置处理器到bean工厂中,为后面解析主函数做准备  

4.2 接下来我们再看下 __ClassPathBeanDefinitionScanner__ 的源码:  
``` java
/**
 * Create a new {@code ClassPathBeanDefinitionScanner} for the given bean factory and
 * using the given {@link Environment} when evaluating bean definition profile metadata.
 * 为给定的bean工厂创建ClassPathBeanDefinitionScanner,并在评估bean定义配置元数据时使用给定的Environment
 */
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
        Environment environment, @Nullable ResourceLoader resourceLoader) {

    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    this.registry = registry;
    //注册默认过滤器 主要是@Component、ManagedBean、Named
    if (useDefaultFilters) {
        registerDefaultFilters();
    }
    //设置environment
    setEnvironment(environment);
    //设置类加载器
    setResourceLoader(resourceLoader);
}
```
我们先来看下 __registerDefaultFilters()__ 方法具体做了什么:
``` java
/**
 * Register the default filter for {@link Component @Component}.
 * <p>This will implicitly register all annotations that have the
 * {@link Component @Component} meta-annotation including the
 * {@link Repository @Repository}, {@link Service @Service}, and
 * {@link Controller @Controller} stereotype annotations.
 * <p>Also supports Java EE 6's {@link javax.annotation.ManagedBean} and
 * JSR-330's {@link javax.inject.Named} annotations, if available.
 * 为@Component注册默认过滤器.他可以注册所有隐式含有@Component元注解,包括@Repository、@Service、@Controller
 * 同时还可以支持 ManagedBean和Named
 */
@SuppressWarnings("unchecked")
protected void registerDefaultFilters() {
    this.includeFilters.add(new AnnotationTypeFilter(Component.class));
    ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
    try {
        this.includeFilters.add(new AnnotationTypeFilter(
                ((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
        logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
    }
    catch (ClassNotFoundException ex) {
        // JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
    }
    try {
        this.includeFilters.add(new AnnotationTypeFilter(
                ((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
        logger.trace("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
    }
    catch (ClassNotFoundException ex) {
        // JSR-330 API not available - simply skip.
    }
}
```
下面还有一个方法比较有意思,那就是 __setResourceLoader(resourceLoader)__ 方法,乍一眼看上去它就是设置了资源加载器,并没有什么特别的作用,我们翻翻的源码看看:  
``` java
/**
 * Set the {@link ResourceLoader} to use for resource locations.
 * This will typically be a {@link ResourcePatternResolver} implementation.
 * <p>Default is a {@code PathMatchingResourcePatternResolver}, also capable of
 * resource pattern resolving through the {@code ResourcePatternResolver} interface.
 * 设置资源加载器以用于资源位置. 通常是ResourcePatternResolver的实现.
 */
@Override
public void setResourceLoader(@Nullable ResourceLoader resourceLoader) {
    this.resourcePatternResolver = ResourcePatternUtils.getResourcePatternResolver(resourceLoader);
    this.metadataReaderFactory = new CachingMetadataReaderFactory(resourceLoader);
    this.componentsIndex = CandidateComponentsIndexLoader.loadIndex(this.resourcePatternResolver.getClassLoader());
}

/**
 * The location to look for components.
 * <p>Can be present in multiple JAR files.
 * 为了寻找组件的位置.可以出现在多个jar文件中
 */
public static final String COMPONENTS_RESOURCE_LOCATION = "META-INF/spring.components";

/**
 * Load and instantiate the {@link CandidateComponentsIndex} from
 * {@value #COMPONENTS_RESOURCE_LOCATION}, using the given class loader. If no
 * index is available, return {@code null}.
 * 使用给定的类加载器加载和实例化来自META-INF/spring.components的CandidateComponentsIndex.没有则返回null
 */
@Nullable
public static CandidateComponentsIndex loadIndex(@Nullable ClassLoader classLoader) {
    ClassLoader classLoaderToUse = classLoader;
    if (classLoaderToUse == null) {
        classLoaderToUse = CandidateComponentsIndexLoader.class.getClassLoader();
    }
    //主要就是通过doLoadIndex这个方法 那这里我们其实也可以用于扩展
    return cache.computeIfAbsent(classLoaderToUse, CandidateComponentsIndexLoader::doLoadIndex);
}

//这段代码就比较好理解了,尝试从META-INF/spring.components中读取配置,没有则返回,有则加载指定位置的文件,同时返回候选者的数量
private static CandidateComponentsIndex doLoadIndex(ClassLoader classLoader) {
    if (shouldIgnoreIndex) {
        return null;
    }
    try {
        Enumeration<URL> urls = classLoader.getResources(COMPONENTS_RESOURCE_LOCATION);
        if (!urls.hasMoreElements()) {
            return null;
        }
        List<Properties> result = new ArrayList<>();
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
            result.add(properties);
        }
        if (logger.isDebugEnabled()) {
            logger.debug("Loaded " + result.size() + "] index(es)");
        }
        int totalCount = result.stream().mapToInt(Properties::size).sum();
        return (totalCount > 0 ? new CandidateComponentsIndex(result) : null);
    }
    catch (IOException ex) {
        throw new IllegalStateException("Unable to load indexes from location [" +
                COMPONENTS_RESOURCE_LOCATION + "]", ex);
    }
}
```
>综上所述: __ClassPathBeanDefinitionScanner__ 的作用就是扫描所有类路径下所有被@Component、@Repository、@Service、@Controller以及ManagedBean和Named注解的类,并将它们注册为bean定义.  

>createApplicationContext总结

>第四步主要就是创建AnnotationConfigServletWebServerApplicationContext上下文,在构造AnnotationConfigServletWebServerApplicationContext的同时又创建用于注册所有bean的AnnotatedBeanDefinitionReader和扫描所有指定注解类的ClassPathBeanDefinitionScanner.
>其中AnnotatedBeanDefinitionReader又注册了多个后置处理器为后续做准备,最重要的就是ConfigurationClassPostProcessor.

- - -

### 组件扫描扩展
- - -
背景:虽然类路径扫描速度非常快,但是可以通过在编译时创建一个静态候选列表来提高应用程序的启动性能.  

首先需要先添加maven
``` java
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context-indexer</artifactId>
        <version>5.1.8.RELEASE</version>
        <optional>true</optional>
    </dependency>
</dependencies>
```
他的作用就是在编译过程中生成一个包含jar文件的META-INF/spring.components文件.此文件中就包含应用中所有以 __@Component以及以@Component为元注解__ 的类.
如下图所示:
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/spring.components示例.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="spring.components示例"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/spring.components示例.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">spring.components示例</div>
    </a>
</center>

那我们可以这样使用:
``` java
public static void main(String[] args) {
    ConfigurableApplicationContext context = SpringApplication.run(DemoApplication.class, args);
    //"META-INF/spring.components"
    CandidateComponentsIndex candidateComponentsIndex = CandidateComponentsIndexLoader.loadIndex(DemoApplication.class.getClassLoader());
    if (Objects.nonNull(candidateComponentsIndex)){
        Set<String> candidates = candidateComponentsIndex.getCandidateTypes("com.hcc.example", "org.springframework.stereotype.Component");
        for (String candidate:candidates){
            if (candidate.contains("MyComponent")){
                MyComponent myComponent = (MyComponent)context.getBeanFactory().getBean("myComponent");
                myComponent.test();
            }
        }
    }
}
```
具体如果扩展就可以根据我们的需要进行了.  
当在类路径上找到META-INF/Spring.components时,索引将自动启用.如果某个索引部分可用于某些库(或用例),但无法为整个应用程序生成,则可以通过将spring.index.ignore设置为true(作为系统属性或类路径根目录下的spring.properties文件)来回滚到常规类路径安排(就像根本没有索引一样).
- - -

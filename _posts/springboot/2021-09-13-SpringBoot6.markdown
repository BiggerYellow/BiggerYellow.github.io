---
layout: default
springboot: true
modal-id: 20000006
date: 2021-09-13
img: pexels-rachel-claire-4577793.jpg
alt: SpringBoot
project-date: Septemper 2021
client: SpringBoot
category: SpringBoot
subtitle: SpringBoot启动源码解析(六)
description: SpringBootApplication Run方法解析(四):refreshContext
---
### Run方法入口
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
- - -
### refreshContext
- - -
&emsp;&emsp;本篇我们一起分析下SpringBoot启动流程中最重要的一步 __refreshContext__ ,字面意思就是刷新上下文,基本上这个完成后上下文就差不多可以使用了.
  
``` java
private void refreshContext(ConfigurableApplicationContext context) {
    //刷新上下文, 调用AbstractApplicationContext的refresh方法
    refresh(context);
    //是否需要注册下线钩子 默认为true
    if (this.registerShutdownHook) {
        try {
            //执行AbstractApplicationContext的注册下线钩子方法
            context.registerShutdownHook();
        }
        catch (AccessControlException ex) {
            // Not allowed in some environments.
        }
    }
}

/**
 * Refresh the underlying {@link ApplicationContext}.
 * 刷新潜在的上下文
 */
protected void refresh(ApplicationContext applicationContext) {
    Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
    ((AbstractApplicationContext) applicationContext).refresh();
}
```
&emsp;&emsp;由此可见,在SpringBoot中 __refresh__ 的作用就是调用spring中真正的刷新上下文操作,并在刷新完成后注册上下文下线钩子.  
&emsp;&emsp;接下来, 我们来真正看下在spring中的 __refresh__ 方法源码:  

``` java
/** Synchronization monitor for the "refresh" and "destroy". */
// 刷新和销毁的同步监视器
private final Object startupShutdownMonitor = new Object();

public void refresh() throws BeansException, IllegalStateException {
    //锁定当前监视器锁 保证只有一个
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        //1.为刷新准备上下文,主要是设置启动时间和激活标志位以及初始化一些其他的属性
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        //2.告诉子类刷新内部的bean工厂 主要为bean工厂设置serializationId 并返回该bean工厂
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        //3.准备在此上下文中使用的bean工厂 详细解析见下方
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            // 4. 允许在上下文子类中对bean工厂进行后置处理 注册WebApplicationContextServletContextAwareProcessor后置处理器和注册request、session等特定web范围 
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            // 非常核心 5.调用在上下文中注册为bean的工厂处理器
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
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
__refresh__ 中的每一个方法都是非常重要的,那就可以大约分为十三步,我们一步步往下分析.  
    
1. 第一步准备上下文:prepareRefresh,相关源码如下:  

``` java
//调用AnnotationConfigServletWebServerApplicationContext重写的prepareRefresh
@Override
protected void prepareRefresh() {
    this.scanner.clearCache();
    //调用父类AbstractApplicationContext的prepareRefresh方法
    super.prepareRefresh();
}

/**
 * Prepare this context for refreshing, setting its startup date and
 * active flag as well as performing any initialization of property sources.
 * 为刷新准备上下文, 设置它的开始时间和激活标志以及执行任何属性源的初始化
 */
protected void prepareRefresh() {
    // Switch to active.
    this.startupDate = System.currentTimeMillis();
    this.closed.set(false);
    this.active.set(true);

    if (logger.isDebugEnabled()) {
        if (logger.isTraceEnabled()) {
            logger.trace("Refreshing " + this);
        }
        else {
            logger.debug("Refreshing " + getDisplayName());
        }
    }

    // Initialize any placeholder property sources in the context environment.
    //在上下文环境中初始化任何占位符属性源  主要就重置 ServletContext和ServletConfig 默认为空 所以未进行处理
    initPropertySources();

    // Validate that all properties marked as required are resolvable:
    // see ConfigurablePropertyResolver#setRequiredProperties
    //验证所有标记为必须的属性都是可解析的
    getEnvironment().validateRequiredProperties();

    // Store pre-refresh ApplicationListeners...
    //初始化用于存储ApplicationListeners的集合 
    if (this.ealryApplicationListeners == null) {
        this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
    }
    else {
        // Reset local application listeners to pre-refresh state.
        this.applicationListeners.clear();
        this.applicationListeners.addAll(this.earlyApplicationListeners);
    }

    // Allow for the collection of early ApplicationEvents,
    // to be published once the multicaster is available...
    // 初始化早期事件收集器   允许收集早期的ApplicationEvents, 一旦多播器可用就发布
    this.earlyApplicationEvents = new LinkedHashSet<>();
}
```
&emsp;&emsp;可见第一步主要就是设置了启动时间 __startupDate__ 和启动标志 __active=true__ , 初始化上下文中 __ServletContext和ServletConfig__ 属性,最后并初始化存放早期应用监听者和应用事件的集合 __earlyApplicationListeners、earlyApplicationEvents__ .  

2.第二步obtainFreshBeanFactory,相关源码如下:

``` java
/**
 * Tell the subclass to refresh the internal bean factory.
 * 告诉子类刷下内部bean工厂
 */
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    //刷新bean工厂 即设置refreshed状态 并 设置bean工厂的序列化id
    refreshBeanFactory();
    //返回持有的bean工厂
    return getBeanFactory();
}

/**
 * Do nothing: We hold a single internal BeanFactory and rely on callers
 * to register beans through our public methods (or the BeanFactory's).
 * 我们拥有一个内部bean工厂并依靠调用者通过我们的公共方法注册bean
 */
@Override
protected final void refreshBeanFactory() throws IllegalStateException {
    //使用cas的方式设置 refreshed状态 只有原来为false才会设置成功 否则抛出非法状态异常
    if (!this.refreshed.compareAndSet(false, true)) {
        throw new IllegalStateException(
                "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
    }
    //为该bean工厂设置序列化id
    this.beanFactory.setSerializationId(getId());
}

/**
 * Specify an id for serialization purposes, allowing this BeanFactory to be
 * deserialized from this id back into the BeanFactory object, if needed.
 * 为序列化目的指定id,如果需要,运行将此bean工厂从这个id反序列化回bean工厂对象
 */
public void setSerializationId(@Nullable String serializationId) {
    //如果序列化id不为空 则在serializableFactories中保存当前序列化id和bean工厂的对应关系
    if (serializationId != null) {
        serializableFactories.put(serializationId, new WeakReference<>(this));
    }
    else if (this.serializationId != null) {
        serializableFactories.remove(this.serializationId);
    }
    //设置当前bean工厂的序列化id
    this.serializationId = serializationId;
}

/**
 * Return the single internal BeanFactory held by this context
 * (as ConfigurableListableBeanFactory).
 * 返回上下文持有的内部单个bean工厂
 */
@Override
public final ConfigurableListableBeanFactory getBeanFactory() {
    return this.beanFactory;
}
```
&emsp;&emsp;由此可见,第二步的作用就是刷新上下文中的bean工厂,刷新主要包括通过CAS设置bean工厂中的refreshed状态为true,并设置bean工厂的序列化id __serializationId__ ,此值为应用的名称,即 __spring.application.name__ 指定的名称.  
&emsp;&emsp;最后并返回刷新后的bean工厂即可.  

3.第三步prepareBeanFactory相关源码如下:

``` java
/**
 * Configure the factory's standard context characteristics,
 * such as the context's ClassLoader and post-processors.
 * 配置工厂的标准上下文特征,例如上下文的类加载和后置处理器
 */
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // Tell the internal bean factory to use the context's class loader etc.
    // 告诉内部bean工厂使用上下文的类加载器等 首先设置类加载器
    beanFactory.setBeanClassLoader(getClassLoader());
    // 然后根据类加载器创建StandardBeanExpressionResolver 它的作用是解析#{}表达式通常配合@Value注解使用  #{}用于执行SpEL表达式,并将内容赋值给属性 @{}用于加载外部属性文件中的值 两者可以混合使用
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    // 设置自定义property编辑注册器 主要是解析application.properties中的属性
    // 操作如下 https://blog.csdn.znet/qq_43414291/article/details/111226055
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // Configure the bean factory with context callbacks.
    // 使用上下文回调设置bean工厂
    //在bean工厂中添加ApplicationContextAwareProcessor后置处理器  他的主要作用是在初始化bean之前 将所有实现了Aware相关接口的属性设置到bean中
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    //忽略指定接口的自动装配功能
    //主要目的:spring是懒加载,当类A中有属性B时,从容器中获取A对象时会判断属性B是否初始化,没有初始化的话会自动初始化B.
	//			然而有些情况是不希望初始化B的.例如B实现了BeanNameAware、BeanFactoryAware、BeanClassLoaderAware接口(该接口的作用是获取其在容器中的名称、容器、类加载器)
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // BeanFactory interface not registered as resolvable type in a plain factory.
    // MessageSource registered (and found for autowiring) as a bean.
    //使用相应的自动装配值注册一个特殊的依赖类型,后面若需要自动装配该类型直接使用接口
    //registerResolvableDependency作用: 在spring自动装配的时候如果一个接口有多个实现类，并且都已经放到IOC中，那么自动装配会出现异常，spring不知道把哪个实现类注入进入。
    //									指定该类型接口，如果外部要注入该类型接口的对象，则会注入我们指定的对象，而不会去管其他接口实现类
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // Register early post-processor for detecting inner beans as ApplicationListeners.
    //注册早期后置处理器 为了检测内部bean作为 ApplicationListener
    //ApplicationListenerDetector的作用就是检测所有实现了ApplicationListener接口的单例并添加到上下文中
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // Detect a LoadTimeWeaver and prepare for weaving, if found.
    //如果有的话, 检测LoadTimeWeaver并准备织入
    //判断bean工厂中是否包含loadTimeWeaver
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        //如果包含的话注册LoadTimeWeaverAwareProcessor后置处理器
        //它的作用是 在bean初始化前 将默认的LoadTimeWeaver传递给实现了LoadTimeWeaverAware接口的bean
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        //设置临时类加载器进行类型匹配
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // Register default environment beans.
    //如果bean工厂中不包含 environment的bean 则向bean工厂注册它
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    //如果bean工厂中不包含 systemProperties的bean  直接向bean工厂注册它
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    //如果bean工厂中不包含systemEnvironment的bean  直接向bean工厂注册它
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
}
```
&emsp;&emsp;可见第三步的作用就是设置bean工厂的通用功能.例如设置类加载器、设置 __BeanExpressionResolver__ 用来解析 __#{}__ 属性、设置 __PropertyEditorRegistrar__ 来自定义解析 __application.properties__ 中的值.  
添加 __ApplicationContextAwareProcessor__ 后置处理器在bean初始化前设置实现所有相关Aware接口的属性同时要将这些Aware接口忽略防止进行自动装配, 既然有忽略那也就有注册,注册 __BeanFactory、ResourceLoader、ApplicationEventPublisher、ApplicationContext__ 为已解析类型,当遇到这几种类型直接使用即可.  
&emsp;&emsp;然后在检查bean工厂中是否有 __LoadTimeWeaver__ 的bean,有则注册 __LoadTimeWeaverAwareProcessor__ 后置处理器用来将默认的 __LoadTimeWeaver__ 传递给实现了 __LoadTimeWeaverAware__ 接口的bean并设置临时类加载器以进行类型匹配.  
&emsp;&emsp;最后就是检测 __environment、systemProperties、systemEnvironment__ 这三个bean是否存在,存在则进行注册.  
&emsp;&emsp;以上步骤执行完后, 一个bean工厂也就可以准备使用了.  

4.第四步postProcessBeanFactory相关源码如下:

``` java
//org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext#postProcessBeanFactory
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    //首先继续调用父类的postProcessBeanFactory方法继续处理bean工厂
    super.postProcessBeanFactory(beanFactory);
    //如果指定扫描路径则进行扫描 默认为null
    if (!ObjectUtils.isEmpty(this.basePackages)) {
        this.scanner.scan(this.basePackages);
    }
    //如果注解类不为空则进行注册该bean  默认为空
    if (!this.annotatedClasses.isEmpty()) {
        this.reader.register(ClassUtils.toClassArray(this.annotatedClasses));
    }
}
```
&emsp;&emsp;可见直接调用了父类的postProcessBeanFactory方法, AnnotationConfigServletWebServerApplicationContext的父类为ServletWebServerApplicationContext,那么继续看父类的代码:  
``` java
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    //注册WebApplicationContextServletContextAwareProcessor后置处理器
    //主要就是用来设置bean初始化之前设置servletContext和servletConfig
    beanFactory.addBeanPostProcessor(new WebApplicationContextServletContextAwareProcessor(this));
    //忽略ServletContextAware的自动装配功能
    beanFactory.ignoreDependencyInterface(ServletContextAware.class);
    //注册web应用范围 默认注册request和session
    registerWebApplicationScopes();
}

private void registerWebApplicationScopes() {
    //定义ExistingWebApplicationScopes 用来存储已经注册的范围
    ExistingWebApplicationScopes existingScopes = new ExistingWebApplicationScopes(getBeanFactory());
    //注册默认的request、session范围
    WebApplicationContextUtils.registerWebApplicationScopes(getBeanFactory());
    //将注册的范围保存到ExistingWebApplicationScopes
    existingScopes.restore();
}

/**
 * Register web-specific scopes ("request", "session", "globalSession", "application")
 * with the given BeanFactory, as used by the WebApplicationContext.
 * 使用WebApplicationContext使用的给定bean工厂注册特定于web的范围"request", "session", "globalSession", "application" 
 */
public static void registerWebApplicationScopes(ConfigurableListableBeanFactory beanFactory,
        @Nullable ServletContext sc) {
    //注册request、session
    beanFactory.registerScope(WebApplicationContext.SCOPE_REQUEST, new RequestScope());
    beanFactory.registerScope(WebApplicationContext.SCOPE_SESSION, new SessionScope());
    //如果ServletContext不为空 则注册 globalSession、application
    if (sc != null) {
        ServletContextScope appScope = new ServletContextScope(sc);
        beanFactory.registerScope(WebApplicationContext.SCOPE_APPLICATION, appScope);
        // Register as ServletContext attribute, for ContextCleanupListener to detect it.
        sc.setAttribute(ServletContextScope.class.getName(), appScope);
    }
    //使用相应的自动装配值注册一个特殊的依赖类型,后面若需要自动装配该类型直接使用接口
    beanFactory.registerResolvableDependency(ServletRequest.class, new RequestObjectFactory());
    beanFactory.registerResolvableDependency(ServletResponse.class, new ResponseObjectFactory());
    beanFactory.registerResolvableDependency(HttpSession.class, new SessionObjectFactory());
    beanFactory.registerResolvableDependency(WebRequest.class, new WebRequestObjectFactory());
    if (jsfPresent) {
        FacesDependencyRegistrar.registerFacesDependencies(beanFactory);
    }
}
```
&emsp;&emsp;可见父类ServletWebServerApplicationContext的作用就是注册一个 __WebApplicationContextServletContextAwareProcessor__ 后置处理器来在bean初始化之前设置 __ServletContext和ServletConfig__ ,然后就是注册web指定的范围 __request、session__ .
由此可见第四步主要就是后置处理Bean工厂,具体实现在ServletWebServerApplicationContext中,结论如上.  

5.第五步invokeBeanFactoryPostProcessors,非常重要,个人认为核心逻辑都在这里,我们好好看看,相关源码如下:  

&emsp;&emsp;我们先抛出大体的结论,通过 __invokeBeanFactoryPostProcessors__ 方法处理,此方法中处理 __BeanDefinitionRegistryPostProcessor__ 和 __BeanFactoryPostProcessor__ 两个后置处理器,  
其中 __BeanDefinitionRegistryPostProcessor__ 中有一个 __ConfigurationClassPostProcessor__ ,它的作用就是解析所有的 __@Configuration__ 修饰的类,也就是主函数,并在解析的过程中通过 __@SpringBootApplication及其元注解__ 解析项目中指定包下的配置类.   

``` java
/**
 * Instantiate and invoke all registered BeanFactoryPostProcessor beans,
 * respecting explicit order if given.
 * 实例化且调用所有注册的bean工厂后置处理器bean,如果给定顺序的按显示顺序来
 * <p>Must be called before singleton instantiation.
 */
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    //真正调用bean工厂后置处理器的地方 解析所有的BeanDefinitionRegistryPostProcessor和BeanFactoryPostProcessor并按照PriorityOrdered、Ordered和默认 三种顺序执行
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

    // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
    // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
    //再次检测扫描是否包含LoadTimeWeaver  例如有可能通过ConfigurationClassPostProcessor类解析@Bean注解 注册 LoadTimeWeaver的bean
    if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }
}

//解析所有的BeanDefinitionRegistryPostProcessor和BeanFactoryPostProcessor并按照PriorityOrdered、Ordered和默认 三种顺序执行
public static void invokeBeanFactoryPostProcessors(
        ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
    //入参beanFactoryPostProcessors有以下三个:
    //1.org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer$CachingMetadataReaderFactoryPostProcessor
    //  实现BeanDefinitionRegistryPostProcessor,通过postProcessBeanDefinitionRegistry回调接口注册internalCachingMetadataReaderFactory用来缓存类的元数据,并将该属性metadataReaderFactory 添加到internalConfigurationAnnotationProcessor中
    //2.org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer$ConfigurationWarningsPostProcessor
    //  实现BeanDefinitionRegistryPostProcessor,通过postProcessBeanDefinitionRegistry回调接口来报告警告 不是特别重要
    //3.org.springframework.boot.context.config.ConfigFileApplicationListener$PropertySourceOrderingPostProcessor
    //  实现BeanFactoryPostProcessor,通过postProcessBeanFactory来找到environment中的defaultProperties,并将其添加到属性源末尾  通常defaultProperties为null 所以无需处理 

    // Invoke BeanDefinitionRegistryPostProcessors first, if any.
    //如果有的话首先调用BeanDefinitionRegistryPostProcessors  它的作用是在BeanFactoryPostProcessor生效前注册进一步的bean定义
    Set<String> processedBeans = new HashSet<>();

    //BeanDefinitionRegistry是BeanFactoryPostProcessor的子类,所以我们肯定要先处理BeanDefinitionRegistry
    //前置处理逻辑
    //判断当前bean工厂是否属于BeanDefinitionRegistry  不属于直接调用注册在上下文中的BeanFactoryPostProcessor 否则继续处理
    //定义两个regularPostProcessors和registryProcessors分别存放两中后置处理器 肯定是要先执行子类BeanDefinitionRegistryPostProcessor
    //遍历上下文中的后置处理器 遇到BeanDefinitionRegistryPostProcessor直接执行postProcessBeanDefinitionRegistry回调并将其添加到registryProcessors集合中
    //                      遇到BeanFactoryPostProcessor则直接添加到regularPostProcessors集合中
    //接下来就是处理BeanDefinitionRegistryPostProcessor的正文了:
    //1. 首先从bean工厂已注册的bean定义中找到实现BeanDefinitionRegistryPostProcessor的bean  
    //   目前只有一个就是ConfigurationClassPostProcessor, 我们再仔细想想这个是不是很眼熟,它又是在何时注册为bean定义的, 
    //   前面我们其实介绍过就是在创建AnnotationConfigServletWebServerApplicationContext上下文构造时的AnnotatedBeanDefinitionReader中的AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry)方法,它会注册ConfigurationClassPostProcessor的bean定义到上下文中
    //2. 既然找到了ConfigurationClassPostProcessor,判断它是否实现了PriorityOrdered,实现了的话则通过getBean方法实例化该bean,并添加到currentRegistryProcessors集合中 getBean方法也是Spring中非常重要的方法,后面我们再介绍
    //3. 实例化完指定bean后将currentRegistryProcessors进行排序并将已经集合中的所有元素添加到registryProcessors集合中
    //4. 最重要的一步 执行所有的BeanDefinitionRegistryPostProcessor后置处理器的postProcessBeanDefinitionRegistry, 这里主要是执行ConfigurationClassPostProcessor的回调, 它的主要作用就是解析主函数 逻辑是非常重要的 后面我们慢慢分析
    //5. 执行完之后将currentRegistryProcessors清空
    //6. 再次按照步骤1-5的顺序 将所有BeanDefinitionRegistryPostProcessor的bean按照Ordered和默认顺序进行处理即可  
    //7. 最后依次执行registryProcessors和regularPostProcessors中的所有后置处理器的postProcessBeanFactory方法
    if (beanFactory instanceof BeanDefinitionRegistry) {
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
        List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
        List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

        //预先执行上下文实例中注册的后置处理器
        for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                BeanDefinitionRegistryPostProcessor registryProcessor =
                        (BeanDefinitionRegistryPostProcessor) postProcessor;
                registryProcessor.postProcessBeanDefinitionRegistry(registry);
                registryProcessors.add(registryProcessor);
            }
            else {
                regularPostProcessors.add(postProcessor);
            }
        }

        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // uninitialized to let the bean factory post-processors apply to them!
        // Separate between BeanDefinitionRegistryPostProcessors that implement
        // PriorityOrdered, Ordered, and the rest.
        //这里不实例化化工厂bean: 我们需要让所有常规bean未初始化以便让bean工厂后置处理器应用他们
        //将实现PriorityOrdered、Ordered和其余部分的BeanDefinitionRegistryPostProcessors分开
        List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

        // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
        //首先调用实现了PriorityOrdered接口的BeanDefinitionRegistryPostProcessors
        String[] postProcessorNames =
                beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();

        // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
        //接着调用实现了Ordered接口的BeanDefinitionRegistryPostProcessors
        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();

        // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
        //最后调用剩下的BeanDefinitionRegistryPostProcessors直到没有
        boolean reiterate = true;
        while (reiterate) {
            reiterate = false;
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                if (!processedBeans.contains(ppName)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                    reiterate = true;
                }
            }
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            currentRegistryProcessors.clear();
        }

        // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
        //调用目前为止处理的所有后置处理器的 postProcessBeanFactory方法
        invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
        invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
    }

    else {
        // Invoke factory processors registered with the context instance.
        // 调用通过上下文实例注册的工厂处理器
        invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
    }

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let the bean factory post-processors apply to them!
    //这里不实例化工厂bean:我们需要让所有常规bean不实例化以便让bean工厂后置处理器应用他们
    
    //上下文中主要有这6个后置处理器:
    //1. org.springframework.context.annotation.internalConfigurationAnnotationProcessor
    //   处理配置类 第一阶段以及处理过 无需再处理
    //2. org.springframework.context.event.internalEventListenerProcessor
    //   主要作用是设置实现EventListenerFactory接口的bean
    //3. propertySourcesPlaceholderConfigurer
    //   它的作用就是解析 ${} 占位符的值和@Value 在Environment和PropertySources中的值
    //4. org.springframework.boot.context.properties.ConfigurationPropertiesBeanDefinitionValidator
    //   主要作用是验证常规bean定义没有创建 ConstructorBinding的bean
    //5. preserveErrorControllerTargetClassPostProcessor
    //   创建ErrorController相关的bean定义并设置preserveTargetClass属性为true
    //6. emBeanDefinitionRegistrarPostProcessor
    //   主要是处理jpa相关属性  无需关心

    //这里就是处理BeanFactoryPostProcessor的逻辑,大致逻辑与处理BeanDefinitionRegistryPostProcessor类似
    //1. 首先从bean工厂的定义中找到实现BeanFactoryPostProcessor接口的所有bean定义
    //2. 接着定义三个集合 priorityOrderedPostProcessors、orderedPostProcessorNames、nonOrderedPostProcessorNames 分别存放实现了 PriorityOrdered、Ordered、默认顺序的bean
    //   这里需要注意 只有实现了PriorityOrdered的bean才会立即实例化
    //3. 首先还是先调用实现了PriorityOrdered接口的后置处理器的postProcessBeanFactory接口
    //4. 接着先获取所有实现了Ordered接口的后置处理器的bean,接着调用postProcessBeanFactory接口
    //5. 按步骤4处理剩下的后置处理器
    //6. 最后清空bean工厂中的元数据缓存
    String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

    // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
    // Ordered, and the rest.
    //将实现PriorityOrdered、Ordered和剩余部分的BeanFactoryPostProcessor分开
    List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    List<String> orderedPostProcessorNames = new ArrayList<>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();
    for (String ppName : postProcessorNames) {
        if (processedBeans.contains(ppName)) {
            // skip - already processed in first phase above
            //第一阶段处理过的直接跳过
        }
        else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
        }
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        }
        else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }

    // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
    //首先调用实现了PriorityOrdered接口的BeanFactoryPostProcessors
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

    // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
    //接着 调用实现了Ordered接口的BeanFactoryProcessors
    List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
    for (String postProcessorName : orderedPostProcessorNames) {
        orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

    // Finally, invoke all other BeanFactoryPostProcessors.
    //最后 调用所有剩下的BeanFactoryPostProcessor
    List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
    for (String postProcessorName : nonOrderedPostProcessorNames) {
        nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

    // Clear cached merged bean definitions since the post-processors might have
    // modified the original metadata, e.g. replacing placeholders in values...
    //清除可能由于后置处理器修改原始元数据的缓存的合并bean定义
    beanFactory.clearMetadataCache();
}
```
我们就 __invokeBeanFactoryPostProcessors__ 先总结一下:  
&emsp;&emsp;看方法名,就是调用实现 __BeanFactoryPostProcessor__ 接口的所有后置处理器,先看一下我们这里需要处理的后置处理器的类图,有些内部类不好合并就不展示了:  
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/BeanFactoryPostProcessor.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="BeanFactoryPostProcessor类图"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/BeanFactoryPostProcessor.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">BeanFactoryPostProcessor类图</div>
    </a>
</center>
&emsp;&emsp;可见我们这里除了处理实现了BeanFactoryPostProcessor的后置处理器,还处理了实现BeanDefinitionRegistryPostProcessor的后置处理器,其实BeanDefinitionRegistryPostProcessor也是实现了BeanFactoryPostProcessor,从类图中可以看出.  
&emsp;&emsp;那我们这里的逻辑就很清晰了,首先处理实现BeanDefinitionRegistryPostProcessor的后置处理器,通过postProcessBeanDefinitionRegistry回调方法,然后是处理实现了BeanFactoryPostProcessor的后置处理器,通过postProcessBeanFactory回调方法.每个后置处理器都有自己的逻辑,需具体分析.  

>invokeBeanFactoryPostProcessors小结  

前置处理逻辑:  
&emsp;&emsp;判断当前bean工厂是否属于BeanDefinitionRegistry,不属于直接调用注册在上下文中的BeanFactoryPostProcessor,否则继续处理.  
&emsp;&emsp;定义两个regularPostProcessors和registryProcessors分别存放两种后置处理器,肯定是要先执行子类BeanDefinitionRegistryPostProcessor
遍历上下文中的后置处理器,遇到BeanDefinitionRegistryPostProcessor直接执行postProcessBeanDefinitionRegistry回调并将其添加到registryProcessors集合中,遇到BeanFactoryPostProcessor则直接添加到regularPostProcessors集合中.    
第一阶段处理BeanDefinitionRegistryPostProcessor:  
1. 首先从bean工厂已注册的bean定义中找到实现BeanDefinitionRegistryPostProcessor的bean  
   目前只有一个就是ConfigurationClassPostProcessor, 我们再仔细想想这个是不是很眼熟,它又是在何时注册为bean定义的, 
   前面我们其实介绍过就是在创建AnnotationConfigServletWebServerApplicationContext上下文构造时的AnnotatedBeanDefinitionReader中的AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry)方法,它会注册ConfigurationClassPostProcessor的bean定义到上下文中
2. 既然找到了ConfigurationClassPostProcessor,判断它是否实现了PriorityOrdered,实现了的话则通过getBean方法实例化该bean,并添加到currentRegistryProcessors集合中 getBean方法也是Spring中非常重要的方法,后面我们再介绍
3. 实例化完指定bean后将currentRegistryProcessors进行排序并将已经集合中的所有元素添加到registryProcessors集合中
4. 最重要的一步 执行所有的BeanDefinitionRegistryPostProcessor后置处理器的postProcessBeanDefinitionRegistry, 这里主要是执行ConfigurationClassPostProcessor的回调, 它的主要作用就是解析主函数 逻辑是非常重要的 后面我们慢慢分析
5. 执行完之后将currentRegistryProcessors清空
6. 再次按照步骤1-5的顺序 将所有BeanDefinitionRegistryPostProcessor的bean按照Ordered和默认顺序进行处理即可  
7. 最后依次执行registryProcessors和regularPostProcessors中的所有后置处理器的postProcessBeanFactory方法


第二阶段处理BeanFactoryPostProcessor,大致逻辑与处理BeanDefinitionRegistryPostProcessor类似:  
1. 首先从bean工厂的定义中找到实现BeanFactoryPostProcessor接口的所有bean定义
2. 接着定义三个集合 priorityOrderedPostProcessors、orderedPostProcessorNames、nonOrderedPostProcessorNames 分别存放实现了 PriorityOrdered、Ordered、默认顺序的bean
   这里需要注意 只有实现了PriorityOrdered的bean才会立即实例化
3. 首先还是先调用实现了PriorityOrdered接口的后置处理器的postProcessBeanFactory接口
4. 接着先获取所有实现了Ordered接口的后置处理器的bean,接着调用postProcessBeanFactory接口
5. 按步骤4处理剩下的后置处理器
6. 最后清空bean工厂中的元数据缓存

&emsp;&emsp;下面我们来着重看下最重要的后置处理器 __ConfigurationClassPostProcessor__ 它的作用是在启动时通过 __postProcessBeanDefinitionRegistry__ 回调处理@Configuration修饰的类.
__postProcessBeanDefinitionRegistry__ 回调源码如下:
``` java
/**
 * Derive further bean definitions from the configuration classes in the registry.
 * 从在注册表中的配置类派生出进一步的bean定义
 */
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    //获取该注册表的哈希值 判断是否已经处理过 若处理过直接抛出异常 未处理则加入已注册集合registriesPostProcessed
    int registryId = System.identityHashCode(registry);
    if (this.registriesPostProcessed.contains(registryId)) {
        throw new IllegalStateException(
                "postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
    }
    if (this.factoriesPostProcessed.contains(registryId)) {
        throw new IllegalStateException(
                "postProcessBeanFactory already called on this post-processor against " + registry);
    }
    this.registriesPostProcessed.add(registryId);
    //处理注册表中所有被@Configuration修饰的bean定义
    processConfigBeanDefinitions(registry);
}
```
可见回调中其实就做了两件事:
1. 获取该注册表的hash值并判断是否处理过,若已处理则直接抛出异常,否则将此值添加到已注册结合registriesPostProcessed
2. 解析注册表中所有被@Configuration修饰的bean定义  
  
接下来我们在继续看看这个 __processConfigBeanDefinitions(registry)__ 的源码:
``` java
/**
 * Build and validate a configuration model based on the registry of
 * {@link Configuration} classes.
 * 基于Configuration类的注册表构建和验证配置模型
 */
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    //1.获取注册表中已注册的bean定义,并挑选出配置类候选(使用@Configuration注解的类)并加入到configCandidates中
    List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
    String[] candidateNames = registry.getBeanDefinitionNames();

    //从前面加载的候选者中找到被@Configuration修饰的类并把该类封装为BeanDefinitionHolder
    //主要是处理主函数类
    for (String beanName : candidateNames) {
        BeanDefinition beanDef = registry.getBeanDefinition(beanName);
        if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
            if (logger.isDebugEnabled()) {
                logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
            }
        }
        else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
            //将匹配的bean添加到配置候选者中
            configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
        }
    }

    // Return immediately if no @Configuration classes were found
    //如果没有发现@Configuration配置的类直接返回
    if (configCandidates.isEmpty()) {
        return;
    }
    
    //2. 解析前置操作:根据@Order排序候选者、设置bean名称生成策略、设置Environment、最重要的就是创建ConfigurationClassParser解析器
    // Sort by previously determined @Order value, if applicable
    //如果适用的话,根据先前确定的@Order的值排序  order值越小优先级越高
    configCandidates.sort((bd1, bd2) -> {
        int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
        int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
        return Integer.compare(i1, i2);
    });

    // Detect any custom bean name generation strategy supplied through the enclosing application context
    //检测 通过封闭应用上下文提供的任何自定义bean名称生成策略
    SingletonBeanRegistry sbr = null;
    if (registry instanceof SingletonBeanRegistry) {
        sbr = (SingletonBeanRegistry) registry;
        if (!this.localBeanNameGeneratorSet) {
            BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(
                    AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR);
            if (generator != null) {
                this.componentScanBeanNameGenerator = generator;
                this.importBeanNameGenerator = generator;
            }
        }
    }

    if (this.environment == null) {
        this.environment = new StandardEnvironment();
    }

    // Parse each @Configuration class
    //解析每一个@Configuration修饰的类
    ConfigurationClassParser parser = new ConfigurationClassParser(
            this.metadataReaderFactory, this.problemReporter, this.environment,
            this.resourceLoader, this.componentScanBeanNameGenerator, registry);
    
    //3. 通过ConfigurationClassParser的parse方法解析候选类 也就是主函数
    //3.1 首先通过parse方法进行解析主函数,这里就将所有的注解都解析的干干净净 具体逻辑稍后分析
    //3.1 解析完后进行验证,主要验证解析后配置类及其中的属性是否合法
    //3.3 此时已经将所有的配置类都解析好了,但是还有通过@Import或@Bean或@ImportedResources或ImportBeanDefinitionRegister的bean未注册,这步就是注册这些类的
    //3.4 最后就是筛选出所有已经真正解析过的配置类 若存在未解析的加入到candidates中继续进行解析
    Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
    Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
    do {
        //使用ConfigurationClassParser解析每一个配置候选者
        //即解析主函数 主要作用是 解析每个注解
        //1.@PropertySource
        //2.@ComponentScan 根据注解的路径扫描所有需要加载的配置类  默认是扫描主类下的所有文件
        //3.@Import 根据@Import注解加载所有的导入类  最重要的是通过SpringBoot中的AutoConfigurationImportSelector来加载所有自动配置的类
        //4.@ImportResource
        //5.@Bean
        //6.default methods on interfaces
        //7.Process superclass
        parser.parse(candidates);
        parser.validate();

        //4. 通过reader.loadBeanDefinitions方法注册导入的配置类自身、相关@Bean方法导入的类、@ImportResource导入的类和@Import导入的类的定义
        Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
        configClasses.removeAll(alreadyParsed);

        // Read the model and create bean definitions based on its content
        //读取模型并根据其内容创建bean定义
        if (this.reader == null) {
            this.reader = new ConfigurationClassBeanDefinitionReader(
                    registry, this.sourceExtractor, this.resourceLoader, this.environment,
                    this.importBeanNameGenerator, parser.getImportRegistry());
        }
        //将解析过的配置类中的一些属性注册为bean定义
        this.reader.loadBeanDefinitions(configClasses);
        //alreadyParsed存放所有实例化过的配置类
        alreadyParsed.addAll(configClasses);

        candidates.clear();
        
        //5.最后检测后扫描到的配置类中是否有未扫描到的,存在则加入到candidates中继续解析
        //如果注册表中的bean定义数量 大于 之前 加载的候选类数量
        if (registry.getBeanDefinitionCount() > candidateNames.length) {
            //最新的候选配置类数组
            String[] newCandidateNames = registry.getBeanDefinitionNames();
            //旧的候选配置类集合
            Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
            Set<String> alreadyParsedClasses = new HashSet<>();
            //添加到alreadyParsedClasses中
            for (ConfigurationClass configurationClass : alreadyParsed) {
                alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
            }
            //比较新旧候选类  只有不存在于旧的候选者集合 且 属于配置候选者 才添加到candidates
            for (String candidateName : newCandidateNames) {
                if (!oldCandidateNames.contains(candidateName)) {
                    BeanDefinition bd = registry.getBeanDefinition(candidateName);
                    if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
                            !alreadyParsedClasses.contains(bd.getBeanClassName())) {
                        candidates.add(new BeanDefinitionHolder(bd, candidateName));
                    }
                }
            }
            candidateNames = newCandidateNames;
        }
    }
    while (!candidates.isEmpty());

    // Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
    //将ImportRegistry注册为一个bean 为了支持ImportAware的配置类
    if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
        sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
    }

    if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
        // Clear cache in externally provided MetadataReaderFactory; this is a no-op
        // for a shared cache since it'll be cleared by the ApplicationContext.
        //清除外部提供的MetadataReaderFactory中的缓存; 这是共享缓存的空操作,因为它将被ApplicationContext清除
        ((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
    }
}
```
由此可见 __processConfigBeanDefinitions__ 主要分为五步:  
1. 获取注册表中已注册的bean定义,并挑选出配置类候选(使用@Configuration注解的类)并加入到configCandidates中
2. 解析前置操作:根据@Order排序候选者、设置bean名称生成策略、设置Environment、最重要的就是创建ConfigurationClassParser解析器
3. 通过ConfigurationClassParser的parse方法解析候选类 也就是主函数(核心逻辑),总结来说就是解析配置类上的所有注解属性并将这些bean添加到bean工厂中 (详细分析见下章)
4. 通过reader.loadBeanDefinitions方法注册导入的配置类自身、相关@Bean方法导入的类、@ImportResource导入的类和@Import导入的类的定义
5. 最后检测后扫描到的配置类中是否有未扫描到的,存在则加入到candidates中继续解析
- - -

### 基本思路
- - -
- - -

### 遍历图解
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/图片地址.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="图片描述"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/图片地址.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">图片描述</div>
    </a>
</center>



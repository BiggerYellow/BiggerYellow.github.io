---
layout: default
springboot: true
modal-id: 20000005
date: 2021-09-10
img: pexels-pok-rie-136728.jpg
alt: SpringBoot
project-date: September 2021
client: SpringBoot
category: SpringBoot
subtitle: SpringBoot启动源码解析(五)
description: SpringBootApplication Run方法解析(三):prepareContext
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

### prepareContext
- - -
本篇我们一起来看下 __prepareContext__ 方法主要做了什么:  
第五步主要就是根据前面创建的 __environment、listeners、applicationArguments、printedBanner__ 参数准备创建的上下文 __context__ ,相关源码如下:

``` java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
        SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
    //5.1为上下文设置environment
    context.setEnvironment(environment);
    //处理任何相关的后置处理,主要是设置 beanNameGenerator、resourceLoader、addConversionService
    postProcessApplicationContext(context);
    //5.2应用所有在构造函数阶段初始化的ApplicationContextInitializer
    applyInitializers(context);
    //5.3发布环境已准备事件
    listeners.contextPrepared(context);
    //5.4打印启动信息
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    // Add boot specific singleton beans
    //向bean工厂中注册springApplicationArguments和springBootBanner 两个单例
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    //设置是否允许bean定义被覆盖,默认为false
    if (beanFactory instanceof DefaultListableBeanFactory) {
        ((DefaultListableBeanFactory) beanFactory)
                .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    //如果应用设置为懒加载则添加懒加载后置处理器, 默认为false不用添加
    if (this.lazyInitialization) {
        context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
    }
    // Load the sources
    //5.5加载资源,这里其实就是换的启动函数,并将它注册为bean
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    load(context, sources.toArray(new Object[0]));
    //5.6发布上下文已加载事件
    listeners.contextLoaded(context);
}
```

5.1 首先看第一步相关的源码
``` java
/**
 * Delegates given environment to underlying {@link AnnotatedBeanDefinitionReader} and
 * {@link ClassPathBeanDefinitionScanner} members.
 * 委托给定的环境给底层的AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner成员
 */
@Override
public void setEnvironment(ConfigurableEnvironment environment) {
    super.setEnvironment(environment);
    this.reader.setEnvironment(environment);
    this.scanner.setEnvironment(environment);
}

/**
 * Apply any relevant post processing the {@link ApplicationContext}. Subclasses can
 * apply additional processing as required.
 * 应用相关的后处理应用上下文. 子类可以根据需要应用额外的处理
 */
protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
    //如果应用中存在beanNameGenerator则注册internalConfigurationBeanNameGenerator bean定义
    if (this.beanNameGenerator != null) {
        context.getBeanFactory().registerSingleton(AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
                this.beanNameGenerator);
    }
    //如果存在资源加载器则设置到上下文中
    if (this.resourceLoader != null) {
        if (context instanceof GenericApplicationContext) {
            ((GenericApplicationContext) context).setResourceLoader(this.resourceLoader);
        }
        if (context instanceof DefaultResourceLoader) {
            ((DefaultResourceLoader) context).setClassLoader(this.resourceLoader.getClassLoader());
        }
    }
    //如果addConversionService为true表明需要设置类型转换服务
    if (this.addConversionService) {
        context.getBeanFactory().setConversionService(ApplicationConversionService.getSharedInstance());
    }
}
``` 
首先看 __setEnvironment__ 方法,其中两个成员是不是很熟悉, 就是在初始化 __AnnotationConfigServletWebServerApplicationContext__ 上下文时创建的成员类,那么本方法也比较通俗易懂了,无非就是设置环境信息.  
然后我们再看 __postProcessApplicationContext__ ,它的主要作用就是设置了 __ConversionService__ 服务.

5.2 第二步就是应用所有 __ApplicationContextInitializer__ ,相关源码如下:
``` java
/**
 * Apply any {@link ApplicationContextInitializer}s to the context before it is
 * refreshed.
 * 在应用刷新前应用所有ApplicationContextInitializer到上下文中
 */
@SuppressWarnings({ "rawtypes", "unchecked" })
protected void applyInitializers(ConfigurableApplicationContext context) {
    for (ApplicationContextInitializer initializer : getInitializers()) {
        Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(initializer.getClass(),
                ApplicationContextInitializer.class);
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        initializer.initialize(context);
    }
}
```
第二步就是在获取所有在 __SpringApplication__ 构造函数中初始化的 __ApplicationContextInitializer__ ,并执行他们各自实现的 __initialize()__ 方法,如果我们自定义 __ApplicationContextInitializer__ 的话,也是在此处执行我们的类.  

5.3 第三步发布上下文准备完成事件
``` java
void contextPrepared(ConfigurableApplicationContext context) {
    for (SpringApplicationRunListener listener : this.listeners) {
        listener.contextPrepared(context);
    }
}
```

5.4 第四步主要是执行一些上下文的准备动作,打印日志以及注册相关bean定义等. 
``` java
if (this.logStartupInfo) {
    logStartupInfo(context.getParent() == null);
    logStartupProfileInfo(context);
}
// Add boot specific singleton beans
//向bean工厂中注册springApplicationArguments和springBootBanner 两个单例
ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
if (printedBanner != null) {
    beanFactory.registerSingleton("springBootBanner", printedBanner);
}
//设置是否允许bean定义被覆盖,默认为false
if (beanFactory instanceof DefaultListableBeanFactory) {
    ((DefaultListableBeanFactory) beanFactory)
            .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
}
//如果应用设置为懒加载则添加懒加载后置处理器, 默认为false不用添加
if (this.lazyInitialization) {
    context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
}
```
主要就是向bean工厂中注册 __springApplicationArguments、springBootBanner__ bean定义以及设置 __allowBeanDefinitionOverriding__ 属性.  

5.5 第五步是本章最重要的部分,即以主函数创建bean定义并注册到bean工厂中
``` java
// Load the sources
//5.5加载资源,这里其实就是换的启动函数,并将它注册为bean
Set<Object> sources = getAllSources();
Assert.notEmpty(sources, "Sources must not be empty");
load(context, sources.toArray(new Object[0]));
```

第五步其实就分为两个方法,获取主函数 __getAllSources__ 并加载该主函数 __load__ . 
5.5.1 首先我们来看一下获取主函数 __getAllSources__:
``` java
/**
 * Return an immutable set of all the sources that will be added to an
 * ApplicationContext when {@link #run(String...)} is called. This method combines any
 * primary sources specified in the constructor with any additional ones that have
 * been {@link #setSources(Set) explicitly set}.
 * 当调用run时, 返回将应用上下文的所有源的不可变集合. 
 * 此方法将构造函数中指定的任何主要来源与已显示设置的任何其他来源结合起来
 */
public Set<Object> getAllSources() {
    Set<Object> allSources = new LinkedHashSet<>();
    //这个primarySources就是我们的主函数
    if (!CollectionUtils.isEmpty(this.primarySources)) {
        allSources.addAll(this.primarySources);
    }
    if (!CollectionUtils.isEmpty(this.sources)) {
        allSources.addAll(this.sources);
    }
    return Collections.unmodifiableSet(allSources);
}
```
该方法作用就是返回主函数的不可变集合.  
  
5.5.2接下来我们再看看最核心的加载方法 __load__ :
``` java
/**
 * Load beans into the application context.
 * 加载bean到应用上下文中
 */
protected void load(ApplicationContext context, Object[] sources) {
    if (logger.isDebugEnabled()) {
        logger.debug("Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
    }
    //根据主函数常创建BeanDefinitionLoader,BeanDefinitionLoader的作用是注册bean定义
    BeanDefinitionLoader loader = createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources);
    //设置loader相关的属性
    if (this.beanNameGenerator != null) {
        loader.setBeanNameGenerator(this.beanNameGenerator);
    }
    if (this.resourceLoader != null) {
        loader.setResourceLoader(this.resourceLoader);
    }
    if (this.environment != null) {
        loader.setEnvironment(this.environment);
    }
    //注册bean定义
    loader.load();
}
```
可见load方法中又可以分为两步,根据主函数创建BeanDefinitionLoader和调用该BeanDefinitionLoader的load方法加载bean.  
先看创建BeanDefinitionLoader的方法:
``` java
/**
 * Factory method used to create the {@link BeanDefinitionLoader}.
 * 创建BeanDefinitionLoader的工厂方法
 */
protected BeanDefinitionLoader createBeanDefinitionLoader(BeanDefinitionRegistry registry, Object[] sources) {
    return new BeanDefinitionLoader(registry, sources);
}

/**
 * Create a new {@link BeanDefinitionLoader} that will load beans into the specified
 * {@link BeanDefinitionRegistry}.
 * 创建一个新的BeanDefinitionLoader, 它将bean加载到指定的BeanDefinitionRegistry
 */
BeanDefinitionLoader(BeanDefinitionRegistry registry, Object... sources) {
    Assert.notNull(registry, "Registry must not be null");
    Assert.notEmpty(sources, "Sources must not be empty");
    this.sources = sources;
    this.annotatedReader = new AnnotatedBeanDefinitionReader(registry);
    this.xmlReader = new XmlBeanDefinitionReader(registry);
    if (isGroovyPresent()) {
        this.groovyReader = new GroovyBeanDefinitionReader(registry);
    }
    this.scanner = new ClassPathBeanDefinitionScanner(registry);
    this.scanner.addExcludeFilter(new ClassExcludeFilter(sources));
}
```
根据主函数和注册表进行创建BeanDefinitionLoader,并初始化AnnotatedBeanDefinitionReader和ClassPathBeanDefinitionScanner

最后我们来下最重要的load方法:
``` java
//加载主资源
private int load(Object source) {
    Assert.notNull(source, "Source must not be null");
    //我们的主函数肯定是个类  所以走load类的方法
    if (source instanceof Class<?>) {
        return load((Class<?>) source);
    }
    ...
    throw new IllegalArgumentException("Invalid source type " + source.getClass());
}

private int load(Class<?> source) {
    //查看是否支持 groovy.lang.MetaClass 且 source是否为GroovyBeanDefinitionSource类或者是他的超类或接口
    if (isGroovyPresent() && GroovyBeanDefinitionSource.class.isAssignableFrom(source)) {
        // Any GroovyLoaders added in beans{} DSL can contribute beans here
        GroovyBeanDefinitionSource loader = BeanUtils.instantiateClass(source, GroovyBeanDefinitionSource.class);
        load(loader);
    }
    //判断当前类是否被@Component修饰,如果是则注册该资源
    if (isComponent(source)) {
        this.annotatedReader.register(source);
        return 1;
    }
    return 0;
}

private boolean isComponent(Class<?> type) {
    // This has to be a bit of a guess. The only way to be sure that this type is
    // eligible is to make a bean definition out of it and try to instantiate it.
    //这不得不有点猜测, 确保此类型符合条件的唯一方法就是从中创建bean定义并尝试实例化他
    if (MergedAnnotations.from(type, SearchStrategy.TYPE_HIERARCHY).isPresent(Component.class)) {
        return true;
    }
    // Nested anonymous classes are not eligible for registration, nor are groovy
    // closures
    // 嵌套匿名类不符合注册条件, groovy闭包也不符合注册条件
    return !type.getName().matches(".*\\$_.*closure.*") && !type.isAnonymousClass()
            && type.getConstructors() != null && type.getConstructors().length != 0;
}
```
由上可见, __doRegisterBean__ 之前都是进行类检测作用,主函数是否由 __@Component__ 修饰,且它是否支持 __groovy__ 语义.  
接下来我们继续看看它真正的 __doRegisterBean__ 方法.

``` java
/**
 * Register a bean from the given bean class, deriving its metadata from
 * class-declared annotations.
 * 从给定的bean类注册bean, 从类声明的注释中获取其元数据
 */
private <T> void doRegisterBean(Class<T> beanClass, @Nullable String name,
        @Nullable Class<? extends Annotation>[] qualifiers, @Nullable Supplier<T> supplier,
        @Nullable BeanDefinitionCustomizer[] customizers) {
    //定义注解类型的bean定义,来看下它的构造方法:
    //所以它的主要作用就是解析指定类的所有注解
    AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
    //检查当前bean定义是包含@Conditional且是否满足条件
    if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
        return;
    }

    abd.setInstanceSupplier(supplier);
    //解析适用于abd的 ScopeMetadata 并设置到 bean定义中
    ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
    abd.setScope(scopeMetadata.getScopeName());
    //生成bean名称
    String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

    //处理通用bea注解 主要处理@Lazy, @Primary, @DependsOn, @Role, @Description等注解
    AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
    //存在以上注解则设置bean属性
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
    //是否有自定义器 自定义 bean定义
    if (customizers != null) {
        for (BeanDefinitionCustomizer customizer : customizers) {
            customizer.customize(abd);
        }
    }
    //创建该bean 定义持有器
    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
    //设置当前bean定义的代理模式 根据proxyBeanMethods字段
    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    //注册bean定义
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
}

/**
 * Create a new AnnotatedGenericBeanDefinition for the given bean class.
 * 为给定的类创建一个新的 AnnotatedGenericBeanDefinition
 */
public AnnotatedGenericBeanDefinition(Class<?> beanClass) {
    setBeanClass(beanClass);
    //核心就是这步,它将解析给定的类的所有注解
    this.metadata = AnnotationMetadata.introspect(beanClass);
}

/**
 * Determine if an item should be skipped based on {@code @Conditional} annotations.
 * 根据 @Conditional注解确定是否应跳过项目
 */
public boolean shouldSkip(@Nullable AnnotatedTypeMetadata metadata, @Nullable ConfigurationPhase phase) {
    //判断该类的元注解是否包含 @Conditional  不包含直接返回false
    if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
        return false;
    }

    //当parse为null时, 尝试使用@Conditional解析@Configuration类 和 非@Configuration修饰的bean
    if (phase == null) {
        if (metadata instanceof AnnotationMetadata &&
                ConfigurationClassUtils.isConfigurationCandidate((AnnotationMetadata) metadata)) {
            return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
        }
        return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
    }

    List<Condition> conditions = new ArrayList<>();
    for (String[] conditionClasses : getConditionClasses(metadata)) {
        for (String conditionClass : conditionClasses) {
            Condition condition = getCondition(conditionClass, this.context.getClassLoader());
            conditions.add(condition);
        }
    }

    AnnotationAwareOrderComparator.sort(conditions);

    for (Condition condition : conditions) {
        ConfigurationPhase requiredPhase = null;
        if (condition instanceof ConfigurationCondition) {
            requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();
        }
        if ((requiredPhase == null || requiredPhase == phase) && !condition.matches(this.context, metadata)) {
            return true;
        }
    }

    return false;
}

//处理通用bean定义注解 
static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd, AnnotatedTypeMetadata metadata) {
    //首先检查 @Lazy注解
    AnnotationAttributes lazy = attributesFor(metadata, Lazy.class);
    if (lazy != null) {
        abd.setLazyInit(lazy.getBoolean("value"));
    }
    //判断当前bean定义的元注解中是否还有 @Lazy注解
    else if (abd.getMetadata() != metadata) {
        lazy = attributesFor(abd.getMetadata(), Lazy.class);
        if (lazy != null) {
            abd.setLazyInit(lazy.getBoolean("value"));
        }
    }

    //元数据是否以 @Primary为注解
    if (metadata.isAnnotated(Primary.class.getName())) {
        abd.setPrimary(true);
    }
    //元数据是否以 @DependsOn为注解
    AnnotationAttributes dependsOn = attributesFor(metadata, DependsOn.class);
    if (dependsOn != null) {
        abd.setDependsOn(dependsOn.getStringArray("value"));
    }

    //元数据是否以 @Role为注解
    AnnotationAttributes role = attributesFor(metadata, Role.class);
    if (role != null) {
        abd.setRole(role.getNumber("value").intValue());
    }
    //元数据是否以 @Description为注解
    AnnotationAttributes description = attributesFor(metadata, Description.class);
    if (description != null) {
        abd.setDescription(description.getString("value"));
    }
}
```
由此可见, __doRegisterBean__ 的作用就是解析主函数的相关源注解,并将其作为bean定义注册到bean工厂中,以供后续进行真正解析这个类.  
- - -

### prepareContext总结
- - -
可以大约分为6步:  
1. 设置 __environment__ 到上下文中,并执行相关的后置处理,设置类型装换服务
2. 应用在构造函数阶段实例化的所有 __ApplicationContextInitializer__ 初始化器到上下文
3. 此时上下文已经准备就绪,发布 __contextPrepared__ 事件
4. 打印启动信息,并注册 __springApplicationArguments、springBootBanner__ bean定义,并设置上下文的是否允许bean定义覆盖属性 __allowBeanDefinitionOverriding__
5. 第五步就是最重要的一步了,这里将进行对主函数注解的解析并校验,校验通过则注册到bean工厂中,供后面解析配置类处理
6. 最后一步就是发布上下文已加载事件
- - -


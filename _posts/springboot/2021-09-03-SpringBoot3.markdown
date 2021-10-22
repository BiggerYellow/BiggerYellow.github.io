---
layout: default
springboot: true
modal-id: 20000003
date: 2021-09-03
img: pexels-rachel-claire-5490361.jpg
alt: SpringBoot3
project-date: September 2021
client: SpringBoot
category: SpringBoot
subtitle: SpringBoot启动源码解析(三)
description: SpringBootApplication Run方法解析(一):prepareEnvironment
---
### Run方法流程图
- - -
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/run流程图.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="run流程图"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/run流程图.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">run流程图</div>
    </a>
</center>
- - -

### Run方法入口:主要分析prepareEnvironment
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
本章我们主要分析 __prepareEnvironment__ 及其之前的步骤.  
1. 第一步比较好理解,创建定时器统计启动时间,并获取的事件发布者 __SpringApplicationRunListener__ 来发布启动过程中的各个事件,首先发布的就是 __starting__ 事件.  
2. 第二步创建应用参数,默认传入的是空集合  
3. 第三步创建环境实例,首先看下 __ConfigurableEnvironment__ 相关类图
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/ConfigurableEnvironment.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="ConfigurableEnvironment"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/ConfigurableEnvironment.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">ConfigurableEnvironment</div>
    </a>
</center>

- PropertyResolver: 用于针对任何基础源解析属性的接口. 主要提供解析属性和获取属性的方法.    
- ConfigurablePropertyResolver: 提供用于访问和自定义将属性值从一种类型转换为另一种类型时使用的ConversionService的设置.
- Environment: 应用中表明环境的接口,主要有两个属性: __profile__ 和 __properties__
- ConfigurableEnvironment: 继承自 __ConfigurablePropertyResolver__ 和 __Environment__ ,所以它提供了设置profile和操作底层属性的能力.  
- StandardEnvironment: 继承自 __AbstractEnvironment__ 不仅提供了 __ConfigurableEnvironment__ 的能力还提供了设置 __systemEnvironment__ 和 __systemProperties__ 属性源的能力.  
- StandardServletEnvironment: 主要用于 __Servlet__ 应用的环境,默认基本都是它.  
- StandardReactiveWebEnvironment: 主要用于 __Reactive__ 应用的环境,SpringBoot中是空实现. 

相关类及其主要功能了解后,我们在回过头来看看这个 __prepareEnvironment__ 方法:  
``` java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments) {
    // Create and configure the environment
    //3.1根据当前应用类型创建并且配置environment 通常是StandardServletEnvironment
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    //3.2为环境设置属性资源和配置文件
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    //3.3将已有属性以key为 configurationProperties的形式添加到属性中
    ConfigurationPropertySources.attach(environment);
    //3.4发布环境准备完成通知
    listeners.environmentPrepared(environment);
    //3.5将environmrnt绑定到应用中
    bindToSpringApplication(environment);
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
                deduceEnvironmentClass());
    }
    //3.6 再次将已有属性以key为 configurationProperties的形式添加到属性中
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```

3.1 首先是创建并获取环境实例,源码如下:
    
``` java
private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
        return this.environment;
    }
    switch (this.webApplicationType) {
    case SERVLET:
        return new StandardServletEnvironment();
    case REACTIVE:
        return new StandardReactiveWebEnvironment();
    default:
        return new StandardEnvironment();
    }
}
```

因为我们当前应用类型是 __Servlet__ ,所以我们需要对应就是创建 __StandardServletEnvironment__ 实例.我们继续看下对应的构造方法:
``` java
//因为 StandardServletEnvironment extends StandardEnvironment 且 StandardEnvironment extends AbstractEnvironment
//我们看下父类AbstractEnvironment的构造
/**
 * Create a new {@code Environment} instance, calling back to
 * {@link #customizePropertySources(MutablePropertySources)} during construction to
 * allow subclasses to contribute or manipulate {@link PropertySource} instances as
 * appropriate.
 * 创建一个新的Environment实例, 在构造期间回调customizePropertySources方法以允许子类根据需要贡献或操作 PropertySource实例
 */
public AbstractEnvironment() {
    customizePropertySources(this.propertySources);
}

//StandardServletEnvironment中重写的customizePropertySourcesf方法
/** Servlet context init parameters property source name: {@value}. */
//Servlet上下文初始化参数属性源名称
public static final String SERVLET_CONTEXT_PROPERTY_SOURCE_NAME = "servletContextInitParams";

/** Servlet config init parameters property source name: {@value}. */
//Serlet配置初始化参数属性源名称
public static final String SERVLET_CONFIG_PROPERTY_SOURCE_NAME = "servletConfigInitParams";

/** JNDI property source name: {@value}. */
//JNDI属性源名称
public static final String JNDI_PROPERTY_SOURCE_NAME = "jndiProperties";

/**
 * Customize the set of property sources with those contributed by superclasses as
 * well as those appropriate for standard servlet-based environments:
 * 使用超类提供的属性源以及适用于基于servlet的标准环境的属性源自定义一组属性源:
 * <ul>
 * <li>{@value #SERVLET_CONFIG_PROPERTY_SOURCE_NAME}
 * <li>{@value #SERVLET_CONTEXT_PROPERTY_SOURCE_NAME}
 * <li>{@value #JNDI_PROPERTY_SOURCE_NAME}
 * </ul>
 */
@Override
protected void customizePropertySources(MutablePropertySources propertySources) {
    propertySources.addLast(new StubPropertySource(SERVLET_CONFIG_PROPERTY_SOURCE_NAME));
    propertySources.addLast(new StubPropertySource(SERVLET_CONTEXT_PROPERTY_SOURCE_NAME));
    if (JndiLocatorDelegate.isDefaultJndiEnvironmentAvailable()) {
        propertySources.addLast(new JndiPropertySource(JNDI_PROPERTY_SOURCE_NAME));
    }
    super.customizePropertySources(propertySources);
}


//StandardEnvironment中重写的customizePropertySources方法
/** System environment property source name: {@value}. */
//系统环境属性源名称
public static final String SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME = "systemEnvironment";

/** JVM system properties property source name: {@value}. */
//JVM系统属性源名称
public static final String SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME = "systemProperties";

/**
 * Customize the set of property sources with those appropriate for any standard
 * Java environment:
 * 使用适用于任何java标准环境的自定义属性源集合
 * <ul>
 * <li>{@value #SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME}
 * <li>{@value #SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME}
 * </ul>
 */
@Override
protected void customizePropertySources(MutablePropertySources propertySources) {
    propertySources.addLast(
            new PropertiesPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, getSystemProperties()));
    propertySources.addLast(
            new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));
}
```
可见在创建 __StandardServletEnvironment__ 实例时根据子类重写的 __customizePropertySources__ 方法将 __servletContextInitParams__ 、 __servletConfigInitParams__ 、__systemEnvironment__ 和 __systemProperties__ 四种属性添加到环境实例中.

3.2 设置环境属性资源和配置文件,源码如下
``` java
/**
 * Template method delegating to
 * {@link #configurePropertySources(ConfigurableEnvironment, String[])} and
 * {@link #configureProfiles(ConfigurableEnvironment, String[])} in that order.
 * Override this method for complete control over Environment customization, or one of
 * the above for fine-grained control over property sources or profiles, respectively.
 * 按照configurePropertySources和configureProfiles的顺序委托给他们的模板方法.
 * 覆盖此方法以完全控制环境自定义,或上述方法之一以分别对属性源或配置文件进行细粒度控制
 */
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
    //添加类型装换服务
    if (this.addConversionService) {
        //使用双层检查创建类型转换服务单例
        ConversionService conversionService = ApplicationConversionService.getSharedInstance();
        environment.setConversionService((ConfigurableConversionService) conversionService);
    }
    //配置属性源
    configurePropertySources(environment, args);
    //配置配置文件
    configureProfiles(environment, args);
}

/**
 * Return a shared default application {@code ConversionService} instance, lazily
 * building it once needed.
 * 使用懒加载的方式创建并返回一个共享的默认应用转换实例
 * 双重检查锁
 */
public static ConversionService getSharedInstance() {
    ApplicationConversionService sharedInstance = ApplicationConversionService.sharedInstance;
    if (sharedInstance == null) {
        synchronized (ApplicationConversionService.class) {
            sharedInstance = ApplicationConversionService.sharedInstance;
            if (sharedInstance == null) {
                sharedInstance = new ApplicationConversionService();
                ApplicationConversionService.sharedInstance = sharedInstance;
            }
        }
    }
    return sharedInstance;
}

/**
 * Add, remove or re-order any {@link PropertySource}s in this application's
 * environment.
 * 在应用环境中 添加、删除或重排序任何 PropertySources
 */
protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {
    //获取当前环境实例已有的属性源
    MutablePropertySources sources = environment.getPropertySources();
    //若存在defaultProperties且不为空则添加到当前环境实例中  默认为空 直接跳过
    if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
        sources.addLast(new MapPropertySource("defaultProperties", this.defaultProperties));
    }
    //是否打开添加命令行属性  且 存在输入的参数  默认无入参 直接跳过
    if (this.addCommandLineProperties && args.length > 0) {
        String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
        if (sources.contains(name)) {
            PropertySource<?> source = sources.get(name);
            CompositePropertySource composite = new CompositePropertySource(name);
            composite.addPropertySource(
                    new SimpleCommandLinePropertySource("springApplicationCommandLineArgs", args));
            composite.addPropertySource(source);
            sources.replace(name, composite);
        }
        else {
            sources.addFirst(new SimpleCommandLinePropertySource(args));
        }
    }
}

/**
 * Configure which profiles are active (or active by default) for this application
 * environment. Additional profiles may be activated during configuration file
 * processing via the {@code spring.profiles.active} property.
 * 为该应用程序配置哪些配置文件是活动的(默认是活动的).
 * 在配置文件处理期间可以通过spring.profiles.active属性来激活其他配置文件
 */
protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
    //默认additionalProfiles 为空
    Set<String> profiles = new LinkedHashSet<>(this.additionalProfiles);
    profiles.addAll(Arrays.asList(environment.getActiveProfiles()));
    environment.setActiveProfiles(StringUtils.toStringArray(profiles));
}
```
由上可见,第二步主要就是为环境实例添加转换服务 __conversionService__ 、检查是否需要配置默认属性源和是否需要根据命令行参数配置、设置profiles. 

3.3 第三步的源码如下:
``` java
/**
 * The name of the {@link PropertySource} {@link #attach(Environment) adapter}.
 * 属性源适配器的名称
 */
private static final String ATTACHED_PROPERTY_SOURCE_NAME = "configurationProperties";

/**
 * Attach a {@link ConfigurationPropertySource} support to the specified
 * {@link Environment}. Adapts each {@link PropertySource} managed by the environment
 * to a {@link ConfigurationPropertySource} and allows classic
 * {@link PropertySourcesPropertyResolver} calls to resolve using
 * {@link ConfigurationPropertyName configuration property names}.
 * 将ConfigurationPropertySource支持附加到指定的环境中.
 * 使由环境管理的每个PropertySource适应ConfigurationPropertySource, 并允许经典的PropertySourcesPropertyResolver调用使用名称解析
 */
public static void attach(Environment environment) {
    Assert.isInstanceOf(ConfigurableEnvironment.class, environment);
    //判断当前环境实例中 是否存在configurationProperties属性源
    //若存在且环境中属性源不等于他本身,则清除configurationProperties属性源, 并将configurationProperties属性源再次添加到sources中
    //如果不存在的话直接直接将configurationProperties属性源添加到sources中
    MutablePropertySources sources = ((ConfigurableEnvironment) environment).getPropertySources();
    PropertySource<?> attached = sources.get(ATTACHED_PROPERTY_SOURCE_NAME);
    if (attached != null && attached.getSource() != sources) {
        sources.remove(ATTACHED_PROPERTY_SOURCE_NAME);
        attached = null;
    }
    if (attached == null) {
        sources.addFirst(new ConfigurationPropertySourcesPropertySource(ATTACHED_PROPERTY_SOURCE_NAME,
                new SpringConfigurationPropertySources(sources)));
    }
}
```
由上可见,第三步的主要作用就是为 __Environment__ 实例配置 __configurationProperties__ 属性源,为后面的绑定到 __SpringApplication__ 而准备.

3.4 第四步就是发布环境准备就绪事件,这里有个监听器比较重要: __ConfigFileApplicationListener__ ,我们来看下源码
``` java
//ConfigFileApplicationListener implements EnvironmentPostProcessor, SmartApplicationListener, Ordered {
//先看下它的类关系, 它实现自EnvironmentPostProcessor, 即它重写的postProcessEnvironment方法

@Override
public void onApplicationEvent(ApplicationEvent event) {
    //当前发布的事件属于ApplicationEnvironmentPreparedEvent 继续往下周
    if (event instanceof ApplicationEnvironmentPreparedEvent) {
        onApplicationEnvironmentPreparedEvent((ApplicationEnvironmentPreparedEvent) event);
    }
    if (event instanceof ApplicationPreparedEvent) {
        onApplicationPreparedEvent(event);
    }
}

//可见获取的是/META-INF/spring.factories中的所有EnvironmentPostProcessor的实现类, 所以此类ConfigFileApplicationListener也在其中
List<EnvironmentPostProcessor> loadPostProcessors() {
    return SpringFactoriesLoader.loadFactories(EnvironmentPostProcessor.class, getClass().getClassLoader());
}

private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
    //这段就比较好理解了就是使用所有EnvironmentPostProcessor的后置处理器来处理环境实例
    List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
    postProcessors.add(this);
    AnnotationAwareOrderComparator.sort(postProcessors);
    for (EnvironmentPostProcessor postProcessor : postProcessors) {
        postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
    }
}

//具体后置处理器逻辑 其中最重要的就是加载了application.properties属性源
@Override
public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
    addPropertySources(environment, application.getResourceLoader());
}

/**
 * Add config file property sources to the specified environment.
 * 添加配置文件属性源到指定的环境实例中 其实这里就加载了random和applcation.properties两个属性源
 */
protected void addPropertySources(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
    //添加random属性源
    RandomValuePropertySource.addToEnvironment(environment);
    //创建Loader加载器并加载属性
    new Loader(environment, resourceLoader).load();
}

Loader(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
    this.environment = environment;
    //创建属性解析器 解析 ${}
    this.placeholdersResolver = new PropertySourcesPlaceholdersResolver(this.environment);
    this.resourceLoader = (resourceLoader != null) ? resourceLoader : new DefaultResourceLoader();
    //这里最重要,加载/META-INF/spring.factories中的PropertySourceLoader的实现类
    //默认是org.springframework.boot.env.PropertiesPropertySourceLoader,org.springframework.boot.env.YamlPropertySourceLoader 其实就是我们应用种配置的 .properties和 .yml配置文件
    this.propertySourceLoaders = SpringFactoriesLoader.loadFactories(PropertySourceLoader.class,
            getClass().getClassLoader());
}

//再来看下真正的load方法
void load() {
    FilteredPropertySource.apply(this.environment, DEFAULT_PROPERTIES, LOAD_FILTERED_PROPERTY,
            (defaultProperties) -> {
                this.profiles = new LinkedList<>();
                this.processedProfiles = new LinkedList<>();
                this.activatedProfiles = false;
                this.loaded = new LinkedHashMap<>();
                //初始化Profiles 默认初始化 active-profiles 和 spring.profiles.* 包含的所有profile
                initializeProfiles();
                //解析所有的profiles 这里至少有两个 null和default
                while (!this.profiles.isEmpty()) {
                    Profile profile = this.profiles.poll();
                    if (isDefaultProfile(profile)) {
                        addProfileToEnvironment(profile.getName());
                    }
                    //根据指定的profile加载资源 第一次解析null 第二次解析default
                    load(profile, this::getPositiveProfileFilter,
                            addToLoaded(MutablePropertySources::addLast, false));
                    this.processedProfiles.add(profile);
                }
                load(null, this::getNegativeProfileFilter, addToLoaded(MutablePropertySources::addFirst, true));
                //将加载过的资源添加到环境中
                addLoadedPropertySources();
                //应用激活的profiles
                applyActiveProfiles(defaultProperties);
            });
}

private void load(Profile profile, DocumentFilterFactory filterFactory, DocumentConsumer consumer) {
    //getSearchLocations()是获取搜索路径,默认是这个四个classpath:/,classpath:/config/,file:./,file:./config/
    getSearchLocations().forEach((location) -> {
        //判断是否文件夹
        boolean isFolder = location.endsWith("/");
        Set<String> names = isFolder ? getSearchNames() : NO_SEARCH_NAMES;
        //具体的加载动作
        names.forEach((name) -> load(location, name, profile, filterFactory, consumer));
    });
}

private void load(String location, String name, Profile profile, DocumentFilterFactory filterFactory,
            DocumentConsumer consumer) {
        if (!StringUtils.hasText(name)) {
            for (PropertySourceLoader loader : this.propertySourceLoaders) {
                if (canLoadFileExtension(loader, location)) {
                    load(loader, location, profile, filterFactory.getDocumentFilter(profile), consumer);
                    return;
                }
            }
            throw new IllegalStateException("File extension of config file location '" + location
                    + "' is not known to any PropertySourceLoader. If the location is meant to reference "
                    + "a directory, it must end in '/'");
        }
        Set<String> processed = new HashSet<>();
        //这里的propertySourceLoaders就是前面加载的 PropertiesPropertySourceLoader和YamlPropertySourceLoader两个解析器
        for (PropertySourceLoader loader : this.propertySourceLoaders) {
            //每个解析器都有自己的文件后缀并解析
            for (String fileExtension : loader.getFileExtensions()) {
                if (processed.add(fileExtension)) {
                    //获得具体路径后在解析成properties 下面就不继续往下看了
                    loadForFileExtension(loader, location + name, "." + fileExtension, profile, filterFactory,
                            consumer);
                }
            }
        }
    }
```
可见第四步通过发布环境准备就绪事件,所有的监听器中,最终要的就是 __ConfigFileApplicationListener__,通过 __ConfigFileApplicationListener__ 来执行所有环境相关的后置处理器, 他自己本身实现了 __EnvironmentPostProcessor__ ,通过 __postProcessEnvironment__ 来调用 __addPropertySources__ 方法通过 __Loader__ 来加载配置文件属性源.
在构造 __Loader__ 中初始化 __PropertySourceLoader__ 的两个实现类 __PropertiesPropertySourceLoader__ 和 __YamlPropertySourceLoader__ 来加载 __"properties","xml"__ 和 __"yml","yaml"__的配置文件,主要通过 __getFileExtensions()__ 方法指定的文件扩展名分别拼接默认路径 __classpath:/,classpath:/config/,file:./,file:./config/__ ,找到存在的路径则进行解析属性.

3.5 第五步将环境绑定到当前应用中
``` java
/**
 * Bind the environment to the {@link SpringApplication}.
 * 将环境实例绑定到应用中
 */
protected void bindToSpringApplication(ConfigurableEnvironment environment) {
    try {
        //首先看这个Binder.get(environment):根据给定环境实例创建Binder
        Binder.get(environment).bind("spring.main", Bindable.ofInstance(this));
    }
    catch (Exception ex) {
        throw new IllegalStateException("Cannot bind to SpringApplication", ex);
    }
}

public static Binder get(Environment environment) {
    return get(environment, null);
}

public static Binder get(Environment environment, BindHandler defaultBindHandler) {
    //获取前面通过attach附加到环境实例中的ConfigurationPropertySource
    Iterable<ConfigurationPropertySource> sources = ConfigurationPropertySources.get(environment);
    //根据环境创建属性源占位符解析器 即解析 ${}
    PropertySourcesPlaceholdersResolver placeholdersResolver = new PropertySourcesPlaceholdersResolver(environment);
    //创建Binder
    return new Binder(sources, placeholdersResolver, null, null, defaultBindHandler);
}

/**
 * Return a set of {@link ConfigurationPropertySource} instances that have previously
 * been {@link #attach(Environment) attached} to the {@link Environment}.
 * 返回之前通过attached附加到环境实例中的ConfigurationPropertySource集合
 */
public static Iterable<ConfigurationPropertySource> get(Environment environment) {
    Assert.isInstanceOf(ConfigurableEnvironment.class, environment);
    MutablePropertySources sources = ((ConfigurableEnvironment) environment).getPropertySources();
    ConfigurationPropertySourcesPropertySource attached = (ConfigurationPropertySourcesPropertySource) sources
            .get(ATTACHED_PROPERTY_SOURCE_NAME);
    if (attached == null) {
        return from(sources);
    }
    return attached.getSource();
}

//通过其中的PropertyPlaceholderHelper
public PropertySourcesPlaceholdersResolver(Iterable<PropertySource<?>> sources, PropertyPlaceholderHelper helper) {
    this.sources = sources;
    //创建PropertyPlaceholderHelper来解析 ${}
    this.helper = (helper != null) ? helper : new PropertyPlaceholderHelper(SystemPropertyUtils.PLACEHOLDER_PREFIX,
            SystemPropertyUtils.PLACEHOLDER_SUFFIX, SystemPropertyUtils.VALUE_SEPARATOR, true);
}

/**
 * Create a new {@link Binder} instance for the specified sources.
 * 通过指定的资源中创建一个新的Binder实例
 */
public Binder(Iterable<ConfigurationPropertySource> sources, PlaceholdersResolver placeholdersResolver,
        ConversionService conversionService, Consumer<PropertyEditorRegistry> propertyEditorInitializer,
        BindHandler defaultBindHandler, BindConstructorProvider constructorProvider) {
    Assert.notNull(sources, "Sources must not be null");
    this.sources = sources;
    this.placeholdersResolver = (placeholdersResolver != null) ? placeholdersResolver : PlaceholdersResolver.NONE;
    this.conversionService = (conversionService != null) ? conversionService
            : ApplicationConversionService.getSharedInstance();
    this.propertyEditorInitializer = propertyEditorInitializer;
    this.defaultBindHandler = (defaultBindHandler != null) ? defaultBindHandler : BindHandler.DEFAULT;
    if (constructorProvider == null) {
        constructorProvider = BindConstructorProvider.DEFAULT;
    }
    ValueObjectBinder valueObjectBinder = new ValueObjectBinder(constructorProvider);
    JavaBeanBinder javaBeanBinder = JavaBeanBinder.INSTANCE;
    this.dataObjectBinders = Collections.unmodifiableList(Arrays.asList(valueObjectBinder, javaBeanBinder));
}

//具体绑定代码不贴了,嵌套层数太深后期有时间在详细分析
```
可见第五步就是使用3.3新加入的 __configurationProperties__ 属性源和 __${}__ 占位符解析器 __PropertySourcesPlaceholdersResolver__ 来创建 __Binder__ .
最后在通过 __Binder__ 来将 __"spring.main"__ 封装的 __ConfigurationProperty__ 和主函数 __SpringApplication__ 进行绑定.

3.6 最后一步是再次将已有属性以key为 configurationProperties的形式添加到属性中
- - -

### prepareEnvironment总结  
> prepareEnvironment的作用就是将创建 Environment实例,并将各种属性资源添加到Environment中其中包括configurationProperties、servletConfigInitParams、servletContextInitParams、systemProperties、systemEnvironment、以及最重要的applicationConfig: [classpath:/application.properties],最后在通过Environment创建Binder将"spring.main"的属性源与应用SpringApplicaiton进行绑定.  




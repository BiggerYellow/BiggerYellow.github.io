---
layout: default
springboot: true
modal-id: 20000003
date: 2021-09-03
img: pexels-rachel-claire-5490361.jpg
alt: springboot3
project-date: April 2021
client: springboot
category: springboot
subtitle: SpringBoot启动源码解析(三)
description: SpringBootApplication Run方法解析
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
接下来我们就一些核心方法来仔细分析看看.  
1. 第一步比较好理解,创建定时器统计启动时间,并获取的事件发布者 __SpringApplicationRunListener__ 来发布启动过程中的各个事件,首先发布的就是 __starting__ 事件.  
2. 第二步创建应用参数,默认传入的是空集合,见下源码  
``` java
public DefaultApplicationArguments(String... args) {
    Assert.notNull(args, "Args must not be null");
    this.source = new Source(args);
    this.args = args;
}
```
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
    //3.3将属性附加到MutablePropertySources中
    ConfigurationPropertySources.attach(environment);
    //3.4发布环境准备完成通知
    listeners.environmentPrepared(environment);
    //3.5将environmrnt绑定到应用中
    bindToSpringApplication(environment);
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
                deduceEnvironmentClass());
    }
    //将属性附加到MutablePropertySources中
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```


- - -

---
layout: default
springboot: true
modal-id: 20000007
date: 2021-10-03
img: pexels-maverick-coltman-3214311.jpg
alt: SpringBoot
project-date: October 2021
client: SpringBoot
category: SpringBoot
subtitle: SpringBoot启动源码解析(七)
description: SpringBootApplication Refresh方法解析:processConfigBeanDefinitions
---
### 流程图
- - -
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry流程图.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry流程图"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry流程图.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry流程图</div>
    </a>
</center>
- - -
### 方法入口
- - -
调用路径:  
refresh() ->  
invokeBeanFactoryPostProcessors(beanFactory) ->  
invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors,registry) ->   
ConfigurationClassPostProcessor.processConfigBeanDefinitions()  

源码如下:
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
3. 通过ConfigurationClassParser的parse方法解析候选类 也就是主函数(核心逻辑),总结来说就是解析配置类上的所有注解属性并将这些bean添加到bean工厂中 
4. 通过reader.loadBeanDefinitions方法加载所有的配置类,其中注册导入的配置类自身、相关@Bean方法导入的类、@ImportResource导入的类和@Import导入的类的定义
5. 最后检测后扫描到的配置类中是否有未扫描到的,存在则加入到candidates中继续解析

- - -

### processConfigBeanDefinitions源码解析
- - -

1. 既然是要解析配置类,肯定要先找到符合条件的配置类,相关源码如下:  
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
    ...
}
```
&emsp;&emsp;遍历注册表中所有的bean定义,再通过 __ConfigurationClassUtils.checkConfigurationClassCandidate__ 方法判断是否满足配置类条件.满足条件则加入到configCandidates中.最后若configCandidates为空则直接返回.  
&emsp;&emsp;我们继续往看看这个检查配置类的方法做了什么:
``` java
/**
 * Check whether the given bean definition is a candidate for a configuration class
 * (or a nested component class declared within a configuration/component class,
 * to be auto-registered as well), and mark it accordingly.
 * 检查给定的bean定义是否是配置类的候选者(或在配置/组件类中声明的嵌套组件类,也可以自动注册),并且相应的标记他
 * @param beanDef the bean definition to check
 * @param metadataReaderFactory the current factory in use by the caller
 * @return whether the candidate qualifies as (any kind of) configuration class
 */
public static boolean checkConfigurationClassCandidate(
        BeanDefinition beanDef, MetadataReaderFactory metadataReaderFactory) {

    String className = beanDef.getBeanClassName();
    //1.当前bean定义无类名 一般是内部类类名为null 不存在返回false
    //2.当前bean定义存在工厂方法名 存在则返回false
    if (className == null || beanDef.getFactoryMethodName() != null) {
        return false;
    }

    //根据bean定义获取注解元数据
    AnnotationMetadata metadata;
    //当前bean定义是否属于注解相关的bean定义
    if (beanDef instanceof AnnotatedBeanDefinition &&
            //当前bean定义的类名是否与AnnotationMetadata的类名一致
            className.equals(((AnnotatedBeanDefinition) beanDef).getMetadata().getClassName())) {
        // Can reuse the pre-parsed metadata from the given BeanDefinition...
        // 可以重复使用来自给定bean定义的预解析元数据   主要获取启动类的元数据
        metadata = ((AnnotatedBeanDefinition) beanDef).getMetadata();
    }
    else if (beanDef instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) beanDef).hasBeanClass()) {
        // Check already loaded Class if present...
        //检查已加载的类 如果存在的话
        // since we possibly can't even load the class file for this Class.
        //因为我们甚至可能无法加载此类的类文件
        Class<?> beanClass = ((AbstractBeanDefinition) beanDef).getBeanClass();
        if (BeanFactoryPostProcessor.class.isAssignableFrom(beanClass) ||
                BeanPostProcessor.class.isAssignableFrom(beanClass) ||
                AopInfrastructureBean.class.isAssignableFrom(beanClass) ||
                EventListenerFactory.class.isAssignableFrom(beanClass)) {
            return false;
        }
        metadata = AnnotationMetadata.introspect(beanClass);
    }
    else {
        try {
            MetadataReader metadataReader = metadataReaderFactory.getMetadataReader(className);
            metadata = metadataReader.getAnnotationMetadata();
        }
        catch (IOException ex) {
            if (logger.isDebugEnabled()) {
                logger.debug("Could not find class file for introspecting configuration annotations: " +
                        className, ex);
            }
            return false;
        }
    }

    //从元数据中获取Configuration注解的属性
    //若bean定义被@Configuration修饰 且 proxyBeanMethods不为false 则设置configurationClass属性为full
    //若bean定义被@Component、@ComponentScan、@Import、@ImportResource和@Bean修饰 则设置configurationClass属性为lite
    //否则表明没有被@Configuration修饰  直接返回false
    Map<String, Object> config = metadata.getAnnotationAttributes(Configuration.class.getName());
    if (config != null && !Boolean.FALSE.equals(config.get("proxyBeanMethods"))) {
        beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
    }
    else if (config != null || isConfigurationCandidate(metadata)) {
        beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
    }
    else {
        return false;
    }

    // It's a full or lite configuration candidate... Let's determine the order value, if any.
    //他是完整或精简的配置候选  确认他的优先级值,如果有的话
    Integer order = getOrder(metadata);
    if (order != null) {
        beanDef.setAttribute(ORDER_ATTRIBUTE, order);
    }
    //最后返回true  已经解析过配置 表明是被@Configuration修饰的类
    return true;
}
```
&emsp;&emsp;第一步就是找到满足候选者添加的bean,首先保证该bean不存在bean名称且它的工厂方法名必须为空,即表明它是一个真正的类且不是工厂方法.  
&emsp;&emsp;然后获取该bean定义的注解元数据,用处肯定就是获取 __@Configuration__ 的配置了,在根据配置的 __proxyBeanMethods__ 属性设置bean定义的 __configurationClass__ 属性.若bean定义是被 __@Configration__ 修饰的则该属性为 __full__ ;若该属性是被 __@Component、@ComponentScan、@Import、@ImportResource__ 修饰的则该属性为 __lite__.  
&emsp;&emsp;若存在 __@Order__ 属性则设置它的优先级,处理到此时说明该bean已经是候选配置类了,直接返回true.  
>第一步的主要作用就是根据@Configuration的属性检查使用被@Configuration修饰,并设置相应的属性到bean定义中,满足条件则加入到configCandidates中.
2. 第一步已经找到符合条件的配置类了,那么第二步就是为解析做准备:
``` java
/**
 * Build and validate a configuration model based on the registry of
 * {@link Configuration} classes.
 * 基于Configuration类的注册表构建和验证配置模型
 */
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    ...
    
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
    
    ...
}
```
&emsp;&emsp;由上可见,首先将configCandidates中的配置类根据 __@Order__ 的值排序,然后设置bean名称生成器和环境,最后也是最重要的生成 __ConfigurationClassParser__ 配置类解析器.
>第二步的主要作用就是在准备ConfigurationClassParser配置类解析器. 
3. 第三步就是最最重要的一步了,解析配置类:
``` java
/**
 * Build and validate a configuration model based on the registry of
 * {@link Configuration} classes.
 * 基于Configuration类的注册表构建和验证配置模型
 */
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    ...
    
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

        ...
}
```
全部的解析动作都在 __parser.parse()__ 中,我们继续往下看源码:
``` java
public void parse(Set<BeanDefinitionHolder> configCandidates) {
    //1. 判断候选配置类定义属于哪种bean定义 执行相应的解析方法
    for (BeanDefinitionHolder holder : configCandidates) {
        BeanDefinition bd = holder.getBeanDefinition();
        try {
            //如果该bean定义是注解类型的bean定义 调用AnnotatedBeanDefinition中的parse方法
            if (bd instanceof AnnotatedBeanDefinition) {
                parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
            }
            //如果该bean定义是实现与AbstractBeanDefinition的注解 且该bean定义是否指定了一个bean类 是则调用AbstractBeanDefinition中的parse方法
            else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
            }
            else {
                parse(bd.getBeanClassName(), holder.getBeanName());
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
        }
    }

    //2. 处理DeferredImportSelector的子类 主要要是AutoConfigurationImportSelector 用来加载所有的EnableAutoConfiguration的子类
    this.deferredImportSelectorHandler.process();
}
```
&emsp;&emsp;可见解析分为两步,第一步根据bean定义的不同类型调用 __parse()__ 方法进行解析配置类的所有注解,第二步就是调用 __deferredImportSelectorHandler.process()__ 来处理 __@Import__ 导入的自动配置类.详细解析流程见下.  
>第三步的主要作用就是解析配置类,即找到通过@PropertySource、@ComponentScan、@Import、@ImportResource、@Bean指定的配置类
4. 第四步通过 __ConfigurationClassBeanDefinitionReader.loadBeanDefinitions__ 方法加载所有的配置类,源码如下:
``` java
/**
 * Read {@code configurationModel}, registering bean definitions
 * with the registry based on its contents.
 * 读取configurationModel,根据其内容向注册表注册bean定义
 */
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
    TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
    for (ConfigurationClass configClass : configurationModel) {
        loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
    }
}
```
&emsp;&emsp;可见它遍历所有的配置类,调用 __loadBeanDefinitionsForConfigurationClass__ 方法来进行加载,其源码如下:
``` java
/**
 * Read a particular {@link ConfigurationClass}, registering bean definitions
 * for the class itself and all of its {@link Bean} methods.
 * 读取特定的ConfigurationClass类,为类本身及其所有Bean方法注册bean定义
 */
private void loadBeanDefinitionsForConfigurationClass(
        ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {
    //1.检验是否需要跳过 还是通过 @Conditional注解
    //若跳过则将当前配置类从注册中心和导入列表中移除
    if (trackedConditionEvaluator.shouldSkip(configClass)) {
        String beanName = configClass.getBeanName();
        if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
            this.registry.removeBeanDefinition(beanName);
        }
        this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
        return;
    }
    //2.判断该类是否由@Import导入的
    //是则将自身作为bean定义注册
    if (configClass.isImported()) {
        registerBeanDefinitionForImportedConfigurationClass(configClass);
    }
    //3.获取配置类的bean方法 来注册bean定义
    for (BeanMethod beanMethod : configClass.getBeanMethods()) {
        loadBeanDefinitionsForBeanMethod(beanMethod);
    }
    //4.从ImportedResources中注册bean定义
    loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
    //5.从ImportBeanDefinitionRegister中注册bean定义 主要是针对AutoConfigurationPackages.Registrar.class
    loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```
&emsp;&emsp;可见 __loadBeanDefinitionsForConfigurationClass__ 可以分为五步:  
   4.1 第一步前置检查,通过 __@Conditional__ 配置检查该配置类是否需要跳过,若需要跳过则从注册中心和导入列表中移除该配置类  
   4.2 第二步判断该配置类是否通过 __@Import__ 导入的,若是,则将配置类自身作为bean定义注册. __registerBeanDefinitionForImportedConfigurationClass__ 源码如下:  
``` java
/**
 * Register the {@link Configuration} class itself as a bean definition.
 * 注册Configuration类自身作为bean定义
 */
private void registerBeanDefinitionForImportedConfigurationClass(ConfigurationClass configClass) {
    AnnotationMetadata metadata = configClass.getMetadata();
    //通过配置类的元数据创建通用配置注解类
    AnnotatedGenericBeanDefinition configBeanDef = new AnnotatedGenericBeanDefinition(metadata);

    ScopeMetadata scopeMetadata = scopeMetadataResolver.resolveScopeMetadata(configBeanDef);
    configBeanDef.setScope(scopeMetadata.getScopeName());
    String configBeanName = this.importBeanNameGenerator.generateBeanName(configBeanDef, this.registry);
    //处理通用注解
    AnnotationConfigUtils.processCommonDefinitionAnnotations(configBeanDef, metadata);

    //创建bean定义持有者 并注册到注册表中
    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(configBeanDef, configBeanName);
    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    this.registry.registerBeanDefinition(definitionHolder.getBeanName(), definitionHolder.getBeanDefinition());
    //注册bean定义
    configClass.setBeanName(configBeanName);

    if (logger.isTraceEnabled()) {
        logger.trace("Registered bean definition for imported class '" + configBeanName + "'");
    }
	}
```
&emsp;&emsp;这段代码就比较清楚了,将配置类封装为AnnotatedGenericBeanDefinition并设置其属性,最后将其注册到注册中心中.  
    4.3 第三步则是处理所有@Bean方法,相关源码如下:  
``` java
/**
 * Read the given {@link BeanMethod}, registering bean definitions
 * with the BeanDefinitionRegistry based on its contents.
 * 读取给定的bean方法,根据其内容向BeanDefinitionRegistry注册bean定义
 */
@SuppressWarnings("deprecation")  // for RequiredAnnotationBeanPostProcessor.SKIP_REQUIRED_CHECK_ATTRIBUTE
private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
    ConfigurationClass configClass = beanMethod.getConfigurationClass();
    MethodMetadata metadata = beanMethod.getMetadata();
    String methodName = metadata.getMethodName();

    // Do we need to mark the bean as skipped by its condition?
    //我们是否需要通过他的条件将这个bean标记为跳过
    if (this.conditionEvaluator.shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN)) {
        configClass.skippedBeanMethods.add(methodName);
        return;
    }
    if (configClass.skippedBeanMethods.contains(methodName)) {
        return;
    }

    AnnotationAttributes bean = AnnotationConfigUtils.attributesFor(metadata, Bean.class);
    Assert.state(bean != null, "No @Bean annotation attributes");

    // Consider name and any aliases
    //考虑名称和任何别名
    List<String> names = new ArrayList<>(Arrays.asList(bean.getStringArray("name")));
    String beanName = (!names.isEmpty() ? names.remove(0) : methodName);

    // Register aliases even when overridden
    //即使被覆盖也注册别名
    for (String alias : names) {
        this.registry.registerAlias(beanName, alias);
    }

    // Has this effectively been overridden before (e.g. via XML)?
    // 这之前是否有效的被注册
    if (isOverriddenByExistingDefinition(beanMethod, beanName)) {
        if (beanName.equals(beanMethod.getConfigurationClass().getBeanName())) {
            throw new BeanDefinitionStoreException(beanMethod.getConfigurationClass().getResource().getDescription(),
                    beanName, "Bean name derived from @Bean method '" + beanMethod.getMetadata().getMethodName() +
                    "' clashes with bean name for containing configuration class; please make those names unique!");
        }
        return;
    }

    ConfigurationClassBeanDefinition beanDef = new ConfigurationClassBeanDefinition(configClass, metadata);
    beanDef.setResource(configClass.getResource());
    beanDef.setSource(this.sourceExtractor.extractSource(metadata, configClass.getResource()));

    //当前bean方法是否为静态方法
    if (metadata.isStatic()) {
        // static @Bean method
        if (configClass.getMetadata() instanceof StandardAnnotationMetadata) {
            beanDef.setBeanClass(((StandardAnnotationMetadata) configClass.getMetadata()).getIntrospectedClass());
        }
        else {
            beanDef.setBeanClassName(configClass.getMetadata().getClassName());
        }
        beanDef.setUniqueFactoryMethodName(methodName);
    }
    else {
        // instance @Bean method
        beanDef.setFactoryBeanName(configClass.getBeanName());
        beanDef.setUniqueFactoryMethodName(methodName);
    }

    if (metadata instanceof StandardMethodMetadata) {
        beanDef.setResolvedFactoryMethod(((StandardMethodMetadata) metadata).getIntrospectedMethod());
    }

    beanDef.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_CONSTRUCTOR);
    beanDef.setAttribute(org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor.
            SKIP_REQUIRED_CHECK_ATTRIBUTE, Boolean.TRUE);

    AnnotationConfigUtils.processCommonDefinitionAnnotations(beanDef, metadata);

    //设置bean的属性
    Autowire autowire = bean.getEnum("autowire");
    if (autowire.isAutowire()) {
        beanDef.setAutowireMode(autowire.value());
    }

    boolean autowireCandidate = bean.getBoolean("autowireCandidate");
    if (!autowireCandidate) {
        beanDef.setAutowireCandidate(false);
    }

    String initMethodName = bean.getString("initMethod");
    if (StringUtils.hasText(initMethodName)) {
        beanDef.setInitMethodName(initMethodName);
    }

    String destroyMethodName = bean.getString("destroyMethod");
    beanDef.setDestroyMethodName(destroyMethodName);

    // Consider scoping
    //考虑范围
    ScopedProxyMode proxyMode = ScopedProxyMode.NO;
    AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(metadata, Scope.class);
    if (attributes != null) {
        beanDef.setScope(attributes.getString("value"));
        proxyMode = attributes.getEnum("proxyMode");
        if (proxyMode == ScopedProxyMode.DEFAULT) {
            proxyMode = ScopedProxyMode.NO;
        }
    }

    // Replace the original bean definition with the target one, if necessary
    //如果有必要 将原始bean替换为目标bean
    BeanDefinition beanDefToRegister = beanDef;
    if (proxyMode != ScopedProxyMode.NO) {
        BeanDefinitionHolder proxyDef = ScopedProxyCreator.createScopedProxy(
                new BeanDefinitionHolder(beanDef, beanName), this.registry,
                proxyMode == ScopedProxyMode.TARGET_CLASS);
        beanDefToRegister = new ConfigurationClassBeanDefinition(
                (RootBeanDefinition) proxyDef.getBeanDefinition(), configClass, metadata);
    }

    if (logger.isTraceEnabled()) {
        logger.trace(String.format("Registering bean definition for @Bean method %s.%s()",
                configClass.getMetadata().getClassName(), beanName));
    }
    //注册bean定义
    this.registry.registerBeanDefinition(beanName, beanDefToRegister);
}
```
&emsp;&emsp;看到这么长的一堆代码,任谁都不想看,但其实这段代码的逻辑并不复杂,还是先检查bean方法是否需要跳过,不需要跳过在根据配置类和@Bean注解的元数据创建bean定义,最后再注册到注册中心去即可.  
    4.4 从@ImportSource中加载bean定义,相关源码如下:  
``` java
private void loadBeanDefinitionsFromImportedResources(
        Map<String, Class<? extends BeanDefinitionReader>> importedResources) {

    Map<Class<?>, BeanDefinitionReader> readerInstanceCache = new HashMap<>();

    importedResources.forEach((resource, readerClass) -> {
        // Default reader selection necessary?
        //需要选择默认读取器  主要是groovy或 xml方式
        if (BeanDefinitionReader.class == readerClass) {
            if (StringUtils.endsWithIgnoreCase(resource, ".groovy")) {
                // When clearly asking for Groovy, that's what they'll get...
                readerClass = GroovyBeanDefinitionReader.class;
            }
            else {
                // Primarily ".xml" files but for any other extension as well
                readerClass = XmlBeanDefinitionReader.class;
            }
        }

        //从缓存中读取bean定义读取器   不存在先创建bean定义读取器 最后再加载bean定义
        BeanDefinitionReader reader = readerInstanceCache.get(readerClass);
        if (reader == null) {
            try {
                // Instantiate the specified BeanDefinitionReader
                //实例化指定的bean定义读取器
                reader = readerClass.getConstructor(BeanDefinitionRegistry.class).newInstance(this.registry);
                // Delegate the current ResourceLoader to it if possible
                //如果可能的话委托给当前资源加载器
                if (reader instanceof AbstractBeanDefinitionReader) {
                    AbstractBeanDefinitionReader abdr = ((AbstractBeanDefinitionReader) reader);
                    abdr.setResourceLoader(this.resourceLoader);
                    abdr.setEnvironment(this.environment);
                }
                readerInstanceCache.put(readerClass, reader);
            }
            catch (Throwable ex) {
                throw new IllegalStateException(
                        "Could not instantiate BeanDefinitionReader class [" + readerClass.getName() + "]");
            }
        }

        // TODO SPR-6310: qualify relative path locations as done in AbstractContextLoader.modifyLocations
        reader.loadBeanDefinitions(resource);
    });
}
```
&emsp;&emsp;此注解用的不是很多,做了解即可.它的主要作用是通过BeanDefinitionReader去加载指定的配置文件中的属性.  
    4.5 从导入的ImportBeanDefinitionRegistrar的实现类中加载bean定义,这里我们主要针对的是 __AutoConfigurationPackages.Registrar.class__ 类,源码如下:  
``` java
//通过注册表注册bean定义
private void loadBeanDefinitionsFromRegistrars(Map<ImportBeanDefinitionRegistrar, AnnotationMetadata> registrars) {
    registrars.forEach((registrar, metadata) ->
            registrar.registerBeanDefinitions(metadata, this.registry, this.importBeanNameGenerator));
}
```
&emsp;&emsp;它的主要作用是调用每个ImportBeanDefinitionRegistrar的实现类重写的registerBeanDefinitions方法,我们继续看下 __AutoConfigurationPackages.Registrar.class__ 中重写的方法:
    ``` java
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        register(registry, new PackageImport(metadata).getPackageName());
    }
    
    private static final String BEAN = AutoConfigurationPackages.class.getName();
    
    public static void register(BeanDefinitionRegistry registry, String... packageNames) {
        if (registry.containsBeanDefinition(BEAN)) {
            BeanDefinition beanDefinition = registry.getBeanDefinition(BEAN);
            ConstructorArgumentValues constructorArguments = beanDefinition.getConstructorArgumentValues();
            constructorArguments.addIndexedArgumentValue(0, addBasePackages(constructorArguments, packageNames));
        }
        else {
            GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
            beanDefinition.setBeanClass(BasePackages.class);
            beanDefinition.getConstructorArgumentValues().addIndexedArgumentValue(0, packageNames);
            beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
            registry.registerBeanDefinition(BEAN, beanDefinition);
        }
    }
    ```
&emsp;&emsp;可见AutoConfigurationPackages.Registrar的作用就是注册AutoConfigurationPackages的bean定义.  
>第四步就是将上一阶段找到的配置类进行加载,主要针对以下几个方面:配置类自身、@Bean注解导入的类、@ImportSource注解导入的类和@Import导入的ImportBeanDefinitionRegistrar的实现类
5. 第五步检查是否所有的配置类都已经解析过了,解析完直接结束,源码如下:
``` java
/**
 * Build and validate a configuration model based on the registry of
 * {@link Configuration} classes.
 * 基于Configuration类的注册表构建和验证配置模型
 */
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
        ...
        
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
&emsp;&emsp;可见将注册中心中注册的bean定义同已经解析过配置类和之前的候选类做比较,只有满足配置类条件且同时不在已解析集合和候选者集合中时才需要继续解析.若最后candidates为空说明配置类解析完成.  
&emsp;&emsp;最后注册ConfigurationClassPostProcessor的bean定义以及清除缓存.  
>第五步最后检查是否还有漏掉的配置类,有则继续解析,没有则表明解析完成

- - - 

### ConfigurationClassParser.parse(candidates)
- - -
&emsp;&emsp;我们先来看 __parse__ 方法:
``` java
protected final void parse(Class<?> clazz, String beanName) throws IOException {
    //调用真正的解析方法
    processConfigurationClass(new ConfigurationClass(clazz, beanName));
}
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
    //1. 判断是否被Conditional修饰 且 是否应该注册
    if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
        return;
    }
    
    //2. 如果当前配置类已经被处理过 则继续判断是否是@Import导入的,是则合并并返回 不是则移除继续解析
    ConfigurationClass existingClass = this.configurationClasses.get(configClass);
    if (existingClass != null) {
        if (configClass.isImported()) {
            if (existingClass.isImported()) {
                existingClass.mergeImportedBy(configClass);
            }
            // Otherwise ignore new imported config class; existing non-imported class overrides it.
            return;
        }
        else {
            // Explicit bean definition found, probably replacing an import.
            // Let's remove the old one and go with the new one.
            this.configurationClasses.remove(configClass);
            this.knownSuperclasses.values().removeIf(configClass::equals);
        }
    }

    // Recursively process the configuration class and its superclass hierarchy.
    //3. 递归处理配置类及其超类层次结构
    SourceClass sourceClass = asSourceClass(configClass);
    do {候选者实现了 ImportSelector 接口
        //执行解析逻辑
        sourceClass = doProcessConfigurationClass(configClass, sourceClass);
    }
    while (sourceClass != null);

    this.configurationClasses.put(configClass, configClass);
}
```
&emsp;&emsp;我们先来看下 __processConfigurationClass__ 的逻辑:  
1.第一步先判断当前配置类是否应该跳过,判断的条件是什么呢,我们看下源码:  
``` java
/**
 * Determine if an item should be skipped based on {@code @Conditional} annotations.
 * 根据@Conditional注解确定是否应该跳过该项目
 * @param metadata the meta data
 * @param phase the phase of the call
 * @return if the item should be skipped
 */
public boolean shouldSkip(@Nullable AnnotatedTypeMetadata metadata, @Nullable ConfigurationPhase phase) {
    //如果注解元数据为null返回false   或 注解元数据不为null且注解元数据未被Conditional修饰也返回false
    if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
        return false;
    }

    //如果配置解析器为null
    if (phase == null) {
        if (metadata instanceof AnnotationMetadata &&
                ConfigurationClassUtils.isConfigurationCandidate((AnnotationMetadata) metadata)) {
            return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
        }
        return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
    }

    List<Condition> conditions = new ArrayList<>();
    //获取@Conditional的value值并存入conditions中
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
```
&emsp;&emsp;看到 __@Conditional__ 这个注解是不是就很熟悉了,就是根据 __Conditional__ 的属性来判断是否需要跳过.  
2.第二步则是判断当前配置类使用已经解析过,如果是通过 __@Import__ 导入的配置类则合并并返回,不是则移除配置类继续解析  
3.第三步就是获取配置类源信息并通过 __doProcessConfigurationClass__ 来进行解析配置类,最后再将解析过的配置类添加到configurationClasses中  
&emsp;&emsp;接下来我们着重来看看这个 __doProcessConfigurationClass__ :
``` java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
        throws IOException {
    //3.1 首先递归处理任何嵌套的成员类
    if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
        // Recursively process any member (nested) classes first
        processMemberClasses(configClass, sourceClass);
    }

    // Process any @PropertySource annotations
    //3.2 处理所有的@PropertySource注解
    for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), PropertySources.class,
            org.springframework.context.annotation.PropertySource.class)) {
        if (this.environment instanceof ConfigurableEnvironment) {
            processPropertySource(propertySource);
        }
        else {
            logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                    "]. Reason: Environment must implement ConfigurableEnvironment");
        }
    }

    // Process any @ComponentScan annotations
    //3.3 处理所有的@ComponentScan注解
    Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
    //扫描到的ComponentScan不为空且 Conditonal修饰的注解条件不应该跳过
    if (!componentScans.isEmpty() &&
            !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
        for (AnnotationAttributes componentScan : componentScans) {
            // The config class is annotated with @ComponentScan -> perform the scan immediately
            //使用@ComponentScans注解修饰的配置类  -> 立即执行扫描
            //扫描到ComponectScan指定的类  默认classpath下的所有类且被@Component修饰的类
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                    this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
            // Check the set of scanned definitions for any further config classes and parse recursively if needed
            // 检查任何其他配置类的扫描定义集合 并 在需要时递归解析
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                if (bdCand == null) {
                    bdCand = holder.getBeanDefinition();
                }
                //检查通过ComponentScan扫描出来的类是否满足Configuration的候选 满足则进行类解析逻辑
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                    parse(bdCand.getBeanClassName(), holder.getBeanName());
                }
            }
        }
    }

    // Process any @Import annotations
    //3.4 处理所有的@Import的注解
    //getImport方法目前在springboot启动中有以下几个值
    //1.org.springframework.boot.autoconfigure.AutoConfigurationImportSelector
    //2.org.springframework.boot.autoconfigure.AutoConfigurationPackages.Registrar
    //主要用于 处理所有使用@Import注解额类
    processImports(configClass, sourceClass, getImports(sourceClass), true);

    // Process any @ImportResource annotations
    //3.5 处理所有存在的@ImportResource注解
    AnnotationAttributes importResource =
            AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
    if (importResource != null) {
        String[] resources = importResource.getStringArray("locations");
        Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
        for (String resource : resources) {
            String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
            configClass.addImportedResource(resolvedResource, readerClass);
        }
    }

    // Process individual @Bean methods
    //3.6 处理所有独立的@Bean方法
    Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
    for (MethodMetadata methodMetadata : beanMethods) {
        configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
    }

    // Process default methods on interfaces
    //3.7 处理接口上的默认方法
    processInterfaces(configClass, sourceClass);

    // Process superclass, if any
    //3.8 如果有的话处理父类
    if (sourceClass.getMetadata().hasSuperClass()) {
        String superclass = sourceClass.getMetadata().getSuperClassName();
        if (superclass != null && !superclass.startsWith("java") &&
                !this.knownSuperclasses.containsKey(superclass)) {
            this.knownSuperclasses.put(superclass, configClass);
            // Superclass found, return its annotation metadata and recurse
            return sourceClass.getSuperClass();
        }
    }

    // No superclass -> processing is complete
    //没有父类的话 处理完成 返回null
    return null;
}
```
&emsp;&emsp;可以看到 __doProcessConfigurationClass__ 中解析了我们常见的注解,例如 __@ComponentScan、@Import、@Bean__ 等等.下面我们一一看下是如何解析的:  
3.1 首先递归的解析嵌套类成员,源码如下:
``` java
/**
 * Register member (nested) classes that happen to be configuration classes themselves.
 * 注册 恰好是配置类本身的成员嵌套类
 */
private void processMemberClasses(ConfigurationClass configClass, SourceClass sourceClass) throws IOException {
    //获取来源类的成员类
    Collection<SourceClass> memberClasses = sourceClass.getMemberClasses();
    //筛选出符合条件的成员类 执行processConfigurationClass解析逻辑
    if (!memberClasses.isEmpty()) {
        List<SourceClass> candidates = new ArrayList<>(memberClasses.size());
        for (SourceClass memberClass : memberClasses) {
            if (ConfigurationClassUtils.isConfigurationCandidate(memberClass.getMetadata()) &&
                    !memberClass.getMetadata().getClassName().equals(configClass.getMetadata().getClassName())) {
                candidates.add(memberClass);
            }
        }
        OrderComparator.sort(candidates);
        for (SourceClass candidate : candidates) {
            if (this.importStack.contains(configClass)) {
                this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
            }
            else {
                this.importStack.push(configClass);
                try {
                    processConfigurationClass(candidate.asConfigClass(configClass));
                }
                finally {
                    this.importStack.pop();
                }
            }
        }
    }
}
```
&emsp;&emsp;由上可见,首先先处理来源类中所有符合配置类条件的成员类,筛选出来后先将这些配置类进行解析.  
3.2 第二步解析 __@PropertySources__ 注解:
``` java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
        throws IOException {
    ...

    // Process any @PropertySource annotations
    //3.2 处理所有的@PropertySource注解
    for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), PropertySources.class,
            org.springframework.context.annotation.PropertySource.class)) {
        if (this.environment instanceof ConfigurableEnvironment) {
            processPropertySource(propertySource);
        }
        else {
            logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                    "]. Reason: Environment must implement ConfigurableEnvironment");
        }
    }

   ...
}
```
&emsp;&emsp;其中 __AnnotationConfigUtils.attributesForRepeatable__ 方法的作用是获取指定注解的所有属性,通过属性是否存在可以判断是否存在该注解.这里我们判断是否存在@PropertySource的属性,不存在则跳过,存在且当前环境属于ConfigurableEnvironment时才处理该注解.  
&emsp;&emsp;相关源码如下:
``` java
/**
 * Process the given <code>@PropertySource</code> annotation metadata.
 * 处理给定的@PropertySource注解元数据
 * @param propertySource metadata for the <code>@PropertySource</code> annotation found
 * @throws IOException if loading a property source failed
 */
private void processPropertySource(AnnotationAttributes propertySource) throws IOException {
    String name = propertySource.getString("name");
    if (!StringUtils.hasLength(name)) {
        name = null;
    }
    String encoding = propertySource.getString("encoding");
    if (!StringUtils.hasLength(encoding)) {
        encoding = null;
    }
    String[] locations = propertySource.getStringArray("value");
    Assert.isTrue(locations.length > 0, "At least one @PropertySource(value) location is required");
    boolean ignoreResourceNotFound = propertySource.getBoolean("ignoreResourceNotFound");

    Class<? extends PropertySourceFactory> factoryClass = propertySource.getClass("factory");
    PropertySourceFactory factory = (factoryClass == PropertySourceFactory.class ?
            DEFAULT_PROPERTY_SOURCE_FACTORY : BeanUtils.instantiateClass(factoryClass));

    for (String location : locations) {
        try {
            String resolvedLocation = this.environment.resolveRequiredPlaceholders(location);
            Resource resource = this.resourceLoader.getResource(resolvedLocation);
            addPropertySource(factory.createPropertySource(name, new EncodedResource(resource, encoding)));
        }
        catch (IllegalArgumentException | FileNotFoundException | UnknownHostException ex) {
            // Placeholders not resolvable or resource not found when trying to open it
            if (ignoreResourceNotFound) {
                if (logger.isInfoEnabled()) {
                    logger.info("Properties location [" + location + "] not resolvable: " + ex.getMessage());
                }
            }
            else {
                throw ex;
            }
        }
    }
}
```
&emsp;&emsp;主要作用就是根据注解的各个属性获取并创建资源,存在则替换,不存在则新增.  
3.3 第三步解析 __@ComponentScan__ 注解:
``` java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
        throws IOException {
    ...

    // Process any @ComponentScan annotations
    //3.3 处理所有的@ComponentScan注解
    Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
    //扫描到的ComponentScan不为空且 Conditonal修饰的注解条件不应该跳过
    if (!componentScans.isEmpty() &&
            !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
        for (AnnotationAttributes componentScan : componentScans) {
            // The config class is annotated with @ComponentScan -> perform the scan immediately
            //使用@ComponentScans注解修饰的配置类  -> 立即执行扫描
            //扫描到ComponectScan指定的类  默认classpath下的所有类且被@Component修饰的类
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                    this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
            // Check the set of scanned definitions for any further config classes and parse recursively if needed
            // 检查任何其他配置类的扫描定义集合 并 在需要时递归解析
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                if (bdCand == null) {
                    bdCand = holder.getBeanDefinition();
                }
                //检查通过ComponentScan扫描出来的类是否满足Configuration的候选 满足则进行类解析逻辑
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                    parse(bdCand.getBeanClassName(), holder.getBeanName());
                }
            }
        }
    }

    ...
}
```
&emsp;&emsp;可见通过是通过 __AnnotationConfigUtils.attributesForRepeatable__ 方法获取到所有 __@ComponentScan__ 注解的所有属性,然后根据该属性去扫描指定路径下的所有配置类,最近将所有满足条件的配置继续进行配置类解析.  
这里我们着重看下它是如何扫描指定包下的配置类的,它通过 __componentScanParser__ 组件扫描器来进行扫描,源码如下:
``` java
//处理ComponectScan的所有的属性
public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
    ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
            componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);

    Class<? extends BeanNameGenerator> generatorClass = componentScan.getClass("nameGenerator");
    boolean useInheritedGenerator = (BeanNameGenerator.class == generatorClass);
    scanner.setBeanNameGenerator(useInheritedGenerator ? this.beanNameGenerator :
            BeanUtils.instantiateClass(generatorClass));

    ScopedProxyMode scopedProxyMode = componentScan.getEnum("scopedProxy");
    if (scopedProxyMode != ScopedProxyMode.DEFAULT) {
        scanner.setScopedProxyMode(scopedProxyMode);
    }
    else {
        Class<? extends ScopeMetadataResolver> resolverClass = componentScan.getClass("scopeResolver");
        scanner.setScopeMetadataResolver(BeanUtils.instantiateClass(resolverClass));
    }

    scanner.setResourcePattern(componentScan.getString("resourcePattern"));

    for (AnnotationAttributes filter : componentScan.getAnnotationArray("includeFilters")) {
        for (TypeFilter typeFilter : typeFiltersFor(filter)) {
            scanner.addIncludeFilter(typeFilter);
        }
    }
    for (AnnotationAttributes filter : componentScan.getAnnotationArray("excludeFilters")) {
        for (TypeFilter typeFilter : typeFiltersFor(filter)) {
            scanner.addExcludeFilter(typeFilter);
        }
    }

    boolean lazyInit = componentScan.getBoolean("lazyInit");
    if (lazyInit) {
        scanner.getBeanDefinitionDefaults().setLazyInit(true);
    }

    Set<String> basePackages = new LinkedHashSet<>();
    String[] basePackagesArray = componentScan.getStringArray("basePackages");
    for (String pkg : basePackagesArray) {
        String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg),
                ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
        Collections.addAll(basePackages, tokenized);
    }
    //若没有指定basePackageClasses,会取主函数的包名,即扫描主函数所在包下的所有类
    for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
        basePackages.add(ClassUtils.getPackageName(clazz));
    }
    if (basePackages.isEmpty()) {
        basePackages.add(ClassUtils.getPackageName(declaringClass));
    }

    scanner.addExcludeFilter(new AbstractTypeHierarchyTraversingFilter(false, false) {
        @Override
        protected boolean matchClassName(String className) {
            return declaringClass.equals(className);
        }
    });
    //扫描basePackages指定目录下的类  默认扫描classpath下的路径
    return scanner.doScan(StringUtils.toStringArray(basePackages));
}
```
&emsp;&emsp;可见扫描器一开始都在填充 __@ComponentScan__ 的属性,这里需要注意的就是 __basePackageClasses__ 属性,即指定扫描的路径,若未指定则会扫描主函数所在的包.  
&emsp;&emsp;属性封装完成后,即执行扫描,源码如下:
``` java
/**
 * Perform a scan within the specified base packages,
 * returning the registered bean definitions.
 * <p>This method does <i>not</i> register an annotation config processor
 * but rather leaves this up to the caller.
 * 在指定的包中执行扫描，返回注册的bean定义。
 * 这个方法不注册一个注解配置处理器，而是把这个留给调用者
 * @param basePackages the packages to check for annotated classes
 * @return set of beans registered if any for tooling registration purposes (never {@code null})
 */
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Assert.notEmpty(basePackages, "At least one base package must be specified");
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
    for (String basePackage : basePackages) {
        //获取候选者组件
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
        for (BeanDefinition candidate : candidates) {
            //解析作用域元数据 获取@Scope 注解的属性
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
            candidate.setScope(scopeMetadata.getScopeName());
            //生成bean的名字
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
            //判断该候选者是否属于AbstractBeanDefinition 执行后置处理
            if (candidate instanceof AbstractBeanDefinition) {
                postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
            }
            //判断该候选者是否属于可注解的bean定义
            if (candidate instanceof AnnotatedBeanDefinition) {
                //处理通用的注解 例如:@Lazy、@Primary、@DependsOn、@Role、@Description
                AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
            }
            //检查候选者是否已经注册过
            if (checkCandidate(beanName, candidate)) {
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                //根据代理模式创建代理类，没有则不创建
                definitionHolder =
                        AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                beanDefinitions.add(definitionHolder);
                //将扫描出来的类注册到注册表中
                registerBeanDefinition(definitionHolder, this.registry);
            }
        }
    }
    return beanDefinitions;
}
```  
&emsp;&emsp;这里逻辑就比较清晰了,根据注解的属性扫描过滤出满足条件的配置类并注册到bean定义到注册表中.  
3.4 第四步解析 __@Import__ 注解,看到这个注解是不是比较熟悉,在 __@SpringBootApplication__ 注解中的 __@EnableAutoConfiguration__ 中就包含两个 __@Import__ 注解,就是在这里处理的,源码如下:
``` java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
        throws IOException {
    ...

    // Process any @Import annotations
    //3.4 处理所有的@Import的注解
    //getImport方法目前在springboot启动中有以下几个值
    //1.org.springframework.boot.autoconfigure.AutoConfigurationImportSelector
    //2.org.springframework.boot.autoconfigure.AutoConfigurationPackages.Registrar
    //主要用于 处理所有使用@Import注解额类
    processImports(configClass, sourceClass, getImports(sourceClass), true);

    ...
}
```
&emsp;&emsp;由上可见,第四步可以拆分为两个方法, __getImports()__ 和 __processImports()__ ,顾名思义第一个方法肯定是获取所有的 __@Import__ 注解,而第二个方法就是处理所有的 __@Import__ 注解了.    
&emsp;&emsp;首先我们先来看下 __getImports()__ 源码:
``` java
/**
 * Returns {@code @Import} class, considering all meta-annotations.
 * 返回 @Import 的类 考虑所有的元注解
 * 即返回所有@Import注解的 value值所指向的类
 */
private Set<SourceClass> getImports(SourceClass sourceClass) throws IOException {
    Set<SourceClass> imports = new LinkedHashSet<>();
    Set<SourceClass> visited = new LinkedHashSet<>();
    collectImports(sourceClass, imports, visited);
    return imports;
}
/**
 * Recursively collect all declared {@code @Import} values. Unlike most
 * meta-annotations it is valid to have several {@code @Import}s declared with
 * different values; the usual process of returning values from the first
 * meta-annotation on a class is not sufficient.
 * 递归收集所有声明的@import值.  不像所有的元注解 有几个声明为不同的值的@Import注解是合法的.
 * 从类的第一个元注解返回值 通常是不够的
 * <p>For example, it is common for a {@code @Configuration} class to declare direct
 * {@code @Import}s in addition to meta-imports originating from an {@code @Enable}
 * annotation.
 * 举个例子: 除了源自@Enable注解的元导入之外 @Configuration类直接声明@Import是很常见的
 * @param sourceClass the class to search
 * @param imports the imports collected so far
 * @param visited used to track visited classes to prevent infinite recursion
 * @throws IOException if there is any problem reading metadata from the named class
 */
private void collectImports(SourceClass sourceClass, Set<SourceClass> imports, Set<SourceClass> visited)
        throws IOException {

    //递归获取到@Import注解  将该注解的值添加到imports中
    if (visited.add(sourceClass)) {
        for (SourceClass annotation : sourceClass.getAnnotations()) {
            String annName = annotation.getMetadata().getClassName();
            if (!annName.equals(Import.class.getName())) {
                collectImports(annotation, imports, visited);
            }
        }
        imports.addAll(sourceClass.getAnnotationAttributes(Import.class.getName(), "value"));
    }
}
```
&emsp;&emsp;可见 __collectImports()__ 通过递归的方法获取主函数中所有的 __@Import__ 注解,并将对应的value值添加到结果集合中.这里其实就对应 __org.springframework.boot.autoconfigure.AutoConfigurationImportSelector和org.springframework.boot.autoconfigure.AutoConfigurationPackages.Registrar__ 两个类.    
&emsp;&emsp;接着我们再来看看处理逻辑: 
``` java
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
        Collection<SourceClass> importCandidates, boolean checkForCircularImports) {
    //前置检查 
    //导入候选类为空
    if (importCandidates.isEmpty()) {
        return;
    }

    //是否检查循环导入 且 当前配置类存在栈中
    if (checkForCircularImports && isChainedImportOnStack(configClass)) {
        this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
    }
    else {
        //将当前处理的配置类压栈
        this.importStack.push(configClass);
        try {
            //遍历Import的候选类 通过getImports方法获取的候选类
            for (SourceClass candidate : importCandidates) {
                //当前候选者是否可指定为 ImportSelector类
                if (candidate.isAssignable(ImportSelector.class)) {
                    // Candidate class is an ImportSelector -> delegate to it to determine imports
                    //候选类如果属于ImportSelector -> 委托给他来确定导入
                    Class<?> candidateClass = candidate.loadClass();
                    //实例化ImportSelector
                    ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class,
                            this.environment, this.resourceLoader, this.registry);
                    //当前ImportSelector是否属于DeferredImportSelector  是则执行handle方法 将当前的配置和selector放入deferredImportSelectors中
                    if (selector instanceof DeferredImportSelector) {
                        this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
                    }
                    else {//不属于直接继续执行 递归处理Import方法逻辑
                        String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                        Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
                        processImports(configClass, currentSourceClass, importSourceClasses, false);
                    }
                }
                //当前候选者是否可以指定为 ImportBeanDefinitionRegister类
                else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                    // Candidate class is an ImportBeanDefinitionRegistrar ->
                    // delegate to it to register additional bean definitions
                    //候选类是 ImportBeanDefinitionRegistrar -> 委托给他注册额外的bean定义
                    Class<?> candidateClass = candidate.loadClass();
                    //实例化ImportBeanDefinitionRegistrar类
                    ImportBeanDefinitionRegistrar registrar =
                            ParserStrategyUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class,
                                    this.environment, this.resourceLoader, this.registry);
                    //将实例化的类 添加到importBeanDefinitionRegistrars中
                    configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                }
                else {//若以上两个类都不属于  走默认逻辑
                    // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
                    // process it as an @Configuration class
                    //候选者类既不是ImportSelector 或 ImportBeanDefinitionRegistrar ->  把他当做@Configuration类来处理
                    this.importStack.registerImport(
                            currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                    //执行@Configuration处理逻辑
                    processConfigurationClass(candidate.asConfigClass(configClass));
                }
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to process import candidates for configuration class [" +
                    configClass.getMetadata().getClassName() + "]", ex);
        }
        finally {
            this.importStack.pop();
        }
    }
}
```
&emsp;&emsp; __processImports__ 首先肯定是基础的前置检查,然后就是将当前处理的配置类压栈,接着就是处理所有 __@Import__ 导入的类,有以下三种情况:  
   - 候选者实现了 __ImportSelector__ 接口:  
&emsp;&emsp;候选者对应启动类中声明的 __AutoConfigurationImportSelector.class__ 类,首先将此候选类实例化为 __ImportSelector__ ,判断候选者是否实现了 __DeferredImportSelector__ ,未实现则继续从源配置类的元注解中找到需要导入的配置类递归解析.    
&emsp;&emsp;我们着重来看下实现了 __DeferredImportSelector__ 接口的实现逻辑,因为__AutoConfigurationImportSelector.class__ 类就实现了此接口.我们继续看下 __handle__ 源码:

``` java
private List<DeferredImportSelectorHolder> deferredImportSelectors = new ArrayList<>();
/**
 * Handle the specified {@link DeferredImportSelector}. If deferred import
 * selectors are being collected, this registers this instance to the list. If
 * they are being processed, the {@link DeferredImportSelector} is also processed
 * immediately according to its {@link DeferredImportSelector.Group}.
 * 处理指定的DeferredImportSelector. 如果正在收集导入器,则将此梳理注册到列表中.
 * 如果他们正在被处理,DeferredImportSelector 也会根据DeferredImportSelector.Group来立即处理他们
 * @param configClass the source configuration class
 * @param importSelector the selector to handle
 */
public void handle(ConfigurationClass configClass, DeferredImportSelector importSelector) {
    DeferredImportSelectorHolder holder = new DeferredImportSelectorHolder(
            configClass, importSelector);
    //如果deferredImportSelectors为空 执行注册和处理逻辑  不为空则添加到deferredImportSelectors中
    if (this.deferredImportSelectors == null) {
        DeferredImportSelectorGroupingHandler handler = new DeferredImportSelectorGroupingHandler();
        handler.register(holder);
        handler.processGroupImports();
    }
    else {
        this.deferredImportSelectors.add(holder);
    }
}
```
&emsp;&emsp;因为 __deferredImportSelectors__ 已经初始化过,所以直接将 __DeferredImportSelectorHolder__ 添加到 __deferredImportSelectors__ 中供后续加载自动配置类使用.  
   - 候选者实现了 __ImportBeanDefinitionRegistrar__ 接口:  
&emsp;&emsp;候选者对应启动类中声明的 __AutoConfigurationPackages.Registrar.class__ 类,将此类实例化为 __ImportBeanDefinitionRegistrar__ 并添加到配置类的 __importBeanDefinitionRegistrars__ 集合中供后续使用.
   - 候选者未实现以上两个接口:  
&emsp;&emsp;将候选者当做普通配置类调用 __processConfigurationClass__ 方法递归解析.  

3.5 第五步,处理所有 __@ImportResource__ 注解,相关源码如下:  
``` java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
        throws IOException {
    ...
    // Process any @ImportResource annotations
    //3.5 处理所有存在的@ImportResource注解
    AnnotationAttributes importResource =
            AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
    if (importResource != null) {
        String[] resources = importResource.getStringArray("locations");
        Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
        for (String resource : resources) {
            String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
            configClass.addImportedResource(resolvedResource, readerClass);
        }
    }

    ...
}
```
&emsp;&emsp;可见这段逻辑就相对简单一点了,获取 __@ImportResource__ 的元数据,没有则直接跳过,有则继续获取 __locations__ 属性获取需要读取的配置文件地址,然后将读取的配置文件资源加载到配置类中.    
3.6 第六步,处理所有的 __@Bean__ 注解:
``` java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
        throws IOException {
    ...

    // Process individual @Bean methods
    //3.6 处理所有独立的@Bean方法
    //3.6.1 检索源类中所有的@Bean方法
    Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
    //3.6.2 将方法封装成BeanMethod并添加到配置类中
    for (MethodMetadata methodMetadata : beanMethods) {
        configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
    }

    ...
}
```
&emsp;&emsp;可见主要就分为两步,检索和添加.  
3.6.1 __retrieveBeanMethodMetadata__ 的作用就是检索所有的 __@Bean__ 方法,源码如下:
``` java
/**
 * Retrieve the metadata for all <code>@Bean</code> methods.
 * 检索所有@Bean方法的元数据
 */
private Set<MethodMetadata> retrieveBeanMethodMetadata(SourceClass sourceClass) {
    //获取源类的元数据
    AnnotationMetadata original = sourceClass.getMetadata();
    //获取所有@bean注解的方法
    Set<MethodMetadata> beanMethods = original.getAnnotatedMethods(Bean.class.getName());
    if (beanMethods.size() > 1 && original instanceof StandardAnnotationMetadata) {
        // Try reading the class file via ASM for deterministic declaration order...
        //尝试通过ASM读取类文件以获得确定性声明顺序
        // Unfortunately, the JVM's standard reflection returns methods in arbitrary
        // order, even between different runs of the same application on the same JVM.
        //不幸的是 JVM标准的反射以任意顺序返回方法,即使在同一jvm上同一应用程序的不同运行之间也是如此
        try {
            AnnotationMetadata asm =
                    this.metadataReaderFactory.getMetadataReader(original.getClassName()).getAnnotationMetadata();
            Set<MethodMetadata> asmMethods = asm.getAnnotatedMethods(Bean.class.getName());
            if (asmMethods.size() >= beanMethods.size()) {
                Set<MethodMetadata> selectedMethods = new LinkedHashSet<>(asmMethods.size());
                for (MethodMetadata asmMethod : asmMethods) {
                    for (MethodMetadata beanMethod : beanMethods) {
                        if (beanMethod.getMethodName().equals(asmMethod.getMethodName())) {
                            selectedMethods.add(beanMethod);
                            break;
                        }
                    }
                }
                if (selectedMethods.size() == beanMethods.size()) {
                    // All reflection-detected methods found in ASM method set -> proceed
                    beanMethods = selectedMethods;
                }
            }
        }
        catch (IOException ex) {
            logger.debug("Failed to read class file via ASM for determining @Bean method order", ex);
            // No worries, let's continue with the reflection metadata we started with...
        }
    }
    return beanMethods;
}
```
&emsp;&emsp;可见就是从源类的元数据中获取所有的 __@Bean__ 方法,并且当出现多个方法时需要根据ASM来确定方法的顺序.  
3.6.2 就是将3.6.1检索出来的所有的方法封装成 __BeanMethod__ 并添加到配置类中  
3.7 第七步,处理源类实现的接口上的所有方法:
``` java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
        throws IOException {
    ...

    // Process default methods on interfaces
    //3.7 处理接口上的默认方法
    processInterfaces(configClass, sourceClass);

    ...
}

/**
 * Register default methods on interfaces implemented by the configuration class.
 * 在配置类实现的接口上注册默认方法
 */
private void processInterfaces(ConfigurationClass configClass, SourceClass sourceClass) throws IOException {
    //遍历源类实现的所有接口
    for (SourceClass ifc : sourceClass.getInterfaces()) {
        //获取接口上的所有 @Bean方法
        Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(ifc);
        //遍历所有的方法 将非抽象的方法封装为BeanMethod方法并添加到配置类中
        for (MethodMetadata methodMetadata : beanMethods) {
            if (!methodMetadata.isAbstract()) {
                // A default method or other concrete method on a Java 8+ interface...
                //java8接口上的默认方法或其他具体方法
                configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
            }
        }
        //递归处理接口
        processInterfaces(configClass, ifc);
    }
}
```
&emsp;&emsp;根据3.6来看这个就很好理解了,3.6处理的是源类中的所有 __@Bean__ 方法,而3.7处理的是所有接口上的 __@Bean__ 方法,并通过递归的方式保证所有接口都处理完毕.  
3.8 第八步,处理父类:
``` java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
        throws IOException {
    ...

    // Process superclass, if any
    //3.8 如果有的话处理父类
    if (sourceClass.getMetadata().hasSuperClass()) {
        String superclass = sourceClass.getMetadata().getSuperClassName();
        if (superclass != null && !superclass.startsWith("java") &&
                !this.knownSuperclasses.containsKey(superclass)) {
            this.knownSuperclasses.put(superclass, configClass);
            // Superclass found, return its annotation metadata and recurse
            return sourceClass.getSuperClass();
        }
    }

    // No superclass -> processing is complete
    //没有父类的话 处理完成 返回null
    return null;
}
```
&emsp;&emsp;其实到这一步,当前配置类的注解解析已经完成了,若配置类有父类的话直接将父类返回继续进行配置类解析,若没有直接返回null即可.  
>以上就是doProcessConfigurationClass的所有流程,即解析@Component、@PropertySource、@ComponentScan、@Import、@ImportResource、@Bean及其父类和实现的接口的所有注解.

- - -

### deferredImportSelectorHandler.process()
- - -

&emsp;&emsp;ConfigurationClassParser.parse的第一步就是解析配置类,第二步即deferredImportSelectorHandler.process().
&emsp;&emsp;首先抛出结论,主要是针对 __AutoConfigurationImportSelector__ 用来加载所有的EnableAutoConfiguration的子类.  
&emsp;&emsp;源码如下:
``` java
public void process() {
    List<DeferredImportSelectorHolder> deferredImports = this.deferredImportSelectors;
    this.deferredImportSelectors = null;
    try {
        if (deferredImports != null) {
            DeferredImportSelectorGroupingHandler handler = new DeferredImportSelectorGroupingHandler();
            deferredImports.sort(DEFERRED_IMPORT_COMPARATOR);
            //1. 通过DeferredImportSelectorGroupingHandler注册AutoConfigurationGroup分组
            deferredImports.forEach(handler::register);
            //2. 处理分组的导入配置
            handler.processGroupImports();
        }
    }
    finally {
        this.deferredImportSelectors = new ArrayList<>();
    }
}
```
&emsp;&emsp;首先先看第一步:
``` java
public void register(DeferredImportSelectorHolder deferredImport) {
    //获取当前DeferredImportSelector的分组
    Class<? extends Group> group = deferredImport.getImportSelector().getImportGroup();
    //将此分组添加到grouping中
    DeferredImportSelectorGrouping grouping = this.groupings.computeIfAbsent(
            (group != null ? group : deferredImport),
            key -> new DeferredImportSelectorGrouping(createGroup(group)));
    grouping.add(deferredImport);
    this.configurationClasses.put(deferredImport.getConfigurationClass().getMetadata(),
            deferredImport.getConfigurationClass());
}

public Class<? extends Group> getImportGroup() {
		return AutoConfigurationGroup.class;
	}
```
&emsp;&emsp;可见第一步的作用就是获取 __AutoConfigurationGroup__ 自动配置分组,为后续获取自动配置类做准备.  
&emsp;&emsp;接下来看关键的 __handler.processGroupImports()__ 处理逻辑:
``` java
public void processGroupImports() {
        //groupings中就存放着我们上一步添加的AutoConfigurationGroup
        for (DeferredImportSelectorGrouping grouping : this.groupings.values()) {
            //根据getImports()获取所有的配置类  再通过处理Import配置类的方法processImports将每个EnableAutoConfiguration的实现类当做@Configuration解析
            grouping.getImports().forEach(entry -> {
                ConfigurationClass configurationClass = this.configurationClasses.get(
                        entry.getMetadata());
                try {
                    //将自动配置继续执行@Import的处理逻辑 默认都当做普通配置类解析
                    processImports(configurationClass, asSourceClass(configurationClass),
                            asSourceClasses(entry.getImportClassName()), false);
                }
                catch (BeanDefinitionStoreException ex) {
                    throw ex;
                }
                catch (Throwable ex) {
                    throw new BeanDefinitionStoreException(
                            "Failed to process import candidates for configuration class [" +
                                    configurationClass.getMetadata().getClassName() + "]", ex);
                }
            });
        }
    }
```
&emsp;&emsp;可见此段逻辑为获取分组中的导入选择器,获取所有需要导入的配置类重新执行 __processImports__ 逻辑,这里的配置类都会都默认的配置类解析逻辑.  
&emsp;&emsp;我们继续看它是如何获取分组的导入配置类的:
``` java
/**
 * Return the imports defined by the group.
 * 返回通过组定义的导入类
 * @return each import with its associated configuration class
 */
public Iterable<Group.Entry> getImports() {
    for (DeferredImportSelectorHolder deferredImport : this.deferredImports) {
        //调用org.springframework.boot.autoconfigure.AutoConfigurationImportSelector.AutoConfigurationGroup#process 获取所有的EnableAutoConfiguration候选配置类
        this.group.process(deferredImport.getConfigurationClass().getMetadata(),
                deferredImport.getImportSelector());
    }
    //返回满足条件的导入类
    return this.group.selectImports();
}
```
&emsp;&emsp;此方法主要是 __group.process__ 和 __group.selectImports__ , __group.process__ 的作用是通过SPI机制获取所有的EnableAutoConfiguration候选配置类,而 __group.selectImports__ 就是筛选出满足条件的配置类.  
&emsp;&emsp;继续看 __group.process__ :
``` java
public void process(AnnotationMetadata annotationMetadata, DeferredImportSelector deferredImportSelector) {
    Assert.state(deferredImportSelector instanceof AutoConfigurationImportSelector,
            () -> String.format("Only %s implementations are supported, got %s",
                    AutoConfigurationImportSelector.class.getSimpleName(),
                    deferredImportSelector.getClass().getName()));
    //通过SPI机制获取满足条件的EnableAutoConfiguration的实现类
    AutoConfigurationEntry autoConfigurationEntry = ((AutoConfigurationImportSelector) deferredImportSelector)
            .getAutoConfigurationEntry(getAutoConfigurationMetadata(), annotationMetadata);
    this.autoConfigurationEntries.add(autoConfigurationEntry);
    for (String importClassName : autoConfigurationEntry.getConfigurations()) {
        this.entries.putIfAbsent(importClassName, annotationMetadata);
    }
}

/**
 * Return the {@link AutoConfigurationEntry} based on the {@link AnnotationMetadata}
 * of the importing {@link Configuration @Configuration} class.
 * 根据导入的Configuration注解类的AnnotationMetadata返回AutoConfigurationEntry
 * @param autoConfigurationMetadata the auto-configuration metadata
 * @param annotationMetadata the annotation metadata of the configuration class
 * @return the auto-configurations that should be imported
 */
protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata,
        AnnotationMetadata annotationMetadata) {
    //判断是否开启自动配置
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
    }
    //获取@EnableAutoConfiguration注解的属性 即exclude和excludeName
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    //通过SpringFactoriesLoader.loadFactoryNames#META-INF/spring.factories中查找所有的EnableAutoConfiguration候选配置类
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    //通过LinkedHashSet去重
    configurations = removeDuplicates(configurations);
    //获取所有排除的包
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    checkExcludedClasses(configurations, exclusions);
    configurations.removeAll(exclusions);
    //获取配置类过滤器 从META-INF/spring.factories中加载AutoConfigurationImportFilter的实现类
    configurations = filter(configurations, autoConfigurationMetadata);
    //处理自动配置导入事件
    fireAutoConfigurationImportEvents(configurations, exclusions);
    return new AutoConfigurationEntry(configurations, exclusions);
}

protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    //获取所有EnableAutoConfigurationde实现类名
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
            getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
            + "are using a custom packaging, make sure that file is correct.");
    return configurations;
}

protected Class<?> getSpringFactoriesLoaderFactoryClass() {
    return EnableAutoConfiguration.class;
}
```
&emsp;&emsp;由上述调用链就可以很清晰的看出 __getAutoConfigurationEntry__ 通过SPI机制获取 __EnableAutoConfiguration__ 的自动配置实现类.  
&emsp;&emsp;再来看下 __group.selectImports__ :
``` java
public Iterable<Entry> selectImports() {
        if (this.autoConfigurationEntries.isEmpty()) {
            return Collections.emptyList();
        }
        //所有需要排除的配置
        Set<String> allExclusions = this.autoConfigurationEntries.stream()
                .map(AutoConfigurationEntry::getExclusions).flatMap(Collection::stream).collect(Collectors.toSet());
        //找到的所有配置
        Set<String> processedConfigurations = this.autoConfigurationEntries.stream()
                .map(AutoConfigurationEntry::getConfigurations).flatMap(Collection::stream)
                .collect(Collectors.toCollection(LinkedHashSet::new));
        //移除需要排除的
        processedConfigurations.removeAll(allExclusions);
        //重新排序
        return sortAutoConfigurations(processedConfigurations, getAutoConfigurationMetadata()).stream()
                .map((importClassName) -> new Entry(this.entries.get(importClassName), importClassName))
                .collect(Collectors.toList());
    }
```
&emsp;&emsp;以上步骤就很清晰了,移除所有指定的配置,并将实现剩下的配置重新排序返回.
>可见deferredImportSelectorHandler.process()就是SpringBoot的自动配置功能,找到/META-INF/spring.factories中所有的EnableAutoConfiguration实现类,并解析这些配置类.

- - -

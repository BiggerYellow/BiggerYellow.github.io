---
layout: default
springboot: true
modal-id: 20000001
date: 2021-07-22
img: pexels-david-selbert-6468061.jpg
alt: image-alt
project-date: April 2021
client: Start Bootstrap
category: springboot
subtitle: SpringBoot启动源码解析(一)
description: SpringBootApplication注解分析
---
### @SpringBootApplication结构图
- - -
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/@SpringBootApplication注解.jpg">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="@SpringBootApplication注解"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/@SpringBootApplication注解.jpg">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">@SpringBootApplication注解</div>
    </a>
</center>

### @SpringBootApplication
- - -
> 源码如下

``` java
/**
 * Indicates a {@link Configuration configuration} class that declares one or more
 * {@link Bean @Bean} methods and also triggers {@link EnableAutoConfiguration
 * auto-configuration} and {@link ComponentScan component scanning}. This is a convenience
 * annotation that is equivalent to declaring {@code @Configuration},
 * {@code @EnableAutoConfiguration} and {@code @ComponentScan}.
 * 声明一个配置类,他声明一个或多个@Bean方法且触发 EnableAutoConfiguration自动配置和 ComponentScan组件扫描.
 * 这是一个便利的注解等同于声明 @Configuration、@EnableAutoConfiguration 和 @ComponentScan
 * @author Phillip Webb
 * @author Stephane Nicoll
 * @author Andy Wilkinson
 * @since 1.2.0
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
//该注解的作用是开启自动配置
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

	@AliasFor(annotation = EnableAutoConfiguration.class)
	Class<?>[] exclude() default {};

	@AliasFor(annotation = EnableAutoConfiguration.class)
	String[] excludeName() default {};

	@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
	String[] scanBasePackages() default {};

	@AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
	Class<?>[] scanBasePackageClasses() default {};

	@AliasFor(annotation = Configuration.class)
	boolean proxyBeanMethods() default true;

}
```  
__@SpringBootApplication__ 是最顶层注解,他主要由 __@SpringBootConfiguration__ , __@EnableAutoConfiguration__ , __@Component__ 这个三个注解组合而成.   
接下来我们仔细分析下这三个注解分别做了什么事情.  
- - -

### @SpringBootConfiguration
- - -
> 源码如下

``` java
/**
 * Indicates that a class provides Spring Boot application
 * {@link Configuration @Configuration}. Can be used as an alternative to the Spring's
 * standard {@code @Configuration} annotation so that configuration can be found
 * automatically (for example in tests).
 * 表明一个类提供Springboot应用的@Configuration.用来替代Spring标准的@Configuration注解，可以自动发现配置

 * <p>
 * Application should only ever include <em>one</em> {@code @SpringBootConfiguration} and
 * most idiomatic Spring Boot applications will inherit it from
 * {@code @SpringBootApplication}.
 * 应用应该包含一个@SpringBootConfiguration注解并且大多数惯用的SpringBoot应用将继承@SpringBootConfiguration
 * @author Phillip Webb
 * @author Andy Wilkinson
 * @since 1.4.0
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {

	@AliasFor(annotation = Configuration.class)
	boolean proxyBeanMethods() default true;

}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {

	@AliasFor(annotation = Component.class)
	String value() default "";

	boolean proxyBeanMethods() default true;

}
```

可见 __@SpringBootApplication__ 的底层注解就是 __@Configuration__ , 而 __@Component__ 的底层注解是 __@Component__ .   
__@SpringBootApplication__ 的主要作用就是将目标类标记为配置类,交由Spring管理.在SpingBoot中就对应我们的启动函数.    
下面我们看下这个配置主要用于哪里?   
- 首先用于将主函数添加到 __DefaultListableBeanFactory__ 中的 __beanDefinitionMap__ 和 __beanDefinitionNames__ 中,即将主函数注册为上下文中bean工厂的bean定义.  

``` java
//代码位置
->  org.springframework.boot.SpringApplication#prepareContext   
->  org.springframework.boot.SpringApplication#load 
->  org.springframework.boot.BeanDefinitionLoader#load(java.lang.Class<?>)

private int load(Class<?> source) {
    if (isGroovyPresent() && GroovyBeanDefinitionSource.class.isAssignableFrom(source)) {
        // Any GroovyLoaders added in beans{} DSL can contribute beans here
        GroovyBeanDefinitionSource loader = BeanUtils.instantiateClass(source, GroovyBeanDefinitionSource.class);
        load(loader);
    }
    // 这里判断source(即主函数)是否被@Component修饰
    if (isComponent(source)) {
        this.annotatedReader.register(source);
        return 1;
    }
    return 0;
}

private boolean isComponent(Class<?> type) {
    // This has to be a bit of a guess. The only way to be sure that this type is
    // eligible is to make a bean definition out of it and try to instantiate it.
    if (MergedAnnotations.from(type, SearchStrategy.TYPE_HIERARCHY).isPresent(Component.class)) {
        return true;
    }
    // Nested anonymous classes are not eligible for registration, nor are groovy
    // closures
    return !type.getName().matches(".*\\$_.*closure.*") && !type.isAnonymousClass()
            && type.getConstructors() != null && type.getConstructors().length != 0;
}

```

- 最后就是将main方法对应的类解析为配置类, 通过 __ConfigurationClassPostProcessor__ 配置类后置处理器解析bean工厂中所有被 __@Configuration__ 修饰的bean  

``` java
//代码位置
->  1.org.springframework.boot.SpringApplication#refreshContext
->  2.org.springframework.context.support.AbstractApplicationContext#invokeBeanFactoryPostProcessors
->  3.org.springframework.context.support.PostProcessorRegistrationDelegate#invokeBeanDefinitionRegistryPostProcessors
->  4.org.springframework.context.annotation.ConfigurationClassPostProcessor#processConfigBeanDefinitions

/**
 * Build and validate a configuration model based on the registry of
 * {@link Configuration} classes.
 */
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
    String[] candidateNames = registry.getBeanDefinitionNames();

    for (String beanName : candidateNames) {
        BeanDefinition beanDef = registry.getBeanDefinition(beanName);
        if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
            if (logger.isDebugEnabled()) {
                logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
            }
        }
        //此步就是判断当前类是否
        else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
            configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
        }
    }

    // Return immediately if no @Configuration classes were found
    if (configCandidates.isEmpty()) {
        return;
    }
    
    ...
}
```
综上所述, __@SpringBootApplication__ 的作用就是将主函数对应类声明为可以被Spring处理的bean,并解析该bean交由Spring管理.  

- - -

### @EnableAutoConfiguration
- - -
> 源码如下

``` java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
//将主配置类所在的包及子包里面所有的组件扫描加载到Spring容器中
@AutoConfigurationPackage
//通过selectImports将自动配置类xxxAutoConfiguration导入到容器中
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	//	 * 当自动配置开启时可以用于重写的环境属性
	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	Class<?>[] exclude() default {};

	String[] excludeName() default {};

}

/**
 * Indicates that the package containing the annotated class should be registered with
 * {@link AutoConfigurationPackages}.
 * 使用AutoConfigurationPackages注册。如果未指定基础包或基础包的类，使用注解的类将被注册
 * @author Phillip Webb
 * @since 1.3.0
 * @see AutoConfigurationPackages
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {

}
```

可见 __@EnableAutoConfiguration__ 是由 __@AutoConfigurationPackage__ 和 __@Import(AutoConfigurationImportSelector.class)__ 组成, 而 __@AutoConfigurationPackage__ 又是由 __@Import(AutoConfigurationPackages.Registrar.class)__ 组成.    
所以 __@EnableAutoConfiguration__ 其实就是由两个 __@Import__ 注解组成的, 顾名思义它的作用就是导入指定类的,即 __AutoConfigurationImportSelector.class__ 和 __AutoConfigurationPackages.Registrar.class__ .  
接下来我们看下这两个类的作用是什么,具体在哪里生效的?  

``` java
//调用路径
->  1.org.springframework.boot.SpringApplication#run(java.lang.String...)
->  2.org.springframework.boot.SpringApplication#refreshContext
->  3.org.springframework.context.support.AbstractApplicationContext#refresh
->  4.org.springframework.context.support.AbstractApplicationContext#invokeBeanFactoryPostProcessors
->  5.org.springframework.context.support.PostProcessorRegistrationDelegate#invokeBeanDefinitionRegistryPostProcessors
->  6.org.springframework.context.annotation.ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry
->  7.org.springframework.context.annotation.ConfigurationClassPostProcessor#processConfigBeanDefinitions
->  8.org.springframework.context.annotation.ConfigurationClassParser#parse(org.springframework.core.type.AnnotationMetadata, java.lang.String)
->  9.org.springframework.context.annotation.ConfigurationClassParser#doProcessConfigurationClass
->  10.org.springframework.context.annotation.ConfigurationClassParser#processImports  //核心调用点 获取spring.factories中所有的EnableAutoConfiguration的子类
->  11.org.springframework.context.annotation.ConfigurationClassParser.DeferredImportSelectorHandler#process
->  12.org.springframework.context.annotation.ConfigurationClassParser.DeferredImportSelectorGroupingHandler#processGroupImports //将所有满足条件的EnableAutoConfiguration的子类注册为bean

//关键代码分析
@Nullable
protected final SourceClass doProcessConfigurationClass(
        ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
        throws IOException {
    ...

    // Process any @Import annotations
    processImports(configClass, sourceClass, getImports(sourceClass), filter, true);
    
    ...

}

/**
 * Returns {@code @Import} class, considering all meta-annotations.
 * 返回所有@Import的类, 考虑所有元注解
 */
private Set<SourceClass> getImports(SourceClass sourceClass) throws IOException {
    Set<SourceClass> imports = new LinkedHashSet<>();
    Set<SourceClass> visited = new LinkedHashSet<>();
    collectImports(sourceClass, imports, visited);
    return imports;
}

//这个方法就是处理所有@Import的类
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
        Collection<SourceClass> importCandidates, Predicate<String> exclusionFilter,
        boolean checkForCircularImports) {

    if (importCandidates.isEmpty()) {
        return;
    }

    if (checkForCircularImports && isChainedImportOnStack(configClass)) {
        this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
    }
    else {
        this.importStack.push(configClass);
        try {
            for (SourceClass candidate : importCandidates) {
                //候选类是否属于ImportSelector 主要针对AutoConfigurationImportSelector.class
                //核心流程:
                //1.this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
                //2.org.springframework.context.annotation.ConfigurationClassParser.DeferredImportSelectorGroupingHandler#processGroupImports
                //3.org.springframework.context.annotation.ConfigurationClassParser.DeferredImportSelectorGrouping#getImports
                //4.org.springframework.boot.autoconfigure.AutoConfigurationImportSelector.AutoConfigurationGroup#process
                //5.org.springframework.boot.autoconfigure.AutoConfigurationImportSelector#getAutoConfigurationEntry
                //6.org.springframework.boot.autoconfigure.AutoConfigurationImportSelector#getCandidateConfigurations 最重要的就是这里获得EnableAutoConfiguration的所有子类
                if (candidate.isAssignable(ImportSelector.class)) {
                    // Candidate class is an ImportSelector -> delegate to it to determine imports
                    Class<?> candidateClass = candidate.loadClass();
                    ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class,
                            this.environment, this.resourceLoader, this.registry);
                    Predicate<String> selectorFilter = selector.getExclusionFilter();
                    if (selectorFilter != null) {
                        exclusionFilter = exclusionFilter.or(selectorFilter);
                    }
                    if (selector instanceof DeferredImportSelector) {
                        this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
                    }
                    else {
                        String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                        Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames, exclusionFilter);
                        processImports(configClass, currentSourceClass, importSourceClasses, exclusionFilter, false);
                    }
                }
                //候选类是否属于ImportBeanDefinitionRegistrar  主要针对 AutoConfigurationPackages.Registrar.class
                //将AutoConfigurationPackages.Registrar.class实例化,并通过解析后的org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader#loadBeanDefinitions方法
                //中的org.springframework.context.annotation.ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsFromRegistrars方法注册为bean实例
                else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                    // Candidate class is an ImportBeanDefinitionRegistrar ->
                    // delegate to it to register additional bean definitions
                    Class<?> candidateClass = candidate.loadClass();
                    ImportBeanDefinitionRegistrar registrar =
                            ParserStrategyUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class,
                                    this.environment, this.resourceLoader, this.registry);
                    configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                }
                //候选类不属于以上两个父类  直接进行配置类解析逻辑
                else {
                    // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
                    // process it as an @Configuration class
                    this.importStack.registerImport(
                            currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                    processConfigurationClass(candidate.asConfigClass(configClass), exclusionFilter);
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
综上所述, __AutoConfigurationImportSelector.class__ 的作用就是找到所有 __EnableAutoConfiguration__ 的自动配置子类,并注册为bean定义.  
而 __AutoConfigurationPackages.Registrar.class__ 的作用就是扫描主配置类同级目录以及子包将所有组件到Spring中 

- - -

### @ComponentScan
> 源码

``` java
@Nullable
protected final SourceClass doProcessConfigurationClass(
        ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
        throws IOException {
        ...

	    // Process any @ComponentScan annotations
        //处理所有@ComponentScan注解  默认扫描主配置类同级目录及子包
		Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
		if (!componentScans.isEmpty() &&
				!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
			for (AnnotationAttributes componentScan : componentScans) {
				// The config class is annotated with @ComponentScan -> perform the scan immediately
				Set<BeanDefinitionHolder> scannedBeanDefinitions =
						this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
				// Check the set of scanned definitions for any further config classes and parse recursively if needed
				for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
					BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
					if (bdCand == null) {
						bdCand = holder.getBeanDefinition();
					}
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
						parse(bdCand.getBeanClassName(), holder.getBeanName());
					}
				}
			}
		}

        ...

}
```
综上所述: __@ComponentScan__ 的作用就是扫描主配置目录及子包,并将扫描到的配置类进行解析.  

- - -

### 总结
1. @SpringBootApplication:将主函数声明为可以被Spring管理的bean,并交给Spring解析处理.
2. @ComponentScan:扫描主配置目录及子包,并将扫描到的配置类进行解析
3. @EnableAutoConfiguration:自动注册所有自动配置的配置类  
    - @Import(AutoConfigurationImportSelector.class):找到spring.factories中所有EnableAutoConfiguration的子类并按条件过滤注册为bean
    - @Import(AutoConfigurationPackages.Registrar.class):将主配置及子包下的所有组件扫描到Spring中



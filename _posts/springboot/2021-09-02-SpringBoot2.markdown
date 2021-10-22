---
layout: default 
springboot: true  
modal-id: 20000002
date: 2021-09-02
img: pexels-irina-iriser-1122625.jpg
alt: SpringBoot2
project-date: September 2021
client: SpringBoot
category: SpringBoot
subtitle: SpringBoot启动源码解析(二)
description: SpringBootApplication构造方法分析
---
### 主函数流程图
- - -
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/SpringApplication构造流程图.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="SpringApplication构造流程图"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/SpringApplication构造流程图.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">SpringApplication构造流程图</div>
    </a>
</center>
- - -
### 主函数入口
- - -
我们首先从主函数入口一步一步往下看.

``` java
//程序主函数
public static void main(String[] args) {
    SpringApplication.run(DemoApplication.class, args);
}

/**
 * Static helper that can be used to run a {@link SpringApplication} from the
 * specified source using default settings.
 * 用于在一个使用默认配置的指定资源中运行 SpringApplication 的静态工具类
 * @param primarySource the primary source to load
 * @param args the application arguments (usually passed from a Java main method)
 * @return the running {@link ApplicationContext}
 */
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
    return run(new Class<?>[] { primarySource }, args);
}

/**
 * Static helper that can be used to run a {@link SpringApplication} from the
 * specified sources using default settings and user supplied arguments.
 * @param primarySources the primary sources to load
 * @param args the application arguments (usually passed from a Java main method)
 * @return the running {@link ApplicationContext}
 */
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    return new SpringApplication(primarySources).run(args);
}
```

核心就是 __new SpringApplication(primarySources).run(args)__ 这一行.  
我们首先分析一下  __new SpringApplication(primarySources)__.  

### new SpringApplication(primarySources)主构造
- - -
``` java
/**
 * Create a new {@link SpringApplication} instance. The application context will load
 * beans from the specified primary sources (see {@link SpringApplication class-level}
 * documentation for details. The instance can be customized before calling
 * {@link #run(String...)}.
 * 创建一个SpringApplication实例. 应用上下文将从指定主函数中加载bean. (这个实例可以在调用run方法前自定义)
 * @param resourceLoader the resource loader to use
 * @param primarySources the primary bean sources
 * @see #run(Class, String[])
 * @see #setSources(Set)
 */
@SuppressWarnings({ "unchecked", "rawtypes" })
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    //1.设置类加载器
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    //2.将主函数封装成集合设置到primarySources中 可以设置多个主类
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    //3.判断当前应用类型 通常使用的都是WebApplicationType.SERVLET
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    //4.获取并实例化所有的ApplicationContextInitializer实现 主要用于在上下文刷新前进行初始化上下文
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    //5.获取并实例化所有的ApplicationListener实现 主要用于监听启动过程中发布的各种事件
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    //6.生成main对应主函数的类
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

在SpringApplication构造函数中,最重要就是步骤4和步骤5,再看下这两步其实是一样的实现,只不过加载的类不同而已.  
我们再仔细看看这个方法到底做了什么事情:  

``` java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates
    //获取指定类型的实现类名称集合
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    //实例化上面找到的所有实现类
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    //根据实例的order进行排序
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```
此方法主要做的就是找到指定类型的实现类并将这些类都进行实例化.我们再来看看它是如何找到这个类的.
``` java
/**
 * Load the fully qualified class names of factory implementations of the
 * given type from {@value #FACTORIES_RESOURCE_LOCATION}, using the given
 * class loader.
 * 使用给定的类加载器,从 META-INF/spring.factories中加载给定类型的工厂实现的完全限定名
 * @param factoryType the interface or abstract class representing the factory
 * @param classLoader the ClassLoader to use for loading resources; can be
 * {@code null} to use the default
 * @throws IllegalArgumentException if an error occurs while loading factory names
 * @see #loadFactories
 */
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
    String factoryTypeName = factoryType.getName();
    return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
}

/**
 * The location to look for factories.
 * 用于查询工厂的位置
 * <p>Can be present in multiple JAR files.
 * 可以存在于多个jar文件中
 */
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    //首先尝试从缓存中读取,因为该方法执行结束后会将配置文件中所有的信息放到缓存中,如果缓存中有直接返回
    MultiValueMap<String, String> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    try {
        //读取项目中所有存在 META-INF/spring.factories的URL
        Enumeration<URL> urls = (classLoader != null ?
                classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        result = new LinkedMultiValueMap<>();
        //读取每一个URL并将属性以K-V的形式保存到Properties中, 再将所有属性保存在result中
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();
                for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                    result.add(factoryTypeName, factoryImplementationName.trim());
                }
            }
        }
        //将result保存到缓存中
        cache.put(classLoader, result);
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" +
                FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```
通过代码分析发现, __loadSpringFactories__ 的主要作用就是读取所有 __META-INF/spring.factories__ 中的所有配置信息并保存到缓存中, 并返回指定类型的集合.  
现在我们已经找到指定类的所有实现类了,接下来我们就应该实例化它们.  
``` java
private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes,
        ClassLoader classLoader, Object[] args, Set<String> names) {
    List<T> instances = new ArrayList<>(names.size());
    for (String name : names) {
        try {
            //加载每一个class类
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            Assert.isAssignable(type, instanceClass);
            Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
            //使用反射构造该类实例 通常采用默认构造
            T instance = (T) BeanUtils.instantiateClass(constructor, args);
            instances.add(instance);
        }
        catch (Throwable ex) {
            throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, ex);
        }
    }
    return instances;
}
```

如果你经常使用反射的话,这段代码就比较容易理解了,无非就是使用反射创建类实例.  

### 总结
- - -
由上可知,在 __SpringApplication__ 构造中主要是做了设置主函数资源、判断当前应用类型和加载并实例化所有 __ApplicationContextInitializer__ 和 __ApplicationListener__ 的实现类. 
- - -

### 扩展
- - -
主要就是针对 __ApplicationListener__ 和 __ApplicationContextInitializer__ 进行扩展.两者扩展类似,那我就以 __ApplicationListener__ 为例.  

1.第一步肯定是自己创建一个测试类 __MyApplicationListener__ 实现 __ApplicationListener__ ,这里需要注意下我们需要指定需要监听的事件类型,主要类图见下  
2.第二步就是实现需要重写的接口,在接口中添加自己的实现,我们就简单打一行日志  

<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/MyApplicationListener.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="MyApplicationListener"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/MyApplicationListener.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">MyApplicationListener</div>
    </a>
</center>

3.第三步也就是最重要的一步,你想想我们现在已经自己新建了类,但是应用还是无法找你的类,那么我们就应该让应用扫描到新类,这样才能在调用的时候调用我们自己重写的方法.  
那么就需要我们在 __resources__ 目录下新建 __META-INF/spring.factories__ 文件,并在该文件中指定你的实现类,这样就能让应用加载到你的类  
 
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/spring.factories.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="spring.factories"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/spring.factories.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">spring.factories</div>
    </a>
</center>

- - -
__ApplicationEvent__ 对应子类类图如下:
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/ApplicationEvent.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="ApplicationEvent"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/ApplicationEvent.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">ApplicationEvent</div>
    </a>
</center>

### SPI机制
- - -
#### 概念
- - -
其实上面的加载过程就是SPI机制,SpringBoot加以改造了一下.  
SPI的全名为 Service Provider Interface, 目的是提供接口,让第三方提供自定义实现的服务功能.目前不少框架使用它来做服务的扩展.  
- - -

#### 实现
- - -
1.自定义已经接口以及其实现类  

<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/spi基类.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="spi基类"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/spi基类.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">spi基类</div>
    </a>
</center>

2.在 __META-INF/services__ 目录中创建以接口全限定名命名的文件,文件内容为接口具体实现的全限定名  

<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/services.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="META-INF/services"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/services.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">META-INF/services</div>
    </a>
</center>

3.使用 __ServiceLoader.load(Class<T>)__ 类动态加载目录下的实现类  

<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/main.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="main"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/main.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">main</div>
    </a>
</center>

4.具体实现类必须有一个不带参数的构造方法  

- - -


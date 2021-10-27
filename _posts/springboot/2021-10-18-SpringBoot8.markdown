---
layout: default
springboot: true
modal-id: 20000008
date: 2021-10-18
img: pexels-joaqu-9677214.jpg
alt: SpringBoot
project-date: October 2021
client: SpringBoot
category: SpringBoot
subtitle: SpringBoot启动源码解析(八)
description: SpringBootApplication getBean方法解析:org.springframework.beans.factory.support.AbstractBeanFactory#getBean(java.lang.String, java.lang.Class<T>)
---
### 创建bean流程图
- - -
<center>
    <a href="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/GetBean流程图.png">
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" class="img-responsive img-centered" alt="GetBean流程图"
    src="https://cdn.jsdelivr.net/gh/BiggerYellow/BiggerYellow.github.io/img/springboot/GetBean流程图.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">GetBean流程图</div>
    </a>
</center>
- - -

### 源码分析
- - -
&emsp;&emsp;getBean()的底层实现是doGetBean(),直接上源码:
``` java
/**
 * Return an instance, which may be shared or independent, of the specified bean.
 * 返回一个指定bean的实例,这个实例可能是共享的也可能是独立的
 * @param name the name of the bean to retrieve
 *             要检索的bean名称
 * @param requiredType the required type of the bean to retrieve
 *                     要检索的bean类型
 * @param args arguments to use when creating a bean instance using explicit arguments
 * (only applied when creating a new instance as opposed to retrieving an existing one)
 *             当创建bean实例时使用显示参数(只应用于创建新实例而不是检索现有实例的应用)
 * @param typeCheckOnly whether the instance is obtained for a type check,
 * not for actual use
 *                      获取实例是否是为了类型检查,而不是实际 使用
 * @return an instance of the bean
 * @throws BeansException if the bean could not be created
 */
@SuppressWarnings("unchecked")
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
        @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
    //1. 获取bean的名称 工厂bean需要去除前缀 &
    final String beanName = transformedBeanName(name);
    Object bean;

    // Eagerly check singleton cache for manually registered singletons.
    //急切的检查单例缓存是够有手动注册的单例
    //2. 尝试从三级缓存中获取
    Object sharedInstance = getSingleton(beanName);
    //3. 如果获取到实例 且 参数为null 则尝试获取bean
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
        //返回bean实例本身或从bean工厂中获取
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }

    else {
        //4. 如果获取不到实例 则进行常见
        // Fail if we're already creating this bean instance:
        // We're assumably within a circular reference.
        //5. 如果我们已经在创建这个bean实例将失败:大概率在循环引用中
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // Check if bean definition exists in this factory.
        //6. 检查bean定义是否存在工厂中
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // Not found -> check parent.
            //如果在父bean工厂中没有找到当前bean 则检查父工厂
            String nameToLookup = originalBeanName(name);
            if (parentBeanFactory instanceof AbstractBeanFactory) {
                return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                        nameToLookup, requiredType, args, typeCheckOnly);
            }
            else if (args != null) {
                // Delegation to parent with explicit args.
                //如果参数不为null,使用显示参数委托给父工厂
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else if (requiredType != null) {
                // No args -> delegate to standard getBean method.
                //没有参数 则 委托给标准的getBean方法
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
            else {
                return (T) parentBeanFactory.getBean(nameToLookup);
            }
        }
        //7. 如果只是类型检查  需要把bean标记为已创建(或将被创建) 即将beanName添加到alreadyCreated中
        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }

        try {
            //8. 再次获取合并的bean定义
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);

            // Guarantee initialization of beans that the current bean depends on.
            //9. 保证当前bean所依赖的bean的初始化
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }
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

            // Create bean instance.
            //10. 根据bean类型创建bean实例
            if (mbd.isSingleton()) {
                sharedInstance = getSingleton(beanName, () -> {
                    try {
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
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }

            else if (mbd.isPrototype()) {
                // It's a prototype -> create a new instance.
                //如果是原型 -》 创建一个新实例
                Object prototypeInstance = null;
                try {
                    beforePrototypeCreation(beanName);
                    prototypeInstance = createBean(beanName, mbd, args);
                }
                finally {
                    afterPrototypeCreation(beanName);
                }
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }

            else {
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
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
                    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                }
                catch (IllegalStateException ex) {
                    throw new BeanCreationException(beanName,
                            "Scope '" + scopeName + "' is not active for the current thread; consider " +
                            "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                            ex);
                }
            }
        }
        catch (BeansException ex) {
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }

    // Check if required type matches the type of the actual bean instance.
    //11. 检查需要的类型是否匹配真正bean实例的类型
    if (requiredType != null && !requiredType.isInstance(bean)) {
        try {
            T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
            if (convertedBean == null) {
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
            return convertedBean;
        }
        catch (TypeMismatchException ex) {
            if (logger.isTraceEnabled()) {
                logger.trace("Failed to convert bean '" + name + "' to required type '" +
                        ClassUtils.getQualifiedName(requiredType) + "'", ex);
            }
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    //12. 返回创建的bean 
    return (T) bean;
}
```
1. 获取真正的bean名称,去除工厂bean的前缀&
2. 首先尝试从三级缓存中获取
3. 如果可以获取到实例则继续尝试从bean实例本身或者工厂bean中获取
4. 如果取不到实例则执行创建bean逻辑
5. 首先检查该bean是否正在创建中
6. 检查是否存在父bean工厂,若存在且该bean未创建过,则尝试从父bean工厂中获取
7. 若不止类型检查,则标记该bean为已创建
8. 尝试再次获取合并的bean定义并检查其定义
9. 获取该bean依赖属性,存在则先注册这些依赖属性
10. 这是最重要的一步,根据bean的类型创建bean实例,我们重点关注一下单例的创建,这就是bean创建的声明下面我们会详细介绍
11. bean创建完后根据需要的类型进行类型转化
12. 最后返回创建完成的bean  

&emsp;&emsp;可见大约可以分为12步,具体的流程图可以参见上面的.  
&emsp;&emsp;接下来我们就来一步步看看这个doGetBean具体做了什么:  

1.第一步获取真正bean的名称

``` java
/**
 * Return the bean name, stripping out the factory dereference prefix if necessary,
 * and resolving aliases to canonical names.
 *返回bean名称，如果需要的话去除工厂引用的前缀 并将别名解析为规范名称
 */
protected String transformedBeanName(String name) {
    return canonicalName(BeanFactoryUtils.transformedBeanName(name));
}

String FACTORY_BEAN_PREFIX = "&";

/**
 * Return the actual bean name, stripping out the factory dereference
 * prefix (if any, also stripping repeated factory prefixes if found).
 * 返回实际的bean名称，去除工厂引用的前缀  (如果存在的话,也去除重复的工厂前缀 )
 * @param name the name of the bean
 * @return the transformed name
 * @see BeanFactory#FACTORY_BEAN_PREFIX
 */
public static String transformedBeanName(String name) {
    Assert.notNull(name, "'name' must not be null");
    //判断当前bean名称是否以 工厂标志 & 开头, 不是则直接返回
    if (!name.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
        return name;
    }
    //否则通过递归去除bean前面的 & 标识
    return transformedBeanNameCache.computeIfAbsent(name, beanName -> {
        do {
            beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
        }
        while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX));
        return beanName;
    });
}

/**
 * Determine the raw name, resolving aliases to canonical names.
 * 确定原始名称,将别名解析为规范名称
 * @param name the user-specified name
 * @return the transformed name
 */
public String canonicalName(String name) {
    String canonicalName = name;
    // Handle aliasing...
    String resolvedName;
    do {
        resolvedName = this.aliasMap.get(canonicalName);
        if (resolvedName != null) {
            canonicalName = resolvedName;
        }
    }
    while (resolvedName != null);
    return canonicalName;
}
```
&emsp;&emsp;可见transformedBeanName方法由canonicalName和transformedBeanName两个方法组成,transformedBeanName的作用就是判断bean名称是否包含&前缀,不包含直接返回,包含则进行截取.
而canonicalName的作用就是根据截取过的名称确定原始名称.  
2.首先尝试从三级缓存中获取bean
``` java
/**
 * Return the (raw) singleton object registered under the given name.
 * 返回通过给定的名称创建的原始单例对象
 * <p>Checks already instantiated singletons and also allows for an early
 * reference to a currently created singleton (resolving a circular reference).
 * 检查已经初始化的单例 并允许提前引用当前创建的单例 (解决循环引用)
 * @param beanName the name of the bean to look for
 * @param allowEarlyReference whether early references should be created or not
 *                            是否应该创建早期引用
 * @return the registered singleton object, or {@code null} if none found
 */
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    //从缓存单例的map查找指定单例
    Object singletonObject = this.singletonObjects.get(beanName);
    //如果当前beanName的实例对象存在则直接返回
    //如果不存在还要确定当前beanName是否正在创建中  即singletonsCurrentlyInCreation存在该beanName
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        //锁住存放单例的map
        synchronized (this.singletonObjects) {
            //看二级缓存中是否存在早期创建的单例
            singletonObject = this.earlySingletonObjects.get(beanName);
            //不存在且允许早期创建单例引用
            if (singletonObject == null && allowEarlyReference) {
                //查看三级缓存中是否存在当前beanName单例工厂
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                //存在单例工厂则直接获取该单例对象并放入二级缓存singletonsCurrentlyInCreation中 同时将三级缓存中的工厂对象删除
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```
&emsp;&emsp;可见,首先先尝试从一级缓存 __singletonObjects__ 中获取,如果存在bean实例则直接返回.
若不存在且当前bean正在创建中,则继续尝试从二级缓存 __earlySingletonObjects__ 获取早期创建的bean,存在则直接返回.
若不存在且允许创建早期引用,则继续尝试从三级缓存 __singletonFactories__ 中获取单例工厂,存在工厂则创建早期bean引用,并将该早期引用添加到二级缓存 __earlySingletonObjects__ 中并将三级缓存 __singletonFactories__ 中单例工厂移除.  
3.若第二步获取到实例引用,则尝试从bean实例本身或bean工厂中获取
``` java
/**
 * Get the object for the given bean instance, either the bean
 * instance itself or its created object in case of a FactoryBean.
 * 从给定的bean实例中获取对象,bean实例本身或在FactoryBean的情况下创建bean
 * @param beanInstance the shared bean instance
 * @param name name that may include factory dereference prefix
 * @param beanName the canonical bean name
 * @param mbd the merged bean definition
 * @return the object to expose for the bean
 */
protected Object getObjectForBeanInstance(
        Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

    // Don't let calling code try to dereference the factory if the bean isn't a factory.
    //如果bean不是工厂，不要让调用代码尝试取消对工厂的引用
    //1. 如果bean名称是工厂类型,则设置bean定义为工厂bean并返回
    if (BeanFactoryUtils.isFactoryDereference(name)) {
        if (beanInstance instanceof NullBean) {
            return beanInstance;
        }
        if (!(beanInstance instanceof FactoryBean)) {
            throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
        }
        if (mbd != null) {
            mbd.isFactoryBean = true;
        }
        return beanInstance;
    }

    // Now we have the bean instance, which may be a normal bean or a FactoryBean.
    // If it's a FactoryBean, we use it to create a bean instance, unless the
    // caller actually wants a reference to the factory.
    //现在我们有bean实例,可能是正常的bean或一个工厂bean.
    //如果他是一个工厂bean,我们使用它来创建一个bean实例,除非调用者实际上想要对一个对工厂的引用
    //2. 如果该bean实例不是工厂bean则直接返回
    if (!(beanInstance instanceof FactoryBean)) {
        return beanInstance;
    }

    //3. 若该bean名称不包含工厂前缀& 且是FactoryBean 则尝试从缓存中获取 缓存中没有则尝试从工厂中获取
    Object object = null;
    if (mbd != null) {
        mbd.isFactoryBean = true;
    }
    else {
        object = getCachedObjectForFactoryBean(beanName);
    }
    //如果缓存中没有 尝试从工厂中获取
    if (object == null) {
        // Return bean instance from factory.
        //从工厂中返回bean实例
        FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
        // Caches object obtained from FactoryBean if it is a singleton.
        //如果他是单例的话从工厂bean中获取缓存对象
        if (mbd == null && containsBeanDefinition(beanName)) {
            mbd = getMergedLocalBeanDefinition(beanName);
        }
        boolean synthetic = (mbd != null && mbd.isSynthetic());
        object = getObjectFromFactoryBean(factory, beanName, !synthetic);
    }
    return object;
}
```
&emsp;&emsp;可见这里大约可以分为三步.  
&emsp;&emsp;3.1 如果bean名称是工厂类型,则设置bean定义为工厂bean并返回  
&emsp;&emsp;3.2 如果该bean实例不是工厂bean则直接返回  
&emsp;&emsp;3.3 若该bean名称不包含工厂前缀& 且是FactoryBean 则尝试从缓存中获取 缓存中没有则尝试从工厂中获取  
4.若从三级缓存中没有找到实例,那么就直接执行创建bean的逻辑,就下来就好好看下创建bean的逻辑  
5.首先检查该bean是否正在创建中,在创建中大概率是遇到循环引用了
``` java
/**
 * Return whether the specified prototype bean is currently in creation
 * (within the current thread).
 * 返回指定原型的bean是否正在创建中(在当前线程中)
 * @param beanName the name of the bean
 */
protected boolean isPrototypeCurrentlyInCreation(String beanName) {
    Object curVal = this.prototypesCurrentlyInCreation.get();
    return (curVal != null &&
            (curVal.equals(beanName) || (curVal instanceof Set && ((Set<?>) curVal).contains(beanName))));
}
```
6.检查是否存在父bean工厂,若存在且该bean未创建过,则尝试从父bean工厂中获取
``` java
// Check if bean definition exists in this factory.
//检查bean定义是否存在工厂中
BeanFactory parentBeanFactory = getParentBeanFactory();
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
    // Not found -> check parent.
    //如果在父bean工厂中没有找到当前bean 则检查父工厂
    String nameToLookup = originalBeanName(name);
    if (parentBeanFactory instanceof AbstractBeanFactory) {
        return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                nameToLookup, requiredType, args, typeCheckOnly);
    }
    else if (args != null) {
        // Delegation to parent with explicit args.
        //如果参数不为null,使用显示参数委托给父工厂
        return (T) parentBeanFactory.getBean(nameToLookup, args);
    }
    else if (requiredType != null) {
        // No args -> delegate to standard getBean method.
        //没有参数 则 委托给标准的getBean方法
        return parentBeanFactory.getBean(nameToLookup, requiredType);
    }
    else {
        return (T) parentBeanFactory.getBean(nameToLookup);
    }
}
```
&emsp;&emsp;这里就好理解多了,首先查看是否存在父bean工厂,存在且内存中不包含该bean,则尝试从父bean工厂中获取bean  
7.若不止类型检查,则标记该bean为已创建
``` java
//如果只是类型检查  需要把bean标记为已创建(或将被创建) 即将beanName添加到alreadyCreated中
if (!typeCheckOnly) {
    markBeanAsCreated(beanName);
}

/**
 * Mark the specified bean as already created (or about to be created).
 * 标记指定的bean已经创建(或将被创建)
 * <p>This allows the bean factory to optimize its caching for repeated
 * creation of the specified bean.
 * 这允许bean工厂优化他的缓存以重复创建指定bean
 * @param beanName the name of the bean
 */
protected void markBeanAsCreated(String beanName) {
    //双重检查,防止在检查的过程中当前bean被添加到alreadyCreated中
    if (!this.alreadyCreated.contains(beanName)) {
        synchronized (this.mergedBeanDefinitions) {
            if (!this.alreadyCreated.contains(beanName)) {
                // Let the bean definition get re-merged now that we're actually creating
                // the bean... just in case some of its metadata changed in the meantime.
                //我们实际上正在创建bean,让bean定义重新合并...以防万一一些元数据在此期间发生了变化
                //
                clearMergedBeanDefinition(beanName);
                this.alreadyCreated.add(beanName);
            }
        }
    }
}
```
&emsp;&emsp;此步就是将bean名称添加到alreadyCreated集合中,表明该bean已经创建过了.  
8.再次尝试获取合并bean定义,并检查其定义
``` java
//再次获取合并的bean定义
final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
checkMergedBeanDefinition(mbd, beanName, args);

/**
 * Return a merged RootBeanDefinition, traversing the parent bean definition
 * if the specified bean corresponds to a child bean definition.
 * 返回合并的RootBeanDefinition,如果指定的bean对应于子bean定义,则遍历父bean定义
 * @param beanName the name of the bean to retrieve the merged definition for
 *                 要检索合并定义的bean名称
 * @return a (potentially merged) RootBeanDefinition for the given bean
 * 			给定的bean可能合并的RootBeanDefinition
 * @throws NoSuchBeanDefinitionException if there is no bean with the given name
 * @throws BeanDefinitionStoreException in case of an invalid bean definition
 */
protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
    // Quick check on the concurrent map first, with minimal locking.
    // 首先快速检查并发map,使用最小粒度的锁
    RootBeanDefinition mbd = this.mergedBeanDefinitions.get(beanName);
    //如果map中存在该合并定义 且不需要重新合并定义 则直接返回
    if (mbd != null && !mbd.stale) {
        return mbd;
    }
    return getMergedBeanDefinition(beanName, getBeanDefinition(beanName));
}

/**
 * Check the given merged bean definition,
 * potentially throwing validation exceptions.
 * 检查给定的合并bean定义,潜在的抛出检查异常
 * @param mbd the merged bean definition to check
 * @param beanName the name of the bean
 * @param args the arguments for bean creation, if any
 * @throws BeanDefinitionStoreException in case of validation failure
 */
protected void checkMergedBeanDefinition(RootBeanDefinition mbd, String beanName, @Nullable Object[] args)
        throws BeanDefinitionStoreException {

    if (mbd.isAbstract()) {
        throw new BeanIsAbstractException(beanName);
    }
}
```
&emsp;&emsp;获取合并bean定义的方法前面介绍过了,主要就是根据是否存在合并bean定义,不存在或存在且需要重新合成会重新获取,否则直接返回.
检查就更简单了,检查该bean定义是否是抽象的.  
9.获取该bean依赖属性,存在则先注册这些依赖属性
``` java
// Guarantee initialization of beans that the current bean depends on.
//保证当前bean所依赖的bean的初始化
String[] dependsOn = mbd.getDependsOn();
if (dependsOn != null) {
    for (String dep : dependsOn) {
        if (isDependent(beanName, dep)) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
        }
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

/**
 * Register a dependent bean for the given bean,
 * to be destroyed before the given bean is destroyed.
 * 为给定的bean注册一个依赖的bean,在给定的bean被销毁之前被销毁
 * @param beanName the name of the bean
 * @param dependentBeanName the name of the dependent bean
 */
public void registerDependentBean(String beanName, String dependentBeanName) {
    String canonicalName = canonicalName(beanName);

    synchronized (this.dependentBeanMap) {
        Set<String> dependentBeans =
                this.dependentBeanMap.computeIfAbsent(canonicalName, k -> new LinkedHashSet<>(8));
        if (!dependentBeans.add(dependentBeanName)) {
            return;
        }
    }

    synchronized (this.dependenciesForBeanMap) {
        Set<String> dependenciesForBean =
                this.dependenciesForBeanMap.computeIfAbsent(dependentBeanName, k -> new LinkedHashSet<>(8));
        dependenciesForBean.add(canonicalName);
    }
}
```
&emsp;&emsp;若该bean存在其依赖属性,则先注册其依赖属性,注册则是将其对应关系存入dependentBeanMap和dependenciesForBeanMap中.  
10.这是最重要的一步,根据bean的类型创建bean实例,我们重点关注一下单例的创建
``` java
// Create bean instance.
//创建bean实例
if (mbd.isSingleton()) {
    sharedInstance = getSingleton(beanName, () -> {
        try {
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
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```
&emsp;&emsp;如果是单例bean的话,通过getSingleton方法获取单例bean,可以看到getSingleton的入参是一个函数式接口传入的bean工厂用于创建bean,最后获取到bean继续尝试从bean实例本身获取bean工厂中获取bean.  
&emsp;&emsp;我们接下来先看看getSingleton做了什么:
``` java
/**
 * Return the (raw) singleton object registered under the given name,
 * creating and registering a new one if none registered yet.
 * 返回通过给定名称注册的原始单例对象,如果没有注册的话创建并注册一个新的
 * @param beanName the name of the bean
 * @param singletonFactory the ObjectFactory to lazily create the singleton
 * with, if necessary
 * @return the registered singleton object
 */
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(beanName, "Bean name must not be null");
    synchronized (this.singletonObjects) {
        //1. 判断给定名称是否已经创建过bean对象,存在则直接返回 不存在则创建一个新的
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            //2.前置检查
            //是否处于单例销毁阶段
            if (this.singletonsCurrentlyInDestruction) {
                throw new BeanCreationNotAllowedException(beanName,
                        "Singleton bean creation not allowed while singletons of this factory are in destruction " +
                        "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
            }
            if (logger.isDebugEnabled()) {
                logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
            }
            //判断给定bean名称是否在创建中
            beforeSingletonCreation(beanName);
            boolean newSingleton = false;
            boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
            if (recordSuppressedExceptions) {
                this.suppressedExceptions = new LinkedHashSet<>();
            }
            try {
                //3. 通过工厂创建bean
                //调用org.springframework.beans.factory.support.AbstractBeanFactory.createBean方法
                singletonObject = singletonFactory.getObject();
                //表明是新创建的单例
                newSingleton = true;
            }
            catch (IllegalStateException ex) {
                // Has the singleton object implicitly appeared in the meantime ->
                // if yes, proceed with it since the exception indicates that state.
                // 在此间是否隐式出现了单例对象 ->如果是,则继续处理它,因为异常指示该状态
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    throw ex;
                }
            }
            catch (BeanCreationException ex) {
                if (recordSuppressedExceptions) {
                    for (Exception suppressedException : this.suppressedExceptions) {
                        ex.addRelatedCause(suppressedException);
                    }
                }
                throw ex;
            }
            finally {
                //4. 后置检查
                if (recordSuppressedExceptions) {
                    this.suppressedExceptions = null;
                }
                afterSingletonCreation(beanName);
            }
            //新单例的标志 如果是  则添加到singletonObjects中 并从其他二级缓存中删除
            if (newSingleton) {
                addSingleton(beanName, singletonObject);
            }
        }
        return singletonObject;
    }
```
&emsp;&emsp;可见getSingleton可以大约分为四步:
1. 首先锁住一级缓存singletonObjects并尝试从一级缓存中获取该bean,存在则直接返回,不存在则继续创建
2. 进行前置检查,判断当前bean是否处于销毁中之后再检查该bean是否正在创建中
3. 检查结束后就是通过传入的bean工厂进行创建bean,具体逻辑下面分析,创建完bean后设置newSingleton为true,表明是新创建的bean
4. 最后就是后置检查,将bean从singletonsCurrentlyInCreation中移除,并根据newSingleton状态将bean加入一级缓存singletonObjects和registeredSingletons中并从二三级缓存中移除

&emsp;&emsp;我们最后再来看看这个createBean的逻辑,它就是真正创建bean的地方:
``` java
/**
 * Central method of this class: creates a bean instance,
 * populates the bean instance, applies post-processors, etc.
 * 该类的中心方法:创建一个bean实例,填充bean实例,应用后置处理器
 * @see #doCreateBean
 */
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
    //确保此时真正解析了bean类,并在动态解析类无法存储在共享的合并bean定义中的情况下克隆bean定义
    //1.解析bean类并将其设置到bean定义
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }

    // Prepare method overrides.
    //2. 准备重写的方法
    try {
        mbdToUse.prepareMethodOverrides();
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
                beanName, "Validation of method overrides failed", ex);
    }

    try {
        // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
        //3. 给后置处理器一个机会 返回一个代理而不是目标bean实例
        //执行所有实现InstantiationAwareBeanPostProcessor的后置处理器的初始化前后方法  存在则直接返回bean
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
                "BeanPostProcessor before instantiation of bean failed", ex);
    }

    try {
        //4.真正获取bean的地方
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
&emsp;&emsp;可见createBean中的核心逻辑还是在doCreateBean,这里才是真正创建bean的地方,这里主要做了一些前置准备和检查.大约可以分为以下四步:
1. 解析bean类并将其设置到bean定义中
2. 准备所有重写的方法,存在则设置overloaded标记为false
3. 检查是否存在实现InstantiationAwareBeanPostProcessor的后置处理器,存在则尝试执行其初始化前和初始化后方法,若存在返回的bean则直接返回
4. 前面都检查过了,最后就是真正创建bean的地方
  
&emsp;&emsp;我们继续看看这个doCreateBean是如何创建bean:
``` java
/**
 * Actually create the specified bean. Pre-creation processing has already happened
 * at this point, e.g. checking {@code postProcessBeforeInstantiation} callbacks.
 * 真正创建一个指定bean.预创建处理在此时已经发生了.例如检查postProcessBeforeInstantiation回调
 * <p>Differentiates between default bean instantiation, use of a
 * factory method, and autowiring a constructor.
 * 区分默认bean实例化,工厂方法的使用和自动装配构造函数
 * @param beanName the name of the bean
 * @param mbd the merged bean definition for the bean
 * @param args explicit arguments to use for constructor or factory method invocation
 * @return a new instance of the bean
 * @throws BeanCreationException if the bean could not be created
 * @see #instantiateBean
 * @see #instantiateUsingFactoryMethod
 * @see #autowireConstructor
 */
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
        throws BeanCreationException {

    // Instantiate the bean.
    //1.实例化bean包裹类
    BeanWrapper instanceWrapper = null;
    //如果当前合bean定义是单例的,移除未完成的bean包裹缓存并返回
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    //如果包裹类为空的话 创建bean实例
    if (instanceWrapper == null) {
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    final Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }

    // Allow post-processors to modify the merged bean definition.
    //允许后置处理来修改合并的bean定义
    //2. 执行MergedBeanDefinitionPostProcessor后置处理器逻辑
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                //执行MergedBeanDefinitionPostProcessors的后置处理
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Post-processing of merged bean definition failed", ex);
            }
            //设置处理状态为true
            mbd.postProcessed = true;
        }
    }

    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    //急切的缓存单例,以便即使在由BeanFactoryAware等生命周期接口触发时也可以解决循环引用
    //3. 判断是否需要早期单例暴露,需要则将bean工厂加入三级缓存singletonFactories中
    //mbd.isSingleton()  当前bean定义是否为单例
    //this.allowCircularReferences  是否允许循环引用  默认为true
    //isSingletonCurrentlyInCreation(beanName) 当前单例bean是否在创建中
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isTraceEnabled()) {
            logger.trace("Eagerly caching bean '" + beanName +
                    "' to allow for resolving potential circular references");
        }
        //如果允许暴露早期单例  将该bean的bean工厂添加到三级缓存 singletonFactories中
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // Initialize the bean instance.
    //初始化bean实例
    Object exposedObject = bean;
    try {
        //4. 填充属性值
        populateBean(beanName, mbd, instanceWrapper);
        //5. 初始化bean 调用后置处理器的前置和处置处理方法 和 实现initializingBean的方法以及自定义初始化方法
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

    //6. 是否允许暴露早期引用
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
                            "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
                }
            }
        }
    }

    // Register bean as disposable.
    //7. 将bean注册为一次性
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    }

    return exposedObject;
}
```
&emsp;&emsp;我们一步一步来看下doCreateBean:  
1.首先获取bean包装类,内存中有则直接获取没有则直接创建
``` java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
        throws BeanCreationException {

    // Instantiate the bean.
    //1.实例化bean包裹类
    BeanWrapper instanceWrapper = null;
    //如果当前合bean定义是单例的,移除未完成的bean包裹缓存并返回
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    //如果包裹类为空的话 创建bean实例
    if (instanceWrapper == null) {
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    final Object bean = instanceWrapper.getWrappedInstance();
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }
    ...
}
```
&emsp;&emsp;可见首先先尝试从缓存factoryBeanInstanceCache中获取,如果还是null的话则尝试通过createBeanInstance方法创建,最后设置bean定义的resolvedTargetType.  
&emsp;&emsp;我们继续看下创建bean包裹类的逻辑:
``` java
/**
 * Create a new instance for the specified bean, using an appropriate instantiation strategy:
 * factory method, constructor autowiring, or simple instantiation.
 * 为指定的bean创建一个新实例, 使用一个适当的实例化策略:工厂方法、构造装配或简单的实例化
 * @param beanName the name of the bean
 * @param mbd the bean definition for the bean
 * @param args explicit arguments to use for constructor or factory method invocation
 * @return a BeanWrapper for the new instance
 * @see #obtainFromSupplier
 * @see #instantiateUsingFactoryMethod
 * @see #autowireConstructor
 * @see #instantiateBean
 */
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
    // Make sure bean class is actually resolved at this point.
    //1. 保证此刻bean类已经真正解析过
    Class<?> beanClass = resolveBeanClass(mbd, beanName);

    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
    }

    //2. 如果存在创建这个bean实例回调,从回调中获得beanWrapper
    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
    if (instanceSupplier != null) {
        return obtainFromSupplier(instanceSupplier, beanName);
    }

    //3.如果bean定义存在工厂方法名,尝试使用工厂方法实例化
    if (mbd.getFactoryMethodName() != null) {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    // Shortcut when re-creating the same bean...
    //重新创建一个bean时的快捷方式
    //4. 检查bean是否是工厂或者构造创建 是则继续判断是通过自动装配构造还是默认构造实例化
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
        synchronized (mbd.constructorArgumentLock) {
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
    if (resolved) {
        if (autowireNecessary) {
            return autowireConstructor(beanName, mbd, null, null);
        }
        else {
            return instantiateBean(beanName, mbd);
        }
    }

    // Candidate constructors for autowiring?
    //自动装配的候选参数
    //5. 检查后置处理器中是否有自动装配构造函数,有则尝试从自动装配构造创建bean包裹类
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
            mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    // Preferred constructors for default construction?
    //6. 是否存在默认构造的首选构造函数,是则尝试从自动装配构造函数创建
    ctors = mbd.getPreferredConstructors();
    if (ctors != null) {
        return autowireConstructor(beanName, mbd, ctors, null);
    }

    // No special handling: simply use no-arg constructor.
    //7. 没有特殊处理:简单使用无参构造  通常是走这里
    return instantiateBean(beanName, mbd);
}
```
&emsp;&emsp;可见此步就是根据bean指定的构造方式创建bean包裹类.具体逻辑如下:  
&emsp;&emsp;1.1 前置检查保证此刻bean类已经真正解析过  
&emsp;&emsp;1.2 如果存在创建这个bean实例回调,从回调中获得beanWrapper  
&emsp;&emsp;1.3 如果bean定义存在工厂方法名,尝试使用工厂方法实例化  
&emsp;&emsp;1.4 检查bean是否是工厂或者构造创建 是则继续判断是通过自动装配构造还是默认构造实例化  
&emsp;&emsp;1.5 检查后置处理器中是否有自动装配构造函数,有则尝试从自动装配构造创建bean包裹类  
&emsp;&emsp;1.6 是否存在默认构造的首选构造函数,是则尝试从自动装配构造函数创建  
&emsp;&emsp;1.7 没有特殊处理:简单使用无参构造  通常是走这里  
2.第二步检查postProcessed标记,需要则执行实现MergedBeanDefinitionPostProcessor的后置处理器的postProcessMergedBeanDefinition方法
``` java
// Allow post-processors to modify the merged bean definition.
//允许后置处理来修改合并的bean定义
synchronized (mbd.postProcessingLock) {
    if (!mbd.postProcessed) {
        try {
            //执行MergedBeanDefinitionPostProcessors的后置处理
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
        }
        catch (Throwable ex) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Post-processing of merged bean definition failed", ex);
        }
        //设置处理状态为true
        mbd.postProcessed = true;
    }
}

/**
 * Apply MergedBeanDefinitionPostProcessors to the specified bean definition,
 * invoking their {@code postProcessMergedBeanDefinition} methods.
 * 将MergedBeanDefinitionPostProcessors应用到指定的bean定义,调用他们的postProcessMergedBeanDefinition方法
 * @param mbd the merged bean definition for the bean
 * @param beanType the actual type of the managed bean instance
 * @param beanName the name of the bean
 * @see MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition
 */
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
    //找到所有实现了MergedBeanDefinitionPostProcessor的后置处理器 执行 postProcessMergedBeanDefinition
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
        if (bp instanceof MergedBeanDefinitionPostProcessor) {
            MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
            bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
        }
    }
}
```
&emsp;&emsp;可见就是检查postProcessed是否为false,是则遍历所有实现MergedBeanDefinitionPostProcessor接口的后置处理器,执行postProcessMergedBeanDefinition逻辑,最后再设置postProcessed状态为true,表明已处理过了.  
3.防止bean循环引用的前置检查,将bean工厂添加到三级缓存中
``` java
// Eagerly cache singletons to be able to resolve circular references
// even when triggered by lifecycle interfaces like BeanFactoryAware.
//急切的缓存单例,以便即使在由BeanFactoryAware等生命周期接口触发时也可以解决循环引用
//早期单例暴露标志
//mbd.isSingleton()  当前bean定义是否为单例
//this.allowCircularReferences  是否允许循环引用  默认为true
//isSingletonCurrentlyInCreation(beanName) 当前单例bean是否在创建中
boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
        isSingletonCurrentlyInCreation(beanName));
if (earlySingletonExposure) {
    if (logger.isTraceEnabled()) {
        logger.trace("Eagerly caching bean '" + beanName +
                "' to allow for resolving potential circular references");
    }
    //如果允许暴露早期单例  将该bean的bean工厂添加到三级缓存 singletonFactories中
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
}

/**
 * Add the given singleton factory for building the specified singleton
 * if necessary.
 * 如果需要的话 为创建指定单例添加一个给定的单例工厂
 * <p>To be called for eager registration of singletons, e.g. to be able to
 * resolve circular references.
 * @param beanName the name of the bean
 * @param singletonFactory the factory for the singleton object
 */
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(singletonFactory, "Singleton factory must not be null");
    synchronized (this.singletonObjects) {
        if (!this.singletonObjects.containsKey(beanName)) {
            this.singletonFactories.put(beanName, singletonFactory);
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.add(beanName);
        }
    }
}
```
&emsp;&emsp;检查当前bean是否是单例且允许循环引用且当前bean正在创建中,如果允许暴露早期引用的话,则将创建bean引用的工厂添加到三级缓存singletonFactories中  
4.填充属性值,这也是非常重要的一步,填充属性和后置处理器的逻辑也在这里
``` java
/**
 * Populate the bean instance in the given BeanWrapper with the property values
 * from the bean definition.
 * 使用bean定义中的属性值填充给定BeanWrapper中的bean实例
 * @param beanName the name of the bean
 * @param mbd the bean definition for the bean
 * @param bw the BeanWrapper with bean instance
 */
@SuppressWarnings("deprecation")  // for postProcessPropertyValues
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    //1.前置检查bean包裹类
    if (bw == null) {
        if (mbd.hasPropertyValues()) {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
        }
        else {
            // Skip property population phase for null instance.
            return;
        }
    }

    // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
    // state of the bean before properties are set. This can be used, for example,
    // to support styles of field injection.
    //在设置属性前让任何InstantiationAwareBeanPostProcessors有机会修改bean的状态,.
    //例如这可用于支持字段注入样式
    boolean continueWithPropertyPopulation = true;

    //找到所有实现了InstantiationAwareBeanPostProcessor的后置处理器  执行postProcessAfterInstantiation方法
    //2. 检查bean定义是否不是合成的 且含有实现了InstantiationAwareBeanPostProcessor的后置处理器 有则执行其postProcessAfterInstantiation 根据其返回结果判定是否需要继续填充属性 不需要直接返回
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    continueWithPropertyPopulation = false;
                    break;
                }
            }
        }
    }

    if (!continueWithPropertyPopulation) {
        return;
    }

    //3. 根据bean的属性值和自动装配码 来装配属性
    //获取当前bean定义的属性值
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    //获取解析过的自动装配码  根据类型和名称进行装配
    int resolvedAutowireMode = mbd.getResolvedAutowireMode();
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
        // Add property values based on autowire by name if applicable.
        //如果适用通过名称自动装配新增属性值
        if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }
        // Add property values based on autowire by type if applicable.
        //如果适用通过类型自动装配新增属性值
        if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }
        pvs = newPvs;
    }

    //4.检查是否含有InstantiationAwareBeanPostProcessor后置处理器和是否需要检查依赖属性 需要则调用后置处理器和执行依赖属性检查
    //是否含有InstantiationAwareBeanPostProcessor后置处理器
    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

    PropertyDescriptor[] filteredPds = null;
    if (hasInstAwareBpps) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }
        //遍历所有实现InstantiationAwareBeanPostProcessor的后置处理器 通过postProcessProperties
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
                if (pvsToUse == null) {
                    if (filteredPds == null) {
                        filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                    }
                    pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                    if (pvsToUse == null) {
                        return;
                    }
                }
                pvs = pvsToUse;
            }
        }
    }
    if (needsDepCheck) {
        if (filteredPds == null) {
            filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        }
        checkDependencies(beanName, mbd, filteredPds, pvs);
    }

    //5. 若属性不为空,则将属性应用到bean包裹类中
    //如果存在属性值 则解析相关属性 通常是类中的属性 通过getBean创建相关bean 定义
    if (pvs != null) {
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```
&emsp;&emsp;可见填充属性大约可以分为5步,我们仔细分析下:  
&emsp;&emsp;4.1 首先前置检查bean包裹类没什么好说的  
&emsp;&emsp;4.2 然后检查bean定义是否不是合成的且是否含有实现了InstantiationAwareBeanPostProcessor的后置处理器,值有则执行其postProcessAfterInstantiation,根据其返回结果判定是否需要继续填充属性,不需要则设置continueWithPropertyPopulation为false,下面直接返回  
&emsp;&emsp;4.3 根据bean的属性值和自动装配码,自动装配要么根据类型要么根据名称进行装配  
&emsp;&emsp;4.4 检查是否含有InstantiationAwareBeanPostProcessor后置处理器和是否需要检查依赖属性,需要则调用后置处理器和执行依赖属性检查.找到所有实现了InstantiationAwareBeanPostProcessor的后置处理器执行postProcessProperties逻辑,若返回值pvsToUse为空则执行填充属性值postProcessPropertyValues和过滤依赖检查的属性filterPropertyDescriptorsForDependencyCheck  
&emsp;&emsp;4.5 若填充后的属性pvs不为空,则将这些属性应用到bean包裹类中  
5.bean填充过属性后,就是初始化bean,主要是调用后置处理器的前置和处置处理方法和实现initializingBean的方法以及自定义初始化方法  
``` java
/**
 * Initialize the given bean instance, applying factory callbacks
 * as well as init methods and bean post processors.
 * 初始化给定的bean实例,应用工厂回调以及init方法和bean后置处理器
 * <p>Called from {@link #createBean} for traditionally defined beans,
 * and from {@link #initializeBean} for existing bean instances.
 * 从createBean调用传统意义上的bean,从initializeBean调用现有bean实例。
 * @param beanName the bean name in the factory (for debugging purposes)
 * @param bean the new bean instance we may need to initialize
 * @param mbd the bean definition that the bean was created with
 * (can also be {@code null}, if given an existing bean instance)
 * @return the initialized bean instance (potentially wrapped)
 * @see BeanNameAware
 * @see BeanClassLoaderAware
 * @see BeanFactoryAware
 * @see #applyBeanPostProcessorsBeforeInitialization
 * @see #invokeInitMethods
 * @see #applyBeanPostProcessorsAfterInitialization
 */
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareMethods(beanName, bean);
            return null;
        }, getAccessControlContext());
    }
    else {
        //1. 调用相关Aware的接口
        invokeAwareMethods(beanName, bean);
    }

    //2. 若bean定义为null或bean定义不是合成的 则执行后置处理的前置初始化postProcessBeforeInitialization
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        //调用后置处理器中的初始化之前的方法  postProcessBeforeInitialization
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        //3. 调用初始化方法  先是实现了InitializingBean接口的方法 再上类自定义的初始化方法
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                (mbd != null ? mbd.getResourceDescription() : null),
                beanName, "Invocation of init method failed", ex);
    }
    //4. 若bean定义为null或bean定义不是合成的 则执行后置处理的后置初始化postProcessAfterInitialization
    if (mbd == null || !mbd.isSynthetic()) {
        //调用后置处理器初始化之后的方法	postProcessAfterInitialization
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}
```
&emsp;&emsp;可见第五步就是执行bean相关的初始化接口,主要有4处:  
&emsp;&emsp;5.1首先调用bean实现的相关aware接口
``` java
//调用实现相关Aware接口
//如果实现了BeanNameAware接口  则调用setBeanName将bean名称设置进去
//如果实现了BeanClassLoaderAware接口 则调用setBeanClassLoader将类加载器设置进去
//如果实现了BeanFactoryAware接口 则调用setBeanFactory将bean工厂设置进去
private void invokeAwareMethods(final String beanName, final Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ClassLoader bcl = getBeanClassLoader();
            if (bcl != null) {
                ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
            }
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}
```
&emsp;&emsp;看到代码就很清楚了,就是根据不同的aware接口设置不同的属性到bean中  
&emsp;&emsp;5.2若bean定义为null或bean定义不是合成的 则执行后置处理的前置初始化postProcessBeforeInitialization
``` java
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
        throws BeansException {

    Object result = existingBean;
    for (BeanPostProcessor processor : getBeanPostProcessors()) {
        Object current = processor.postProcessBeforeInitialization(result, beanName);
        if (current == null) {
            return result;
        }
        result = current;
    }
    return result;
}
```
&emsp;&emsp;第二步就是执行所有后置处理器的前置逻辑postProcessBeforeInitialization处理所有bean  
&emsp;&emsp;5.3调用初始化方法,先是实现了InitializingBean接口的方法 再上类自定义的初始化方法
``` java
/**
 * Give a bean a chance to react now all its properties are set,
 * and a chance to know about its owning bean factory (this object).
 * 现在给bean一个反应机会,他的所有属性都已经设置,并有机会了解其拥有的bean工厂
 * This means checking whether the bean implements InitializingBean or defines
 * a custom init method, and invoking the necessary callback(s) if it does.
 * 这意味的检查bean是否实现了InitializingBean或定义一个自定义的初始化方法,且 如果有必要的话调用必要的回调
 * @param beanName the bean name in the factory (for debugging purposes)
 * @param bean the new bean instance we may need to initialize
 * @param mbd the merged bean definition that the bean was created with
 * (can also be {@code null}, if given an existing bean instance)
 * @throws Throwable if thrown by init methods or by the invocation process
 * @see #invokeCustomInitMethod
 */
protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
        throws Throwable {

    //当前bean是否实现了InitializingBean接口 实现了则调用afterPropertiesSet方法
    boolean isInitializingBean = (bean instanceof InitializingBean);
    if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
        if (logger.isTraceEnabled()) {
            logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
        }
        if (System.getSecurityManager() != null) {
            try {
                AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
                    ((InitializingBean) bean).afterPropertiesSet();
                    return null;
                }, getAccessControlContext());
            }
            catch (PrivilegedActionException pae) {
                throw pae.getException();
            }
        }
        else {
            ((InitializingBean) bean).afterPropertiesSet();
        }
    }

    //调用自定义初始化方法
    if (mbd != null && bean.getClass() != NullBean.class) {
        String initMethodName = mbd.getInitMethodName();
        if (StringUtils.hasLength(initMethodName) &&
                !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                !mbd.isExternallyManagedInitMethod(initMethodName)) {
            invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}
```
&emsp;&emsp;第三步先判断bean是否实现了InitializingBean,实现了则执行afterPropertiesSet方法.然后判断bean是否有自定义的init方法,有则执行  
&emsp;&emsp;5.4 然后就是后置逻辑,若bean定义为null或bean定义不是合成的则执行后置处理的后置初始化postProcessAfterInitialization  
&emsp;&emsp;5.5 最后bean就初始化完成返回此bean  
6.检查是否允许暴露早期引用,允许则尝试创建早期引用
``` java
//是否允许暴露早期引用
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
                        "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
            }
        }
    }
}
```
&emsp;&emsp;若允许暴露早期引用的话,则尝试从三级缓存中获取早期引用,获取到则直接返回;没获取到则继续检查依赖的bean.  
7.最后将bean注册为一次性并返回创建的bean
``` java
/**
 * Add the given bean to the list of disposable beans in this factory,
 * registering its DisposableBean interface and/or the given destroy method
 * to be called on factory shutdown (if applicable). Only applies to singletons.
 * 将给定的bean添加到此工厂的一次性bean列表中,注册disposableBean的接口或在工厂关闭时调用给定销毁方法.只应用于单例
 * @param beanName the name of the bean
 * @param bean the bean instance
 * @param mbd the bean definition for the bean
 * @see RootBeanDefinition#isSingleton
 * @see RootBeanDefinition#getDependsOn
 * @see #registerDisposableBean
 * @see #registerDependentBean
 */
protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
    AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);
    if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {
        if (mbd.isSingleton()) {
            // Register a DisposableBean implementation that performs all destruction
            // work for the given bean: DestructionAwareBeanPostProcessors,
            // DisposableBean interface, custom destroy method.
            registerDisposableBean(beanName,
                    new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
        }
        else {
            // A bean with a custom scope...
            Scope scope = this.scopes.get(mbd.getScope());
            if (scope == null) {
                throw new IllegalStateException("No Scope registered for scope name '" + mbd.getScope() + "'");
            }
            scope.registerDestructionCallback(beanName,
                    new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
        }
    }
}
``` 
&emsp;&emsp;若bean不是原型且需要关闭时销毁继续判断bean是否为单例,为单例直接将bean添加到disposableBeans中,不是单例则则注册销毁回调  

11.检查需要的类型是否匹配真正bean实例的类型
``` java
// Check if required type matches the type of the actual bean instance.
//11. 检查需要的类型是否匹配真正bean实例的类型
if (requiredType != null && !requiredType.isInstance(bean)) {
    try {
        T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
        if (convertedBean == null) {
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
        return convertedBean;
    }
    catch (TypeMismatchException ex) {
        if (logger.isTraceEnabled()) {
            logger.trace("Failed to convert bean '" + name + "' to required type '" +
                    ClassUtils.getQualifiedName(requiredType) + "'", ex);
        }
        throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
    }
}
```
&emsp;&emsp;可见这里根据入参传的requiredType判断是否需要将bean装换为requiredType需要的bean类型,并将转换后的类型返回  
12.最后返回创建的bean
- - -



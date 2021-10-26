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
&emsp;&emps;可见大约可以分为12步,具体的流程图可以参见上面的.  
&emsp;&emsp;接下来我们就来一步步看看这个doGetBean具体做了什么:
1. 第一步获取bean的名称
``` java

```

- - -



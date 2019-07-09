---
layout: post
title: "Spring IOC 中的获取 Bean 方法"
date: "2019-05-22"
description: "Spring IOC 中的获取 Bean 方法"
tag: [java, spring]
---

### 回顾 --- 启动一个 Spring Application Context
```
// DefaultListableBeanFactory
public void preInstantiateSingletons() throws BeansException {
    ...
    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {
                ...
                if (isEagerInit) {
                    getBean(beanName);
                }
            }
            else {
                getBean(beanName);
            }
        }
    }
    ...
}
```
在启动一个 Spring Application Context 的过程中, Spring 会实例化配置的 bean 定义, 即 getBean 方法

#### 获取 bean
```
// AbstractBeanFactory#getBean
public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}

protected <T> T doGetBean(
        final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
        throws BeansException {
    /**
     * 将 name 转换为 beanName
     * 1. 如果 name 以 & 开头的 FactoryBean, 则去除 & (重复) 前缀; 如果不以 & 开头的普通 Bean, 则直接返回
     * 2. 若 name 是一个别名, 则解析返回规范名 (支持多重别名: name <- alias1 <- alias2)
     */
    final String beanName = transformedBeanName(name);
    Object bean;

    // 为手动注册的单例 bean, 尽早的检查是否有单例缓存 (并做一些缓存处理, TODO)
    Object sharedInstance = getSingleton(beanName);
    // 如果有缓存单例 bean, 并且 args 不为 null (args 不为 null 意味着创建 bean 而不是获取 bean)
    if (sharedInstance != null && args == null) {
        if (logger.isDebugEnabled()) {
            if (isSingletonCurrentlyInCreation(beanName)) {
                logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                        "' that is not fully initialized yet - a consequence of a circular reference");
            }
            else {
                logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
            }
        }
        /**
         * 当 sharedInstance 不为 FactoryBean, 并且 name 以 & 开头, 则抛出异常
         * 当 sharedInstance 不为 FactoryBean 或者 name 以 & 开头, 则直接返回 (FactoryBean 也仅是一种有特殊功能的 bean)
         * 当 sharedInstance 为 FactoryBean, 并且 name 不以 & 开头, 则使用 FactoryBean 创建 bean 返回
         */
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }
    /**
     * 如上述条件不满足
     * 1. sharedInstance 为 null, 即 beanName 对应 bean 还未创建
     * 2. args 不为 null
     */
    else {
        /**
         * 如果我们已经创建了这个 (Prototype 类型) bean 实例, 则抛出异常: 我们假定有循环依赖的问题
         */
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // 检查这个 bean 定义是否在此 BeanFactory 中
        BeanFactory parentBeanFactory = getParentBeanFactory();
        // 如果父 BeanFactory 不为空, 并且本 BeanFactory 中不包含 beanName 的定义
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // 本 BeanFactory 中没有则查询父 BeanFactory
            String nameToLookup = originalBeanName(name);
            if (args != null) {
                // args 不为 null, 显示使用 args 委托给父 BeanFactory
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else {
                // args 为 null, 委托到标准的 getBean 方法
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
        }

        if (!typeCheckOnly) {
            // 标记 beanName 对应的 bean 已创建 (将 beanName 放在 alreadyCreated 集合中)
            markBeanAsCreated(beanName);
        }

        try {
            // 合并 beanDefinition (TODO)
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);

            // 保证当前 bean 依赖的 beans 先初始化
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    /**
                     * 检查是否存在 depends-on 循环依赖, 如:
                     * <bean id="beanA" class="BeanA" depends-on="beanB">
                     * <bean id="beanB" class="BeanB" depends-on="beanA">
                     *
                     * beanA 要求 beanB 在其之前被创建, beanB 要求 beanA 在其之前被创建
                     * 这样形成一个循环 (支持传递性依赖) , 则直接抛出异常
                     */
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }
                    // 先注册依赖记录 (dep <- beanName)
                    registerDependentBean(dep, beanName);
                    try {
                        // 先初始化依赖项
                        getBean(dep);
                    }
                    catch (NoSuchBeanDefinitionException ex) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                    }
                }
            }

            // 如果是 Singleton 类型
            if (mbd.isSingleton()) {
                /**
                 * 这里并没有直接调用 createBean 方法创建, 而是使用 ObjectFactory
                 * 接口实现作为参数, 在 getSingleton 方法内部会间接调用 createBean 方法
                 * 这样做的原因是为了将单例 bean 缓存以及后续获取
                 */
                sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                    @Override
                    public Object getObject() throws BeansException {
                        try {
                            return createBean(beanName, mbd, args);
                        }
                        catch (BeansException ex) {
                            /**
                             * 从单例缓存中显式移除实例: 因为它有可能为了解决循环引用
                             * 在创建的过程中激进的放入了缓存中
                             * 同时也移除到此 bean 的临时引用
                             */
                            destroySingleton(beanName);
                            throw ex;
                        }
                    }
                });
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }
            // 如果是 Prototype 类型
            else if (mbd.isPrototype()) {
                // 创建一个新实例
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
            // 如果是其他类型, 则委托给指定 Scope 创建
            else {
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }
                try {
                    Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
                        @Override
                        public Object getObject() throws BeansException {
                            beforePrototypeCreation(beanName);
                            try {
                                return createBean(beanName, mbd, args);
                            }
                            finally {
                                afterPrototypeCreation(beanName);
                            }
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

    // 检查要求的类型是否与 bean 实例的实际类型匹配 (这里会进行类型转换)
    if (requiredType != null && bean != null && !requiredType.isInstance(bean)) {
        try {
            return getTypeConverter().convertIfNecessary(bean, requiredType);
        }
        catch (TypeMismatchException ex) {
            if (logger.isDebugEnabled()) {
                logger.debug("Failed to convert bean '" + name + "' to required type '" +
                        ClassUtils.getQualifiedName(requiredType) + "'", ex);
            }
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}
```
`AbstractBeanFactory#getBean` 方法代码执行步骤如下
- 转换 beanName 为规范名
- 根据 beanName 从单例缓存中获取实例
- 若获取实例不为 null 且 args 为 null, 则调用 getObjectForBeanInstance 方法获取 bean
- 若上一步为 false, 则获取父 beanFactory; 父 beanFactory 不为 null 且本 beanFactory 中没有 beanName 对应的 beanDefinition, 则使用父 beanFactory 的 `getBean` 方法获取 bean 后返回
- 若上一步为 false, 获取 beanName 的本 beanFactory 中合并后的 beanDefinition (即 parent 属性)
- 若 beanName 对应的 beanDefinition 有 depends-on 依赖, 则检查是够有依赖循环
- 根据 beanDefinition 的 scope 类型 (如单例, 原型等) 创建实例, 然后再调用 getObjectForBeanInstance 方法获取 bean
- 校验 bean 的类型, 类型转换也在此步中处理
- 返回 bean

流程图如下
![AbstractBeanFactory-getBean.png](https://i.loli.net/2019/05/23/5ce60c26000cc95097.png)

#### 转换 beanName
DefaultSingletonBeanRegistry#singletonObjects 缓存 `<beanName,beanInstance>` 对, 这里需要将 name 转换为 beanName 后再获取 beanInstance; 转换工作有两步
- 去掉 name 的 & 前缀
- 如果 name 是别名将其转换为规范名   
```
// AbstractBeanFactory#transformedBeanName
protected String transformedBeanName(String name) {
    return canonicalName(BeanFactoryUtils.transformedBeanName(name));
}

// BeanFactoryUtils#transformedBeanName
public static String transformedBeanName(String name) {
    Assert.notNull(name, "'name' must not be null");
    String beanName = name;
    // 循环处理 & 前缀, 如: name = "&&&&helloService" => beanName = "helloService"
    while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
        beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
    }
    return beanName;
}

// SimpleAliasRegistry#canonicalName
public String canonicalName(String name) {
    String canonicalName = name;
    String resolvedName;
    /**
     * 循环处理多重别名 (即别名指向别名), 如:
     * <bean id="helloService" class="HelloService"/>
     * <alias name="hello" alias="aliasA"/>
     * <alias name="aliasA" alias="aliasB"/>
     */
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

####　获取单例 bean 实例

```
// DefaultSingletonBeanRegistry
public Object getSingleton(String beanName) {
	return getSingleton(beanName, true);
}

/**
 * allowEarlyReference 参数表示当前 bean 在创建中时是否允许其他 bean 引用
 * 这用于解决循环引用 (注意与循环依赖的区别) 的问题, 如:
 * <bean id="hello" class="Hello">
 *     <property name="world" ref="world"/>
 * </bean>
 * <bean id="world" class="World">
 *     <property name="hello" ref="hello"/>
 * </bean>
 *
 * 如上所示, hello 引用 world, world 引用 hello, 这里形成循环引用
 * Spring 在初始化 hello 时, 由于其引用 world, 转而先去初始化 world,
 * 而 world 又引用 hello, hello 此时处于初始化中; allowEarlyReference 为 true,
 * 即允许 world 引用处于初始化中的 hello, 从而 world 完成初始化, hello 也完成初始化
 *
 * singletonObjects: 缓存初始化完成的 bean
 * earlySingletonObjects: 缓存初始化未完成的 bean
 * singletonFactories: 缓存单例工厂类
 */
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 获取单例 bean, singletonObjects 中缓存了初始化完成的 bean
	Object singletonObject = this.singletonObjects.get(beanName);
    // 如果处于正在初始化中
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		synchronized (this.singletonObjects) {
            // 获取尚未初始化完成的单例 bean, earlySingletonObjects 中缓存了尚未初始化完成的 bean
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
                // 获取对应工厂类 (ObjectFactory)
				ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
				if (singletonFactory != null) {
                    // 将尚未完成初始化的 bean 放入 earlySingletonObjects 中
					singletonObject = singletonFactory.getObject();
					this.earlySingletonObjects.put(beanName, singletonObject);
					this.singletonFactories.remove(beanName);
				}
			}
		}
	}
	return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

#### 合并父 BeanDefinition 和子 BeanDefinition
Spring 支持配置继承, 即在标签中的 `parent` 属性配置父类 bean, 这样子类 bean 可以继承父类 bean 的配置信息
```
// AbstractBeanFactory
protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
	// Quick check on the concurrent map first, with minimal locking.
    // 先在 concurrent map 中快速检查, 这样锁粒度最小
	RootBeanDefinition mbd = this.mergedBeanDefinitions.get(beanName);
	if (mbd != null) {
		return mbd;
	}
	return getMergedBeanDefinition(beanName, getBeanDefinition(beanName));
}

protected RootBeanDefinition getMergedBeanDefinition(String beanName, BeanDefinition bd)
		throws BeanDefinitionStoreException {

	return getMergedBeanDefinition(beanName, bd, null);
}

/**
 * <bean id="hello" class="Hello">
 *     <property name="a" value="a"/>
 *     <property name="b" value="b"/>
 * </bean>
 * <bean id="world" class="World" parent="hello">
 *     <property name="a" value="aa"/>
 * </bean>
 * 合并后等效于:
 * <bean id="hello" class="Hello">
 *     <property name="a" value="a"/>
 *     <property name="b" value="b"/>
 * </bean>
 * <bean id="world" class="World">
 *     <property name="a" value="aa"/>
 *     <property name="b" value="b"/>
 * </bean>
 */
protected RootBeanDefinition getMergedBeanDefinition(
        String beanName, BeanDefinition bd, BeanDefinition containingBd)
        throws BeanDefinitionStoreException {

    synchronized (this.mergedBeanDefinitions) {
        RootBeanDefinition mbd = null;

        // Check with full lock now in order to enforce the same merged instance.
        // 在全锁粒度下检查, 为了确保是同一个合并的实例
        if (containingBd == null) {
            mbd = this.mergedBeanDefinitions.get(beanName);
        }

        if (mbd == null) {
            // 当 bean 定义的 parent 为 null
            if (bd.getParentName() == null) {
                // 拷贝给定的 bean 定义
                if (bd instanceof RootBeanDefinition) {
                    mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();
                }
                else {
                    mbd = new RootBeanDefinition(bd);
                }
            }
            else {
                // 子类 bean 定义: 需要合并 parent
                BeanDefinition pbd;
                try {
                    String parentBeanName = transformedBeanName(bd.getParentName());
                    // 当 beanName 与 parentBeanName 不同, 相同表示不在 parentBeanName 在 parentBeanFactory 中
                    if (!beanName.equals(parentBeanName)) {
                        pbd = getMergedBeanDefinition(parentBeanName);
                    }
                    else {
                        BeanFactory parent = getParentBeanFactory();
                        if (parent instanceof ConfigurableBeanFactory) {
                            pbd = ((ConfigurableBeanFactory) parent).getMergedBeanDefinition(parentBeanName);
                        }
                        else {
                            throw new NoSuchBeanDefinitionException(parentBeanName,
                                    "Parent name '" + parentBeanName + "' is equal to bean name '" + beanName +
                                    "': cannot be resolved without an AbstractBeanFactory parent");
                        }
                    }
                }
                catch (NoSuchBeanDefinitionException ex) {
                    throw new BeanDefinitionStoreException(bd.getResourceDescription(), beanName,
                            "Could not resolve parent bean definition '" + bd.getParentName() + "'", ex);
                }
                // 拷贝父类 bean 定义并使用 子类 bean 定义覆盖
                mbd = new RootBeanDefinition(pbd);
                mbd.overrideFrom(bd);
            }

            // scope 默认设置为 singleton
            if (!StringUtils.hasLength(mbd.getScope())) {
                mbd.setScope(RootBeanDefinition.SCOPE_SINGLETON);
            }

            /**
             * 非单例 bean 中包含的 bean 不能是单例 bean
             * 在这里动态修正, 因为可能是外部 bean 的父子合并的结果,
             * 在这种情况下, 原始的内部 bean 定义不具有继承合并的外部 bean 的单例状态
             */
            if (containingBd != null && !containingBd.isSingleton() && mbd.isSingleton()) {
                mbd.setScope(containingBd.getScope());
            }

            // 暂时缓存合并的 bean 定义 (为了获取元数据的改变, 后续有可能再次合并)
            if (containingBd == null && isCacheBeanMetadata()) {
                this.mergedBeanDefinitions.put(beanName, mbd);
            }
        }

        return mbd;
    }
}
```

#### 获取 bean 实例对象
`createBean` 的过程暂时放下, 来看 `getObjectForBeanInstance` 的过程
```
// AbstractBeanFactory
protected Object getObjectForBeanInstance(
		Object beanInstance, String name, String beanName, RootBeanDefinition mbd) {

    // 如果 name 以 & 开头, 但 beanInstance 不是一个 FactoryBean
	if (BeanFactoryUtils.isFactoryDereference(name) && !(beanInstance instanceof FactoryBean)) {
		throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
	}

	// Now we have the bean instance, which may be a normal bean or a FactoryBean.
	// If it's a FactoryBean, we use it to create a bean instance, unless the
	// caller actually wants a reference to the factory.
    /**
     * bean 实例可能是一个普通 bean 也可能是一个 FactoryBean
     * 如果是一个 FactoryBean, 我们用它来创建 bean 实例,
     * 除非调用者想获取工厂 bean 的引用 (即 name 带 & 前缀)
     */
	if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
		return beanInstance;
	}

	Object object = null;
	if (mbd == null) {
        // FactoryBeanRegistrySupport#factoryBeanObjectCache 缓存了 FactoryBean 创建的 bean
		object = getCachedObjectForFactoryBean(beanName);
	}
	if (object == null) {
        // 获取工厂 bea 实例
		FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
		// 获取 bean 定义
		if (mbd == null && containsBeanDefinition(beanName)) {
			mbd = getMergedLocalBeanDefinition(beanName);
		}
		boolean synthetic = (mbd != null && mbd.isSynthetic());
        // 调用 getObjectFromFactoryBean 方法继续获取实例
		object = getObjectFromFactoryBean(factory, beanName, !synthetic);
	}
	return object;
}

// FactoryBeanRegistrySupport
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
    // factoryBean 创建的是单例 bean 并且 factoryBean 也是单例
	if (factory.isSingleton() && containsSingleton(beanName)) {
		synchronized (getSingletonMutex()) {
            // 从缓存中获取 FactoryBean 创建的实例
			Object object = this.factoryBeanObjectCache.get(beanName);
			if (object == null) {
				object = doGetObjectFromFactoryBean(factory, beanName);
                /**
                 * 在以上的 getObject() 的调用中不会缓存, 只有在 post-process 和 store 才会
                 * (例如: 由于自定义 getBean 的调用触发了循环引用处理)
                 */
				Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
				if (alreadyThere != null) {
					object = alreadyThere;
				}
				else {
					if (object != null && shouldPostProcess) {
						if (isSingletonCurrentlyInCreation(beanName)) {
							// Temporarily return non-post-processed object, not storing it yet..
                            // 如果 beanName 对应的 bean 还在创建中, 则临时返回未 post-processed 的对象, 此时尚未存储
							return object;
						}
						beforeSingletonCreation(beanName);
						try {
							object = postProcessObjectFromFactoryBean(object, beanName);
						}
						catch (Throwable ex) {
							throw new BeanCreationException(beanName,
									"Post-processing of FactoryBean's singleton object failed", ex);
						}
						finally {
							afterSingletonCreation(beanName);
						}
					}
                    // 如果 factoryBean 是单例 bean, 则缓存创建的 bean
					if (containsSingleton(beanName)) {
						this.factoryBeanObjectCache.put(beanName, (object != null ? object : NULL_OBJECT));
					}
				}
			}
			return (object != NULL_OBJECT ? object : null);
		}
	}
	else {
		Object object = doGetObjectFromFactoryBean(factory, beanName);
		if (object != null && shouldPostProcess) {
			try {
				object = postProcessObjectFromFactoryBean(object, beanName);
			}
			catch (Throwable ex) {
				throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
			}
		}
		return object;
	}
}

private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
        throws BeanCreationException {

    Object object;
    try {
        if (System.getSecurityManager() != null) {
            AccessControlContext acc = getAccessControlContext();
            try {
                object = AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
                    @Override
                    public Object run() throws Exception {
                            return factory.getObject();
                        }
                    }, acc);
            }
            catch (PrivilegedActionException pae) {
                throw pae.getException();
            }
        }
        else {
            // 调用工厂方法生成对象
            object = factory.getObject();
        }
    }
    catch (FactoryBeanNotInitializedException ex) {
        throw new BeanCurrentlyInCreationException(beanName, ex.toString());
    }
    catch (Throwable ex) {
        throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
    }

    // 因为 FactoryBean 未初始化完成, 不接受 null 返回: 尽管许多 FactoryBeans 也仅仅返回 null    
    if (object == null && isSingletonCurrentlyInCreation(beanName)) {
        throw new BeanCurrentlyInCreationException(
                beanName, "FactoryBean which is currently in creation returned null from getObject");
    }
    return object;
}
```

#### 小结
本文主要讨论了 `AbstractBeanFactory#getBean` 方法的逻辑, 以及其中调用方法的逻辑; 当这里仍有一些方法未拿出来讨论, 在本文中一起讨论则会导致文章太长难以阅读, 遗留的方法将会在后续文章中讨论
- getSingleton
- createBean


>**参考:**
[Spring IOC 容器源码分析 - 获取单例 bean](http://www.tianxiaobo.com/2018/06/01/Spring-IOC-%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E8%8E%B7%E5%8F%96%E5%8D%95%E4%BE%8B-bean/)

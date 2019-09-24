---
layout: post
title: "Spring IOC 中的创建 Bean 方法"
date: "2019-05-23"
description: "Spring IOC 中的创建 Bean 方法"
tag: [java, spring]
---

### 回顾 --- 获取 bean
```
// AbstractBeanFactory
protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {
...
    // 当声明为单例时
    if (mbd.isSingleton()) {
		sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
			@Override
			public Object getObject() throws BeansException {
				try {
					return createBean(beanName, mbd, args);
				}
				catch (BeansException ex) {
					destroySingleton(beanName);
					throw ex;
				}
			}
		});
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
	}
...
}
```
在获取 bean 的方法中, 当未命中缓存, 会调用相关方法创建并缓存 bean; 以上获取单例 bean 的 getSingleton 方法中, 参数 `singletonFactory` 为匿名实现的 `ObjectFactory` 接口; 此接口匿名实现用于创建 bean, getSingleton 方法中会调用它
#### 获取单例 bean
```
// DefaultSingletonBeanRegistry
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "'beanName' must not be null");
		synchronized (this.singletonObjects) {
        // 根据 beanName 从单例缓存中获取
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null) {
			if (this.singletonsCurrentlyInDestruction) {
				throw new BeanCreationNotAllowedException(beanName,
						"Singleton bean creation not allowed while singletons of this factory are in destruction " +
						"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
			}
            // 创建前检查: 是否为排除创建的 beanName, 并将 beanName 添加到 singletonsCurrentlyInCreation 中
			beforeSingletonCreation(beanName);
			boolean newSingleton = false;
			boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
			if (recordSuppressedExceptions) {
				this.suppressedExceptions = new LinkedHashSet<Exception>();
			}
			try {
                // 调用 ObjectFactory 实现创建 bean
				singletonObject = singletonFactory.getObject();
				newSingleton = true;
			}
			catch (IllegalStateException ex) {
				// Has the singleton object implicitly appeared in the meantime ->
				// if yes, proceed with it since the exception indicates that state.
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
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = null;
				}
                // 创建后检查: 是否为排除创建的 beanName, 并将 beanName 从 singletonsCurrentlyInCreation 中移除
				afterSingletonCreation(beanName);
			}
			if (newSingleton) {
                // 添加单例, 添加或移除相关缓存
				addSingleton(beanName, singletonObject);
			}
		}
		return (singletonObject != NULL_OBJECT ? singletonObject : null);
	}
}
```
getSingleton 方法主要做了一些创建 bean 前后的检查和缓存工作, 创建单例 bean 的主要逻辑在 ObjectFactory 接口的实现类中; 根据之前分析的文章中可知, ObjectFactory#getObject 的实现就是 createBean 方法

#### 创建 Bean
```
// AbstractAutowireCapableBeanFactory
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
	if (logger.isDebugEnabled()) {
		logger.debug("Creating instance of bean '" + beanName + "'");
	}
	RootBeanDefinition mbdToUse = mbd;

    /**
     * 在这里需确保 bean class 已经被加载,
     * 共享合并的 bean 定义没有存储动态加载的 class,
     * 在这种情况下克隆这个 bean 定义并设置
     */
	Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
	if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
		mbdToUse = new RootBeanDefinition(mbd);
		mbdToUse.setBeanClass(resolvedClass);
	}

    // 准备方法覆写: lookup-method 和 replace-method
	try {
		mbdToUse.prepareMethodOverrides();
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
				beanName, "Validation of method overrides failed", ex);
	}

	try {
        // 执行 BeanPostProcessors 以返回一个目标 bean 实例的代理 (AOP 相关, TODO)
		Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
		if (bean != null) {
			return bean;
		}
	}
	catch (Throwable ex) {
		throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
				"BeanPostProcessor before instantiation of bean failed", ex);
	}

    // 创建 bean
	Object beanInstance = doCreateBean(beanName, mbdToUse, args);
	if (logger.isDebugEnabled()) {
		logger.debug("Finished creating instance of bean '" + beanName + "'");
	}
	return beanInstance;
}
```
createBean 方法主要做了 bean class 的动态加载以及方法覆写, 创建 bean 的主要工作在 resolveBeforeInstantiation 方法和 doCreateBean 方法中; resolveBeforeInstantiation 方法与 AOP 相关, 这里暂时不深入讨论

#### 调用 doCreateBean 方法创建 bean
```
// AbstractAutowireCapableBeanFactory
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
        throws BeanCreationException {

    // 实例化 bean, BeanWrapper 是一个包装接口, 提供操作和分析 JavaBean 的功能
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        // factoryBeanInstanceCache 用于缓存未完成的 FactoryBean 实例, 所以这里获取的都是 FactoryBean 的包装
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        /**
         * 创建 bean 实例, 其中包括三种创建 bean 实例的方式
         * 1. 使用工厂方法创建
         * 2. 使用构造方法自动注入
         * 3. 使用无参构造方法
         */
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    // 获取原生 bean 和 bean class
    final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
    Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
    mbd.resolvedTargetType = beanType;

    // 允许 post-processors 修改合并的 bean 定义
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                // 应用 MergedBeanDefinitionPostProcessor 类型接口, 例如自动注入注解, 初始化销毁注解等
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Post-processing of merged bean definition failed", ex);
            }
            mbd.postProcessed = true;
        }
    }

    /**
     * 激进的缓存单例是为了能够解析循环引用, 即使由 BeanFactoryAware 等生命周期接口触发
     *
     * isSingleton: bean 定义声明为单例的
     * allowCircularReferences: beanFactory 允许循环依赖
     * isSingletonCurrentlyInCreation: beanName 对应的 bean 是否正在创建中
     */
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isDebugEnabled()) {
            logger.debug("Eagerly caching bean '" + beanName +
                    "' to allow for resolving potential circular references");
        }
        // ???
        addSingletonFactory(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
                return getEarlyBeanReference(beanName, mbd, bean);
            }
        });
    }

    // 初始化 bean 实例
    Object exposedObject = bean;
    try {
        // 填充 bean
        populateBean(beanName, mbd, instanceWrapper);
        if (exposedObject != null) {
            // 初始化 bean
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
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

    // ???
    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            }
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
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

    // 为 bean 注册销毁
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
`AbstractAutowireCapableBeanFactory#doCreateBean` 方法的执行步骤如下
- 如果 bean 是单例类型, 则先尝试获取 factoryBean 实例的缓存
- 若上一步获取的 BeanWrapper 为 null, 则创建 bean 实例的 BeanWrapper
- 如果 bean 定义允许应用 post-processor, 则使用 MergedBeanDefinitionPostProcessor 实现修改合并的 bean
定义
- 如果允许提前暴露本单例 bean, 则添加对应的单例工厂 (???)
- 如果原生 bean 不为 null, 则填充属性
- 如果原生 bean 不为 null, 则进行初始化
- 如果 earlySingletonExposure 为 true, 则 xxx (TODO)
- 为 bean 注册销毁
- 返回 bean

#### 小结
本文主要讨论了 `DefaultSingletonBeanRegistry#getSingleton` 和 `AbstractAutowireCapableBeanFactory#createBean` 方法; 在 createBean 方法过程中遗留了以下访问未深入讨论
- resolveBeforeInstantiation
- createBeanInstance
- populateBean
- initializeBean

>**参考:**
[Spring IOC 容器源码分析 - 创建单例 bean 的过程](http://www.tianxiaobo.com/2018/06/04/Spring-IOC-%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E5%88%9B%E5%BB%BA%E5%8D%95%E4%BE%8B-bean/)

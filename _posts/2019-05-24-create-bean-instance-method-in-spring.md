---
layout: post
title: "Spring IOC 中的创建 Bean 实例方法"
date: "2019-05-24"
description: "Spring IOC 中的创建 Bean 实例方法"
tag: [java, spring]
---

### 回顾 --- 创建 Bean
```
// AbstractAutowireCapableBeanFactory
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
			throws BeanCreationException {
...
    if (instanceWrapper == null) {
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
...
}
```

#### 创建 bean 实例
```
// AbstractAutowireCapableBeanFactory
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
    // 在这里确保 bean class 已经被加载
	Class<?> beanClass = resolveBeanClass(mbd, beanName);

    // 如果 bean class 不是 public, 并且 bean 定义也不允许访问非 public 方法
	if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName,
				"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
	}

    // 如果工厂方法不为 null, 则使用工厂方法初始化
	if (mbd.getFactoryMethodName() != null) {
		return instantiateUsingFactoryMethod(beanName, mbd, args);
	}

    // 重建相同 bean 的快捷方式
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

    // 暗示构造器自动注入
	Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
	if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
			mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
		return autowireConstructor(beanName, mbd, ctors, args);
	}

    // 没有特殊的处理, 则使用无参构造器
	return instantiateBean(beanName, mbd);
}
```

#### 使用 FactoryMethod 实例化 bean
```
// AbstractAutowireCapableBeanFactory
protected BeanWrapper instantiateUsingFactoryMethod(
        String beanName, RootBeanDefinition mbd, Object[] explicitArgs) {

    return new ConstructorResolver(this).instantiateUsingFactoryMethod(beanName, mbd, explicitArgs);
}

// ConstructorResolver
public BeanWrapper instantiateUsingFactoryMethod(
        final String beanName, final RootBeanDefinition mbd, final Object[] explicitArgs) {
    // 创建 BeanWrapperImpl 实例, 用于包装 bean
    BeanWrapperImpl bw = new BeanWrapperImpl();
    this.beanFactory.initBeanWrapper(bw);

    Object factoryBean;
    Class<?> factoryClass;
    boolean isStatic;

    /**
     * FactoryMethod 分为实例工厂方法和静态工厂方法, 配置如下
     * - 实例工厂方法 (getHello 为非静态方法)
     * <bean id="helloFactory" class="HelloFactory"/>
     * <bean id="hello" factory-bean="helloFactory" factory-method="getHello"/>
     * - 静态工厂方法 (getHello 方法为静态方法)
     * <bean id="hello" class="HelloFactory" factory-method="getHello"/>
     *
     * 注: Spring FactoryBean 的方式在 doGetBean 方法中已经被实例化完成
     */
    String factoryBeanName = mbd.getFactoryBeanName();
    // 如果 factoryBeanName 不为 null, 则必为实例工厂方法
    if (factoryBeanName != null) {
        if (factoryBeanName.equals(beanName)) {
            throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
                    "factory-bean reference points back to the same bean definition");
        }
        factoryBean = this.beanFactory.getBean(factoryBeanName);
        if (factoryBean == null) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "factory-bean '" + factoryBeanName + "' (or a BeanPostProcessor involved) returned null");
        }
        if (mbd.isSingleton() && this.beanFactory.containsSingleton(beanName)) {
            throw new IllegalStateException("About-to-be-created singleton instance implicitly appeared " +
                    "through the creation of the factory bean that its bean definition points to");
        }
        factoryClass = factoryBean.getClass();
        isStatic = false;
    }
    else {
        // 静态工厂方法
        if (!mbd.hasBeanClass()) {
            throw new BeanDefinitionStoreException(mbd.getResourceDescription(), beanName,
                    "bean definition declares neither a bean class nor a factory-bean reference");
        }
        factoryBean = null;
        factoryClass = mbd.getBeanClass();
        isStatic = true;
    }

    Method factoryMethodToUse = null;
    ArgumentsHolder argsHolderToUse = null;
    Object[] argsToUse = null;
    // 解析使用的参数列表
    if (explicitArgs != null) {
        argsToUse = explicitArgs;
    }
    else {
        Object[] argsToResolve = null;
        synchronized (mbd.constructorArgumentLock) {
            factoryMethodToUse = (Method) mbd.resolvedConstructorOrFactoryMethod;
            if (factoryMethodToUse != null && mbd.constructorArgumentsResolved) {
                argsToUse = mbd.resolvedConstructorArguments;
                if (argsToUse == null) {
                    argsToResolve = mbd.preparedConstructorArguments;
                }
            }
        }
        if (argsToResolve != null) {
            argsToUse = resolvePreparedArguments(beanName, mbd, bw, factoryMethodToUse, argsToResolve);
        }
    }
    // FactoryMethod 为 null, 参数列表也为 null
    if (factoryMethodToUse == null || argsToUse == null) {
        // 则需要确定要使用的工厂方法, 尝试匹配给定参数的所有方法名; 获取用户定义的类 (因为可能已经是增强类)
        factoryClass = ClassUtils.getUserClass(factoryClass);

        Method[] rawCandidates = getCandidateMethods(factoryClass, mbd);
        List<Method> candidateSet = new ArrayList<Method>();
        for (Method candidate : rawCandidates) {
            if (Modifier.isStatic(candidate.getModifiers()) == isStatic && mbd.isFactoryMethod(candidate)) {
                candidateSet.add(candidate);
            }
        }
        Method[] candidates = candidateSet.toArray(new Method[candidateSet.size()]);
        // 将候选的方法排序: public 方法优先, 参数多优先
        AutowireUtils.sortFactoryMethods(candidates);

        ConstructorArgumentValues resolvedValues = null;
        boolean autowiring = (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR);
        int minTypeDiffWeight = Integer.MAX_VALUE;
        Set<Method> ambiguousFactoryMethods = null;
        // 计算最少的参数个数
        int minNrOfArgs;
        if (explicitArgs != null) {
            minNrOfArgs = explicitArgs.length;
        }
        else {
            // 因为方法调用没有传递参数, 所以需要在 bean 定义中解析执行构造器的参数
            ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
            resolvedValues = new ConstructorArgumentValues();
            minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
        }

        LinkedList<UnsatisfiedDependencyException> causes = null;
        // 计算匹配的方法
        for (Method candidate : candidates) {
            Class<?>[] paramTypes = candidate.getParameterTypes();

            if (paramTypes.length >= minNrOfArgs) {
                ArgumentsHolder argsHolder;

                if (resolvedValues != null) {
                    // Resolved constructor arguments: type conversion and/or autowiring necessary.
                    try {
                        String[] paramNames = null;
                        ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
                        if (pnd != null) {
                            paramNames = pnd.getParameterNames(candidate);
                        }
                        argsHolder = createArgumentArray(
                                beanName, mbd, resolvedValues, bw, paramTypes, paramNames, candidate, autowiring);
                    }
                    catch (UnsatisfiedDependencyException ex) {
                        if (this.beanFactory.logger.isTraceEnabled()) {
                            this.beanFactory.logger.trace("Ignoring factory method [" + candidate +
                                    "] of bean '" + beanName + "': " + ex);
                        }
                        // Swallow and try next overloaded factory method.
                        if (causes == null) {
                            causes = new LinkedList<UnsatisfiedDependencyException>();
                        }
                        causes.add(ex);
                        continue;
                    }
                }

                else {
                    // Explicit arguments given -> arguments length must match exactly.
                    if (paramTypes.length != explicitArgs.length) {
                        continue;
                    }
                    argsHolder = new ArgumentsHolder(explicitArgs);
                }

                int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
                        argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
                // Choose this factory method if it represents the closest match.
                if (typeDiffWeight < minTypeDiffWeight) {
                    factoryMethodToUse = candidate;
                    argsHolderToUse = argsHolder;
                    argsToUse = argsHolder.arguments;
                    minTypeDiffWeight = typeDiffWeight;
                    ambiguousFactoryMethods = null;
                }
                // Find out about ambiguity: In case of the same type difference weight
                // for methods with the same number of parameters, collect such candidates
                // and eventually raise an ambiguity exception.
                // However, only perform that check in non-lenient constructor resolution mode,
                // and explicitly ignore overridden methods (with the same parameter signature).
                else if (factoryMethodToUse != null && typeDiffWeight == minTypeDiffWeight &&
                        !mbd.isLenientConstructorResolution() &&
                        paramTypes.length == factoryMethodToUse.getParameterTypes().length &&
                        !Arrays.equals(paramTypes, factoryMethodToUse.getParameterTypes())) {
                    if (ambiguousFactoryMethods == null) {
                        ambiguousFactoryMethods = new LinkedHashSet<Method>();
                        ambiguousFactoryMethods.add(factoryMethodToUse);
                    }
                    ambiguousFactoryMethods.add(candidate);
                }
            }
        }

        if (factoryMethodToUse == null) {
            if (causes != null) {
                UnsatisfiedDependencyException ex = causes.removeLast();
                for (Exception cause : causes) {
                    this.beanFactory.onSuppressedException(cause);
                }
                throw ex;
            }
            List<String> argTypes = new ArrayList<String>(minNrOfArgs);
            if (explicitArgs != null) {
                for (Object arg : explicitArgs) {
                    argTypes.add(arg != null ? arg.getClass().getSimpleName() : "null");
                }
            }
            else {
                Set<ValueHolder> valueHolders = new LinkedHashSet<ValueHolder>(resolvedValues.getArgumentCount());
                valueHolders.addAll(resolvedValues.getIndexedArgumentValues().values());
                valueHolders.addAll(resolvedValues.getGenericArgumentValues());
                for (ValueHolder value : valueHolders) {
                    String argType = (value.getType() != null ? ClassUtils.getShortName(value.getType()) :
                            (value.getValue() != null ? value.getValue().getClass().getSimpleName() : "null"));
                    argTypes.add(argType);
                }
            }
            String argDesc = StringUtils.collectionToCommaDelimitedString(argTypes);
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "No matching factory method found: " +
                    (mbd.getFactoryBeanName() != null ?
                        "factory bean '" + mbd.getFactoryBeanName() + "'; " : "") +
                    "factory method '" + mbd.getFactoryMethodName() + "(" + argDesc + ")'. " +
                    "Check that a method with the specified name " +
                    (minNrOfArgs > 0 ? "and arguments " : "") +
                    "exists and that it is " +
                    (isStatic ? "static" : "non-static") + ".");
        }
        else if (void.class == factoryMethodToUse.getReturnType()) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Invalid factory method '" + mbd.getFactoryMethodName() +
                    "': needs to have a non-void return type!");
        }
        else if (ambiguousFactoryMethods != null) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                    "Ambiguous factory method matches found in bean '" + beanName + "' " +
                    "(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
                    ambiguousFactoryMethods);
        }

        if (explicitArgs == null && argsHolderToUse != null) {
            argsHolderToUse.storeCache(mbd, factoryMethodToUse);
        }
    }

    try {
        Object beanInstance;

        if (System.getSecurityManager() != null) {
            final Object fb = factoryBean;
            final Method factoryMethod = factoryMethodToUse;
            final Object[] args = argsToUse;
            beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
                @Override
                public Object run() {
                    return beanFactory.getInstantiationStrategy().instantiate(
                            mbd, beanName, beanFactory, fb, factoryMethod, args);
                }
            }, beanFactory.getAccessControlContext());
        }
        else {
            // 使用实例化策略创建实例, 默认使用 FactoryMethod 反射创建
            beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(
                    mbd, beanName, this.beanFactory, factoryBean, factoryMethodToUse, argsToUse);
        }

        if (beanInstance == null) {
            return null;
        }
        // 设置 bean 到 BeanWrapper 中
        bw.setBeanInstance(beanInstance);
        return bw;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                "Bean instantiation via factory method failed", ex);
    }
}
```
上述方法较长, 但核心是获取 FactoryMethod, 如果 bean 定义中没有配置则做筛选匹配, 最后使用合适的方法构建, 并返回包装类

#### 使用构造方法自动注入创建 bean 实例
```
// ConstructorResolver
public BeanWrapper autowireConstructor(final String beanName, final RootBeanDefinition mbd,
			Constructor<?>[] chosenCtors, final Object[] explicitArgs) {
    ...
}                
```
使用构造方法自动注入创建 bean 实例的方法流程与使用 FactoryMethod 方法创建的流程类似; 此方法在最后创建 bean 实例时默认使用反射, 当 lookup-method 和 replace-method 不为空时则使用 CGLIB 增强 bean 实例

#### 通过默认构造方法创建 bean 实例
```
// AbstractAutowireCapableBeanFactory
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
    try {
        Object beanInstance;
        final BeanFactory parent = this;
        if (System.getSecurityManager() != null) {
            beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
                @Override
                public Object run() {
                    return getInstantiationStrategy().instantiate(mbd, beanName, parent);
                }
            }, getAccessControlContext());
        }
        else {
            // 使用实例化策略进行实例化
            beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
        }
        BeanWrapper bw = new BeanWrapperImpl(beanInstance);
        initBeanWrapper(bw);
        return bw;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
    }
}

// SimpleInstantiationStrategy
public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {
    // 如果没有方法覆写则不使用 CGLIB 增强
	if (bd.getMethodOverrides().isEmpty()) {
		Constructor<?> constructorToUse;
		synchronized (bd.constructorArgumentLock) {
			constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
			if (constructorToUse == null) {
				final Class<?> clazz = bd.getBeanClass();
				if (clazz.isInterface()) {
					throw new BeanInstantiationException(clazz, "Specified class is an interface");
				}
				try {
					if (System.getSecurityManager() != null) {
						constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor<?>>() {
							@Override
							public Constructor<?> run() throws Exception {
								return clazz.getDeclaredConstructor((Class[]) null);
							}
						});
					}
					else {
						constructorToUse =	clazz.getDeclaredConstructor((Class[]) null);
					}
					bd.resolvedConstructorOrFactoryMethod = constructorToUse;
				}
				catch (Throwable ex) {
					throw new BeanInstantiationException(clazz, "No default constructor found", ex);
				}
			}
		}
        // 使用反射实例化
		return BeanUtils.instantiateClass(constructorToUse);
	}
	else {
		// 生成 CGLIB 子类
		return instantiateWithMethodInjection(bd, beanName, owner);
	}
}

// BeanUtils
public static <T> T instantiateClass(Constructor<T> ctor, Object... args) throws BeanInstantiationException {
	Assert.notNull(ctor, "Constructor must not be null");
	try {
		ReflectionUtils.makeAccessible(ctor);
        // 反射创建
		return ctor.newInstance(args);
	}
	catch (InstantiationException ex) {
		throw new BeanInstantiationException(ctor, "Is it an abstract class?", ex);
	}
	catch (IllegalAccessException ex) {
		throw new BeanInstantiationException(ctor, "Is the constructor accessible?", ex);
	}
	catch (IllegalArgumentException ex) {
		throw new BeanInstantiationException(ctor, "Illegal arguments for constructor", ex);
	}
	catch (InvocationTargetException ex) {
		throw new BeanInstantiationException(ctor, "Constructor threw exception", ex.getTargetException());
	}
}
```
使用默认构造器创建 bean 实例过程较为简单, 当 bean 定义中有方法覆写时 (lookup-method 和 replace-method) 使用 CGLIB 增强, 否则使用反射创建 bean 实例即可

>**参考:**
[Spring IOC 容器源码分析 - 创建原始 bean 对象](http://www.tianxiaobo.com/2018/06/06/Spring-IOC-%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E5%88%9B%E5%BB%BA%E5%8E%9F%E5%A7%8B-bean-%E5%AF%B9%E8%B1%A1/)

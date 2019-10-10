---
layout: post
title: "Spring IOC 中的填充 Bean 方法"
date: "2019-05-24"
description: "Spring IOC 中的填充 Bean 方法"
tag: [java, spring]
---

### 回顾 --- 创建 bean 实例
```
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
        throws BeanCreationException {
...
    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
        populateBean(beanName, mbd, instanceWrapper);
        if (exposedObject != null) {
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
    }
...
}
```

#### 填充 bean
```
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    PropertyValues pvs = mbd.getPropertyValues();

    if (bw == null) {
        if (!pvs.isEmpty()) {
            throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
        }
        else {
            // 对于 null 实例, 直接跳过属性填充阶段
            return;
        }
    }

    /**
     * 在 bean 的属性设定之前, 给予 InstantiationAwareBeanPostProcessors 修改 bean 状态
     * 的机会; 例如, 这可以用于支持字段注入的风格 (详细说明建 InstantiationAwareBeanPostProcessor 接口)
     */
    boolean continueWithPropertyPopulation = true;

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

    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
            mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

        // 基于名称自动装配添加属性值
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }

        // 基于类型自动装配添加属性值
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }

        pvs = newPvs;
    }

    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

    if (hasInstAwareBpps || needsDepCheck) {
        PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        // 在获取属性列表 pvs 后, 应用到 bean 之前, 对 pvs 做一些处理 (例如对字段设定特殊值, 或者检查依赖等)
        if (hasInstAwareBpps) {
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                    if (pvs == null) {
                        return;
                    }
                }
            }
        }
        // 依赖检查
        if (needsDepCheck) {
            checkDependencies(beanName, mbd, filteredPds, pvs);
        }
    }

    // 应用属性值
    applyPropertyValues(beanName, mbd, bw, pvs);
}
```
此方法主要做了以下工作
- 获取属性列表 pvs
- 在 bean 的属性填充之前, 应用 InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation 方法
- 使用名称或类型解析依赖的其他 bean, 并添加到属性列表 pvs 中
- 在将 pvs 填充到 bean 之前, 应用 InstantiationAwareBeanPostProcessor#postProcessPropertyValues 方法
- 将属性应用到 bean

#### 按名称自动注入
```
protected void autowireByName(
        String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
    /**
     * 获取不符合的非简单类型属性的属性名, Spring 认为的非简单类型包括
     * 八种基本类型, Enum, CharSequence, Number, Date, URI, Locale 类和子类
     * 以及以上类型的数组形式, 例如 int[] 等
     */
    String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
    for (String propertyName : propertyNames) {
        if (containsBean(propertyName)) {
            // 获取 bean
            Object bean = getBean(propertyName);
            // 添加到 pvs
            pvs.add(propertyName, bean);
            // 注册依赖关系
            registerDependentBean(propertyName, beanName);
            if (logger.isDebugEnabled()) {
                logger.debug("Added autowiring by name from bean name '" + beanName +
                        "' via property '" + propertyName + "' to bean named '" + propertyName + "'");
            }
        }
        else {
            if (logger.isTraceEnabled()) {
                logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
                        "' by name: no matching bean found");
            }
        }
    }
}
```
autowireByName 方法较为简单, 不再过多叙述

#### 按类型自动注入
```
protected void autowireByType(
        String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

    TypeConverter converter = getCustomTypeConverter();
    if (converter == null) {
        converter = bw;
    }

    Set<String> autowiredBeanNames = new LinkedHashSet<String>(4);
    String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
    for (String propertyName : propertyNames) {
        try {
            PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
            /**
             * 不要尝试为 Object 类型自动注入类型: 这样做没有意义,
             * 即使从技术上来说是一个不符合的非简单属性
             */
            if (Object.class != pd.getPropertyType()) {
                MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
                // 当有一个权重 post-processor 时, 不允许为类型匹配激进
                boolean eager = !PriorityOrdered.class.isAssignableFrom(bw.getWrappedClass());
                DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
                // 解析获得自动注入的参数
                Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
                if (autowiredArgument != null) {
                    pvs.add(propertyName, autowiredArgument);
                }
                for (String autowiredBeanName : autowiredBeanNames) {
                    // 注册依赖关系
                    registerDependentBean(autowiredBeanName, beanName);
                    if (logger.isDebugEnabled()) {
                        logger.debug("Autowiring by type from bean name '" + beanName + "' via property '" +
                                propertyName + "' to bean named '" + autowiredBeanName + "'");
                    }
                }
                autowiredBeanNames.clear();
            }
        }
        catch (BeansException ex) {
            throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
        }
    }
}
```
autowireByType 方法中调用的 resolveDependency 方法逻辑较为复杂, 这里暂时知道其返回按类型匹配的 bean 即可

#### 应用属性值
```
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
	if (pvs == null || pvs.isEmpty()) {
		return;
	}

	if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
		((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
	}

	MutablePropertyValues mpvs = null;
	List<PropertyValue> original;

	if (pvs instanceof MutablePropertyValues) {
		mpvs = (MutablePropertyValues) pvs;
        // 如果已经属性值被转换过
		if (mpvs.isConverted()) {
			// Shortcut: use the pre-converted values as-is.
			try {
				bw.setPropertyValues(mpvs);
				return;
			}
			catch (BeansException ex) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Error setting property values", ex);
			}
		}
		original = mpvs.getPropertyValueList();
	}
	else {
		original = Arrays.asList(pvs.getPropertyValues());
	}

    // 获取类型转换器
	TypeConverter converter = getCustomTypeConverter();
	if (converter == null) {
		converter = bw;
	}
	BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

    // 创建一个深拷贝, 以解析值的引用
	List<PropertyValue> deepCopy = new ArrayList<PropertyValue>(original.size());
	boolean resolveNecessary = false;
	for (PropertyValue pv : original) {
        // 如果属性值是转换过的
		if (pv.isConverted()) {
			deepCopy.add(pv);
		}
		else {
			String propertyName = pv.getName();
			Object originalValue = pv.getValue();
            // 当 originalValue 为 RuntimeBeanReference 等需要解析的类型, 返回解析后的值
			Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
			Object convertedValue = resolvedValue;
            // 属性值有对应的 set 方法, 并且属性名不是内嵌或下标属性, 如 prop.a 或 prop[0]
			boolean convertible = bw.isWritableProperty(propertyName) &&
					!PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
			if (convertible) {
                // 转换后的值
				convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
			}
            /**
             * 尽可能的存储合并 bean 定义中已转换的属性值
             * 这是为了避免在每次创建 bean 实例时重新转换
             */
			if (resolvedValue == originalValue) {
                // 如果解析值和原始值相等
				if (convertible) {
					pv.setConvertedValue(convertedValue);
				}
				deepCopy.add(pv);
			}
			else if (convertible && originalValue instanceof TypedStringValue &&
					!((TypedStringValue) originalValue).isDynamic() &&
					!(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
                // originalValue 是 String 类型的值且非动态的, 且 convertedValue 不是集合或数组类型
				pv.setConvertedValue(convertedValue);
				deepCopy.add(pv);
			}
			else {
                // 是否仍需要解析的 flag
				resolveNecessary = true;
				deepCopy.add(new PropertyValue(pv, convertedValue));
			}
		}
	}
	if (mpvs != null && !resolveNecessary) {
		mpvs.setConverted();
	}

    // 属性值类表设置为 deepCopy
	try {
		bw.setPropertyValues(new MutablePropertyValues(deepCopy));
	}
	catch (BeansException ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Error setting property values", ex);
	}
}
```
这里在向 bean 中注入属性之前, 先将属性类表中的原始值做了转换, 最后再调用 bean 的 setter 方法将属性列表中的属性值一个一个设置到 bean 中

#### 小结
本文主要讨论了 `AbstractAutowireCapableBeanFactory#populateBean` 和 `AbstractAutowireCapableBeanFactory#applyPropertyValues` 方法; 在过程中遗留了以下访问未深入讨论
- autowireByType
- resolveValueIfNecessary
- setPropertyValues

>**参考:**
[Spring IOC 容器源码分析 - 填充属性到 bean 原始对象](http://www.tianxiaobo.com/2018/06/11/Spring-IOC-%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E5%A1%AB%E5%85%85%E5%B1%9E%E6%80%A7%E5%88%B0-bean-%E5%8E%9F%E5%A7%8B%E5%AF%B9%E8%B1%A1/)

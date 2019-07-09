---
layout: post
title: "Spring IOC 中的初始化 Bean 方法"
date: "2019-05-24"
description: "Spring IOC 中的初始化 Bean 方法"
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

#### 初始化 bean
```
// AbstractAutowireCapableBeanFactory
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
    // 一些 Aware 接口的回调, 包括 BeanNameAware, BeanClassLoaderAware, BeanFactoryAware
	if (System.getSecurityManager() != null) {
		AccessController.doPrivileged(new PrivilegedAction<Object>() {
			@Override
			public Object run() {
				invokeAwareMethods(beanName, bean);
				return null;
			}
		}, getAccessControlContext());
	}
	else {
		invokeAwareMethods(beanName, bean);
	}

	Object wrappedBean = bean;
	if (mbd == null || !mbd.isSynthetic()) {
        // 应用 BeanPostProcessor 接口的 postProcessBeforeInitialization 方法
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}

	try {
        // 调用初始化方法
		invokeInitMethods(beanName, wrappedBean, mbd);
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				(mbd != null ? mbd.getResourceDescription() : null),
				beanName, "Invocation of init method failed", ex);
	}
	if (mbd == null || !mbd.isSynthetic()) {
        // // 应用 BeanPostProcessor 接口的 postProcessAfterInitialization 方法
		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
	}
	return wrappedBean;
}
```
initializeBean 方法的逻辑较为简单, 主要回调 bean 实现的一些接口方法
- 回调 Aware 接口的方法
- 回调 BeanPostProcessor 接口的 postProcessBeforeInitialization 方法
- 回调初始化方法 (包括 InitializingBean 接口的 afterPropertiesSet 方法和 init-method 配置的方法)
- 回调 BeanPostProcessor 接口的 postProcessAfterInitialization 方法

#### 调用初始化方法
```
protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd)
        throws Throwable {
    // 判定 bean 是否实现了 InitializingBean 接口
    boolean isInitializingBean = (bean instanceof InitializingBean);
    // 实现了 InitializingBean 接口, 且 afterPropertiesSet 方法不是外部管理的初始化方法
    if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
        if (logger.isDebugEnabled()) {
            logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
        }
        if (System.getSecurityManager() != null) {
            try {
                AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
                    @Override
                    public Object run() throws Exception {
                        ((InitializingBean) bean).afterPropertiesSet();
                        return null;
                    }
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

    if (mbd != null) {
        // 获取 init-method 配置的初始化方法名
        String initMethodName = mbd.getInitMethodName();
        // initMethodName 不为空, 当 bean 实现了 InitializingBean 接口时, initMethodName 不能为 "afterPropertiesSet"
        if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                !mbd.isExternallyManagedInitMethod(initMethodName)) {
            invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}
```
初始化方法主要调用了 InitializingBean 接口的 afterPropertiesSet 方法和 init-method 配置的方法名, 当两个方法重复时, init-method 配置无效

#### 小结
初始化 bean 的过程较为简单, 主要调用一些 Spring  bean 的生命周期方法; 对于初始化方法, 除了 InitializingBean 接口的 afterPropertiesSet 方法和 init-method 配置的方法外, 还可以使用 `@PostConstruct` 注解标注, 此注解处理在 `InitDestroyAnnotationBeanPostProcessor` 中; 当联合使用三种方式声明相同的初始化方法只会调用一次, 当联合使用三种方式声明不同的初始化方法会调用三次, 执行顺序如下
- `@PostConstruct` 注解的方法
- InitializingBean 接口的 afterPropertiesSet 方法
- init-method 配置的方法

>**参考:**  
[Spring IOC 容器源码分析 - 余下的初始化工作
](http://www.tianxiaobo.com/2018/06/11/Spring-IOC-%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-%E4%BD%99%E4%B8%8B%E7%9A%84%E5%88%9D%E5%A7%8B%E5%8C%96%E5%B7%A5%E4%BD%9C/)  
[Combining lifecycle mechanisms](https://docs.spring.io/spring/docs/4.3.24.RELEASE/spring-framework-reference/html/beans.html#beans-factory-lifecycle-combined-effects)

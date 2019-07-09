---
layout: post
title: "BeanFactoryPostProcessor 和 BeanPostProcessor 的区别"
date: "2019-05-16"
description: "BeanFactoryPostProcessor 和 BeanPostProcessor 的区别"
tag: [java, spring]
---

### BeanFactoryPostProcessor
```
package org.springframework.beans.factory.config;

import org.springframework.beans.BeansException;

/**
 * 允许对一个应用上下文的 bean 定义自定义修改,
 * 调整上下文的底层 bean 工厂中的 bean 的属性值
 *
 * 应用上下文可以自动探测在 bean 定义中的 BeanFactoryPostProcessor beans,
 * 并且在任何其他 bean 创建之前应用它们
 *
 * 对于系统管理员用于覆盖应用上下文中的 bean 属性配置的
 * 自定义配置文件很有用
 *
 * 处理这样配置需求的开箱解决方法见
 * PropertyResourceConfigurer 和它的具体实现
 *
 * BeanFactoryPostProcessor 可以作用和修改 bean 定义,
 * 而不是 bean 实例. 这样做可能会引起提前 bean 实例化,
 * 违反容器以及导致意外的副作用.
 * 如果需要作用域 bean 实例, 考虑实现
 * BeanPostProcessor
 *
 * @author Juergen Hoeller
 * @since 06.07.2003
 * @see BeanPostProcessor
 * @see PropertyResourceConfigurer
 */
public interface BeanFactoryPostProcessor {

	/**
     * 在应用上下文内部 bean 工厂的标准初始化后修改它.
     * 所有的 bean 定义都已经被加载, 但是还没有
     * bean 被创建. 这允许覆盖和添加属性
     * 甚至激进初始化 bean
	 * @param beanFactory 应用上下文使用的 bean 工厂
	 * @throws org.springframework.beans.BeansException 如果有错误
	 */
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}

```
在 Spring Bean 生命周期中的
```
... => BeanFactoryPostProcessor => ... => 实例化 bean =>
```

### BeanPostProcessor
```
package org.springframework.beans.factory.config;

import org.springframework.beans.BeansException;

/**
 * 允许对新的 bean 实例自定义修改的工程 hook,
 * 例如: 检查标记接口或者使用代理包装.
 *
 * ApplicationContexts 可以在 bean 定义中自动探测到 BeanPostProcessor beans,
 * 并且将其应用于后续创建的任何 beans.
 * 普通的 bean 工厂允许程序性注册 post-processors,
 * 并通过这个工厂应用于所有 bean 的创建
 *
 * 通常, post-processors 通过标记接口填充 beans
 * 将会实现 postProcessBeforeInitialization 方法
 * 而 post-processors 使用代理包装 beans 通常
 * 实现 postProcessAfterInitialization 方法
 *
 * @author Juergen Hoeller
 * @since 10.10.2003
 * @see InstantiationAwareBeanPostProcessor
 * @see DestructionAwareBeanPostProcessor
 * @see ConfigurableBeanFactory#addBeanPostProcessor
 * @see BeanFactoryPostProcessor
 */
public interface BeanPostProcessor {

    /**
     * 应用这个 BeanPostProcessor 到给定的新 bean 实例在
     * bean 的任何初始化回调之前 (如 InitializingBean 的 afterPropertiesSet 方法
     * 或者自定义的 init-method 方法). bean 将会使用属性值填充.
     * 返回的 bean 实例可能是原始 bean 的一个包装 bean.
	 * @param bean 新 bean 实例
	 * @param beanName bean 的名称
	 * @return 使用的 bean 实例, 可能是原始 bean 也可能是一个 包装 bean;
     * 如果为 null, 不会调用后续的 BeanPostProcessors
	 * @throws org.springframework.beans.BeansException 如果有错误
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 */
	Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

    /**
     * 应用这个 BeanPostProcessor 到给定的新 bean 实例在
     * bean 的任何初始化回调之前 (如 InitializingBean 的 afterPropertiesSet 方法
     * 或者自定义的 init-method 方法). bean 将会使用属性值填充.
     * 返回的 bean 实例可能是原始 bean 的一个包装 bean.
     * 如果是 FactoryBean, 这个回调会对 FactoryBean 实例
     * 以及 FactoryBean 创建的对象调用 (从 Spring 2.0).
     * post-processor 通过相应的检查 (bean instanceof FactoryBean) 可以决定
     * 应用于 FactoryBean 还是 FactoryBean 创建的实例, 或者都应用
     * 这个回调在 InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation 方法
     * 触发短路之后也会被调用,
     * 与其他 BeanPostProcessor 回调相比
	 * @param bean 新 bean 实例
	 * @param beanName bean 的名称
     * @return 使用的 bean 实例, 可能是原始 bean 也可能是一个 包装 bean;
     * 如果为 null, 不会调用后续的 BeanPostProcessors
	 * @throws org.springframework.beans.BeansException 如果有错误
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 * @see org.springframework.beans.factory.FactoryBean
	 */
	Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```
在 Spring Bean 生命周期中的
```
... => 实例化 bean => ... => BeanPostProcessor 的前置处理 =>
... => afterPropertiesSet => init-method => ... => BeanPostProcessor 的后置处理
```

>**参考:**
[BeanFactoryPostProcessor和BeanPostProcessor的区别](https://www.cnblogs.com/duanxz/p/3750725.html)

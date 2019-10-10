

### 启动一个 Spring Application Context
```
public class App {
    public static void main(String[] args) {
        new ClassPathXmlApplicationContext("application-context.xml" );
    }
}

public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
    this(new String[] {configLocation}, true, null);
}


public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
        throws BeansException {

    super(parent);
    setConfigLocations(configLocations);
    if (refresh) {
        // 刷新应用上下文
        refresh();
    }
}
```

#### 刷新应用上下文
```
// AbstractApplicationContext
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
        // 为刷新上下文做准备
		prepareRefresh();

        // 告诉子类刷新内部的 bean factory
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 准备在本上下文中使用的 bean factory
		prepareBeanFactory(beanFactory);

		try {
            // 在上下文子类中允许 bean factory 的 post-processor
			postProcessBeanFactory(beanFactory);

            // 调用在上下文中注册为 bean 的 factory processors
			invokeBeanFactoryPostProcessors(beanFactory);

            // 注册拦截 bean 创建的 bean processors
			registerBeanPostProcessors(beanFactory);

            // 为上下文初始化 message source
			initMessageSource();

            // 为上下文初始化 event multicaster
			initApplicationEventMulticaster();

            // 在特殊的上下文子类中初始化特殊的 beans
			onRefresh();

            // 检查 listener bean 并注册
			registerListeners();

            // 初始化所有剩余的单例 bean (非懒加载的)
			finishBeanFactoryInitialization(beanFactory);

            // 最后: 发布对应的事件
			finishRefresh();
		}

		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}

            // 销毁已经创建的单例, 避免占用资源
			destroyBeans();

            // 重置 'active' 标识
			cancelRefresh(ex);

            // 抛出异常给调用者
			throw ex;
		}

		finally {
            // 重置 Spirng core 内建的公共缓存, 因为我们不再需要单例 beans 的元数据
			resetCommonCaches();
		}
	}
}
```
refresh 方法做了一系列初始化 Application Context 和 Bean Factory 的的动作, 最后开始初始化 Bean Factory 中定义的 bean

#### 完成 Bean Factory 的初始化
```
// AbstractApplicationContext
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // 为本上下文初始化 conversion 服务
	if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
			beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
		beanFactory.setConversionService(
				beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
	}

    /**
     * 如果之前没有任何 bean post-processor (例如 PropertyPlaceholderConfigurer) 注册,
     * 则注册一个默认的内嵌值解析器: 在这里, 主要解析注解属性的值
     */
	if (!beanFactory.hasEmbeddedValueResolver()) {
		beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
			@Override
			public String resolveStringValue(String strVal) {
				return getEnvironment().resolvePlaceholders(strVal);
			}
		});
	}

    // 尽早初始化 LoadTimeWeaverAware bean, 以便尽早注册它们的转换器
	String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
	for (String weaverAwareName : weaverAwareNames) {
		getBean(weaverAwareName);
	}

    // 停止为类型匹配使用临时的 ClassLoader
	beanFactory.setTempClassLoader(null);

    // 允许缓存所有 bean 定义的元数据, 并不期望有进一步的更改
	beanFactory.freezeConfiguration();

    // 实例化所有剩余的单例 bean (非懒加载的)
	beanFactory.preInstantiateSingletons();
}
```

#### 预实例化单例
```
// DefaultListableBeanFactory
public void preInstantiateSingletons() throws BeansException {
	if (logger.isDebugEnabled()) {
		logger.debug("Pre-instantiating singletons in " + this);
	}

    /**
     * 迭代一个副本, 已允许 init methods 中注册新的 bean 定义
     * 虽然这不是常规工厂启动的一部分, 但是这确实可以工作
     */
	List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

    // 触发所有非拦截在 beans 的初始化
	for (String beanName : beanNames) {
		RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
		if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
			if (isFactoryBean(beanName)) {
				final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
				boolean isEagerInit;
				if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
					isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
						@Override
						public Boolean run() {
							return ((SmartFactoryBean<?>) factory).isEagerInit();
						}
					}, getAccessControlContext());
				}
				else {
					isEagerInit = (factory instanceof SmartFactoryBean &&
							((SmartFactoryBean<?>) factory).isEagerInit());
				}
				if (isEagerInit) {
					getBean(beanName);
				}
			}
			else {
				getBean(beanName);
			}
		}
	}

    // 对适用的 beans 触发 post-initialization 回调
	for (String beanName : beanNames) {
		Object singletonInstance = getSingleton(beanName);
		if (singletonInstance instanceof SmartInitializingSingleton) {
			final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
			if (System.getSecurityManager() != null) {
				AccessController.doPrivileged(new PrivilegedAction<Object>() {
					@Override
					public Object run() {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}
				}, getAccessControlContext());
			}
			else {
				smartSingleton.afterSingletonsInstantiated();
			}
		}
	}
}
```
preInstantiateSingletons 方法逻辑较为简单, 循环所有的单例 bean 名称, 依次实例化 (getBean 方法在缓存中未获取到会进行创建), 最后为特定的单例 bean 进行实例化后的回调

#### 小结
本文主要叙述了新建一个 Application Context, Spring 主要做了那些工作; 只叙述了调用的主要流程, 没有深入具体方法的实现; 在此记录下后续需要深入的方法
- prepareRefresh()
- obtainFreshBeanFactory()
- prepareBeanFactory()
- postProcessBeanFactory()
- invokeBeanFactoryPostProcessors()
- registerBeanPostProcessors()
- initMessageSource()
- initApplicationEventMulticaster()
- registerListeners()
- finishRefresh()
- freezeConfiguration()
- getBean()

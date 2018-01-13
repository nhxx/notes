---
title: spring源码解析03：高级容器
date: 2018-01-03 22:19:10
tags:
---

## 类图

先抛下高级容器大家族的整体类图。从接口继承关系看，ApplicationContext是高级容器的核心超类，往下分成两大类：web容器和非web容器；从实现类的继承关系看，跟接口一样，只不过非web容器根据资源的类型分为xml文件、Annotation两个分支；下文以ClassPathXmlApplicationContext实现为例，从源码层面分析容器的内部实现原理；
![高级容器类图](/images/Context.png)

## ApplicationContext Interfaces

### 1. ApplicationContext

作为高级容器的顶层基类接口，在BeanFactory基础上又扩展了多个接口，增强了多个能力：

> * BeanFactory: 初级容器
> * MessageSource: 消息国际化
> * ApplicationEventPublisher: 发布事件
> * EnvironmentCapable: 可感知环境（比如测试环境、线上环境等）
> * ResourcePatternResolver: 可解析资源

### 2. ConfigurableApplicationContext

在ApplicationContext之上提供了，提供了控制容器生命周期及功能配置的能力。

> * ApplicationContext: 见1
> * Lifecycle: 启停
> * Closeable: 可关闭

## ApplicationContext Implements
	
> * AbstractApplicationContext: 实现了绝大部分上下文容器的全部功能，refresh模板方式得初始化容器，内部组合了较多其他功能的实现类, 包含ResourcePatternResolver, ConfigurableEnvironment, List<BeanFactoryPostProcessor, LifecycleProcessor, MessageSource, ApplicationEventMulticaster, Set<ApplicationListener, ApplicationEvent
> * AbstractRefreshableApplicationContext：实现了refresh流程中，refresh bean容器的逻辑；包含创建DefaultListableBeanFactory，定制行为（allowBeanDefinitionOverriding，allowCircularReferences），扩展了加载bean配置逻辑给子类
> * AbstractRefreshableConfigApplicationContext: 提供配置文件的处理；refresh触发逻辑；
> * AbstractXmlApplicationContext: 封装加载（bean）xml配置文件到容器（通过XmlBeanDefinitionReader）
> * ClassPathXmlApplicationContext: 暴露构造器方法来初始化上下文容器（使用入口）

## 初始化

Context通过构造器方法进行初始化，传入configLocations表示资源文件信息，refresh表示是否重置容器配置，parent表示父上下文容器。
构造器构造过程包含三步：

> 1. 配置父Context
> 2. 保存资源文件属性
> 3. 重置上下文环境（默认情况都是需要重置，也可以订制化）

```
	public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
			throws BeansException {
		// 设置父容器，初始化父类的默认属性（ResourcePatternResolver，Environment等）
		super(parent);
		// 保存（校验）资源信息
		setConfigLocations(configLocations);
		if (refresh) {
			// 核心的初始化Context过程
			refresh();
		}
	}
```

重点分析下refresh的过程，典型的模板模式，源码如下；

```java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// 做一些前期准备工作
			prepareRefresh();

			// 重建beanFactory，解析BeanDefinition信息
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 配置容器的一些标准属性（ClassLoader，PostProcessor等）
			prepareBeanFactory(beanFactory);

			try {
				// 供子类扩展用：
				//  在bean容器初始化后（bean definition都加载到容器，但是都未实际初始化前），允许对beanFactory做些特殊处理
				postProcessBeanFactory(beanFactory);

				// 调用注册在上下文中的BeanFactoryPostProcessor（容器后置处理器）
				invokeBeanFactoryPostProcessors(beanFactory);

				// 注册BeanPostProcessor：遍历所有后置处理器，处理下顺序，并初始化加载进容器
				registerBeanPostProcessors(beanFactory);

				// 初始化MessageSource
				initMessageSource();

				// 初始化事件广播：SimpleApplicationEventMulticaster
				initApplicationEventMulticaster();

				// 扩展给子类用
				onRefresh();

				// 校验并注册ApplicationListener
				registerListeners();

				// 初始化所有的非懒加载单例
				finishBeanFactoryInitialization(beanFactory);

				// 广播refresh结束的事件
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```




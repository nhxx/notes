---
title: spring源码解析02：初级容器
date: 2018-01-03 21:59:39
tags:
---

## 类图

![beanfactory.png](/images/beanfactory.png)

## BeanFactory Interfaces

先看左边接口的继承树结构：
> * BeanFactory：容器的根基类，定义了获取Bean，判断Bean（是否存在，是否单例，是否Prototype，是否Match Class Type）等函数
> * ListableBeanFactory：扩展了获取bean的多种形式，如获取所有bean，根据Annotation获取对应Beans等
> * HierarchicalBeanFactory：扩展增强了上下级联系的容器
> * AutowireCapableBeanFactory：扩展了能自动装配的能力
> * ConfigurableBeanFactory：扩展了配置bean容器的能力，比如设置父类容器，设置ClassLoader，BeanExpression解析，Bean 
Property转化器（作为PropertyEditors的替换），设置BeanPostProcessor，Bean的销毁处理等；

右面接口的继承树关系：
> * AliasRegistry：管理Bean别名
> * BeanDefinitionRegistry：可通过BeanDefinition进行注册的容器；

## BeanFactory Implementations

类图中间是容器的实现类的结构，大致看下几个核心类的实现功能和扩展功能点：
> * DefaultSingletonBeanRegistry：默认的单例bean注册中心；
> * FactoryBeanRegistrySupport：作为AbstractBeanFactory的基类，增加了factoryBean的本地存储，并提供了从FactoryBean中获取Bean的方法；
> * AbstractBeanFactory：实际意义上第一个实现大部分容器功能的抽象基类，该类提供了单例缓存（基于DefaultSingletonBeanRegistry），区分单例/原型的判断方式，FactoryBean的处理，别名（Alias），BeanDefinition的merge（类的单继承），Bean的销毁。提供了ConfigurableBeanFactory的全部功能，并提供了getBeanDefinition，createBean，containsBeanDefinition供子类实现扩展（根据BeanName查询BeanDefinition，根据BeanDefinition创建Bean过程）；
> * AbstractAutowireCapableBeanFactory：扩展了AbstractBeanFactory的createBean，提供了bean创建，属性配置，自动装配，初始化等能力。自动装配的方式：通过构造器，根据类型设置属性，根据Name设置属性；并提供了resolveDependency的模板方法，供子类扩展。
> * DefaultListableBeanFactory：ListableBeanFactory、BeanDefinitionRegistry的默认实现类，成熟的基于Bean definition对象的Bean容器。典型的使用方式：在使用bean之前，先将所有的bean definitions注册到容器中。这样在容器中查找一个bean定义会变得非常方便，因为查找过程就是方便地查询本地bean定义map。DefaultListableBeanFactory可以单独使用，也可以作为自定义容器的超类。要注意，特殊的bean definitions形式的识别通常是单独识别的，而不是通过继承容器。比如：PropertiesBeanDefinitionReader或者XmlBeanDefinitionReader；
> * XmlBeanFactory：注册Bean的入口较多，1通过DefaultListableFactory的registerBeanDefinition注册，2通过XmlBeanFactory，通过内部的XmlBeanDefinitionReader读取xml并解析注册到DefaultListableFactory中（也是通过registerBeanDefinition）。


### DefaultListableBeanFactory 初始化


### DefaultListableBeanFactory 注册BeanDefinition
```
    @Override
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		//STEP 1: 校验
		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
				// 校验beanDefinition的合法性：
				// 1 如果有bean工厂方法，则不能使用lookup-method,replaced-method（统称为method override）
				// 2 如果有用method override，则校验下标签内的方法名（name）是否真实存在
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}

		BeanDefinition oldBeanDefinition;

		// 注册逻辑，从BeanDefinition本地缓存中获取定义，
		//  1. 如果已有，判断是否能覆盖，可以就覆盖，不行就报错退出
		//  2. 如果没有，增加beanDefinitionMap，beanDefinitionNames属性，从manualSingletonNames删除beanName
		oldBeanDefinition = this.beanDefinitionMap.get(beanName);
		if (oldBeanDefinition != null) {
			if (!isAllowBeanDefinitionOverriding()) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
						"': There is already [" + oldBeanDefinition + "] bound.");
			}
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		else {
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<String>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					if (this.manualSingletonNames.contains(beanName)) {
						Set<String> updatedSingletons = new LinkedHashSet<String>(this.manualSingletonNames);
						updatedSingletons.remove(beanName);
						this.manualSingletonNames = updatedSingletons;
					}
				}
			}
			else {
				// Still in startup registration phase
				this.beanDefinitionMap.put(beanName, beanDefinition);
				this.beanDefinitionNames.add(beanName);
				this.manualSingletonNames.remove(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		
		// 如果是覆盖旧定义，或者单例对象的缓存内已经存在该beanName，则重置该beanName；
		// 重置逻辑：
		//  删除该bean的mergedBeanDefinition缓存；
		//  删除该bean的“各种”单例实例，删除基于class type的缓存
		//  遍历删除所有该Bean的子类实例
		if (oldBeanDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
	}
```
注册的逻辑比较简单，只将通过校验的BeanDefinition加到本地Bean定义缓存（beanDefinitionMap，beanDefinitionNames）内接口；
TIP：值得一提的是，没有对BeanDefinition进行严格的校验。

### DefaultListableBeanFactory 加载Bean过程

![bean加载流程](/images/beanFactoryFlow.jpg)

BeanFactory基类容器提供了多种重载的获取bean能力，概况起来就两种，by BeanName、by ClassType；其中by ClassType较复杂，因为底层的容器实现（AbstractBeanFactory）都基于BeanName进行，所以在DefaultListableBeanFactory中，提供了byClass的能力。下面就是提供该能力的核心函数。

```java
@SuppressWarnings("unchecked")
	private <T> NamedBeanHolder<T> resolveNamedBean(Class<T> requiredType, Object... args) throws BeansException {
		Assert.notNull(requiredType, "Required type must not be null");
		// Step 1:先根据class转成所有能匹配到的names
		String[] candidateNames = getBeanNamesForType(requiredType);

		if (candidateNames.length > 1) {
			List<String> autowireCandidates = new ArrayList<String>(candidateNames.length);
			for (String beanName : candidateNames) {
				if (!containsBeanDefinition(beanName) || getBeanDefinition(beanName).isAutowireCandidate()) {
					autowireCandidates.add(beanName);
				}
			}
			if (!autowireCandidates.isEmpty()) {
				candidateNames = autowireCandidates.toArray(new String[autowireCandidates.size()]);
			}
		}

		// Step 2: 如果只匹配到单个，那就根据name和type去getBean
		if (candidateNames.length == 1) {
			String beanName = candidateNames[0];
			return new NamedBeanHolder<T>(beanName, getBean(beanName, requiredType, args));
		}
		// Step 3: 如果匹配到多个，比如dateSource。两层过滤
		//  1. determinePrimaryCandidate：判断是否带primary标签
		//  2. determineHighestPriorityCandidate：基于@Primary的配置值（值越低优先级越大）
		else if (candidateNames.length > 1) {
			Map<String, Object> candidates = new LinkedHashMap<String, Object>(candidateNames.length);
			for (String beanName : candidateNames) {
				if (containsSingleton(beanName)) {
					candidates.put(beanName, getBean(beanName, requiredType, args));
				}
				else {
					candidates.put(beanName, getType(beanName));
				}
			}
			String candidateName = determinePrimaryCandidate(candidates, requiredType);
			if (candidateName == null) {
				candidateName = determineHighestPriorityCandidate(candidates, requiredType);
			}
			if (candidateName != null) {
				Object beanInstance = candidates.get(candidateName);
				if (beanInstance instanceof Class) {
					beanInstance = getBean(candidateName, requiredType, args);
				}
				return new NamedBeanHolder<T>(candidateName, (T) beanInstance);
			}
			throw new NoUniqueBeanDefinitionException(requiredType, candidates.keySet());
		}

		return null;
	}
```

整理逻辑很清晰：
> 0先根据classType获取所有的beanName，
> 1如果只有找到一个beanName，就直接根据beanName获取Bean实例了（getByBean）
> 2如果结果很多就要找到唯一的一个，要做两层过滤；
> 2.1 先看bean定义上是否是primary， 
> 2.2 如果仍然大于一个beanNames，再根据@Primary过滤最高优先级的；

```java
private String[] doGetBeanNamesForType(ResolvableType type, boolean includeNonSingletons, boolean allowEagerInit) {
		List<String> result = new ArrayList<String>();

		// Check all bean definitions.
		for (String beanName : this.beanDefinitionNames) {
			// 过滤别名的bean
			if (!isAlias(beanName)) {
				try {
					RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
					// 过滤抽象的bean，过滤
					if (!mbd.isAbstract() && (allowEagerInit ||
							((mbd.hasBeanClass() || !mbd.isLazyInit() || isAllowEagerClassLoading())) &&
									!requiresEagerInitForType(mbd.getFactoryBeanName()))) {
						// In case of FactoryBean, match object created by FactoryBean.
						boolean isFactoryBean = isFactoryBean(beanName, mbd);
						BeanDefinitionHolder dbd = mbd.getDecoratedDefinition();
						boolean matchFound =
								(allowEagerInit || !isFactoryBean ||
										(dbd != null && !mbd.isLazyInit()) || containsSingleton(beanName)) &&
								(includeNonSingletons ||
										(dbd != null ? mbd.isSingleton() : isSingleton(beanName))) &&
								isTypeMatch(beanName, type);
						if (!matchFound && isFactoryBean) {
							// In case of FactoryBean, try to match FactoryBean instance itself next.
							beanName = FACTORY_BEAN_PREFIX + beanName;
							matchFound = (includeNonSingletons || mbd.isSingleton()) && isTypeMatch(beanName, type);
						}
						if (matchFound) {
							result.add(beanName);
						}
					}
				}
				catch (CannotLoadBeanClassException ex) {
					if (allowEagerInit) {
						throw ex;
					}
					// Probably contains a placeholder: let's ignore it for type matching purposes.
					if (this.logger.isDebugEnabled()) {
						this.logger.debug("Ignoring bean class loading failure for bean '" + beanName + "'", ex);
					}
					onSuppressedException(ex);
				}
				catch (BeanDefinitionStoreException ex) {
					if (allowEagerInit) {
						throw ex;
					}
					// Probably contains a placeholder: let's ignore it for type matching purposes.
					if (this.logger.isDebugEnabled()) {
						this.logger.debug("Ignoring unresolvable metadata in bean definition '" + beanName + "'", ex);
					}
					onSuppressedException(ex);
				}
			}
		}

		// 校验人工注册的单例
		for (String beanName : this.manualSingletonNames) {
			try {
				// In case of FactoryBean, match object created by FactoryBean.
				if (isFactoryBean(beanName)) {
					if ((includeNonSingletons || isSingleton(beanName)) && isTypeMatch(beanName, type)) {
						result.add(beanName);
						// Match found for this bean: do not match FactoryBean itself anymore.
						continue;
					}
					// In case of FactoryBean, try to match FactoryBean itself next.
					beanName = FACTORY_BEAN_PREFIX + beanName;
				}
				// Match raw bean instance (might be raw FactoryBean).
				if (isTypeMatch(beanName, type)) {
					result.add(beanName);
				}
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Shouldn't happen - probably a result of circular reference resolution...
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to check manually registered singleton with name '" + beanName + "'", ex);
				}
			}
		}

		return StringUtils.toStringArray(result);
	}
```

doGetBeanNamesForType没有真正实现name是否匹配class的逻辑；这一判断在AbstractBeanFactory中实现

```java
	@Override
	public boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException {
		// 转成真正的beanName，没有&号，不是alias
		String beanName = transformedBeanName(name);

		// 检查当前已注册初始化的单例实例对象，排除过早暴露的单例
		// 如果找到实例，判断实例类型是否为FactoryBean
		//     如果该实例为FactoryBean，
		//     如果不属于FactoryBean，
		// 如果没有找到，可能之前注册的就是空对象。所以要考虑是否加过这个beanName的定义
		Object beanInstance = getSingleton(beanName, false);
		if (beanInstance != null) {
			if (beanInstance instanceof FactoryBean) {
				if (!BeanFactoryUtils.isFactoryDereference(name)) {
					Class<?> type = getTypeForFactoryBean((FactoryBean<?>) beanInstance);
					return (type != null && typeToMatch.isAssignableFrom(type));
				}
				else {
					return typeToMatch.isInstance(beanInstance);
				}
			}
			else if (!BeanFactoryUtils.isFactoryDereference(name)) {
				if (typeToMatch.isInstance(beanInstance)) {
					// Direct match for exposed instance?
					return true;
				}
				else if (typeToMatch.hasGenerics() && containsBeanDefinition(beanName)) {
					// Generics potentially only match on the target class, not on the proxy...
					RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
					Class<?> targetType = mbd.getTargetType();
					if (targetType != null && targetType != ClassUtils.getUserClass(beanInstance) &&
							typeToMatch.isAssignableFrom(targetType)) {
						// Check raw class match as well, making sure it's exposed on the proxy.
						Class<?> classToMatch = typeToMatch.resolve();
						return (classToMatch == null || classToMatch.isInstance(beanInstance));
					}
				}
			}
			return false;
		}
		else if (containsSingleton(beanName) && !containsBeanDefinition(beanName)) {
			return false;
		}

		// 如果没有找到实例，如果有ParentBeanFactory，
		// 如果当前的BeanFactory没有该BeanDefinition，那就委托给他的父容器中查找
		BeanFactory parentBeanFactory = getParentBeanFactory();
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			return parentBeanFactory.isTypeMatch(originalBeanName(name), typeToMatch);
		}

		// 到这，说明该类还没初始化
		RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);

		Class<?> classToMatch = typeToMatch.resolve();
		if (classToMatch == null) {
			classToMatch = FactoryBean.class;
		}
		Class<?>[] typesToMatch = (FactoryBean.class == classToMatch ?
				new Class<?>[] {classToMatch} : new Class<?>[] {FactoryBean.class, classToMatch});

		// Check decorated bean definition, if any: We assume it'll be easier
		// 检查该bean是否有decorated definition（别名）。
		BeanDefinitionHolder dbd = mbd.getDecoratedDefinition();
		if (dbd != null && !BeanFactoryUtils.isFactoryDereference(name)) {
			RootBeanDefinition tbd = getMergedBeanDefinition(dbd.getBeanName(), dbd.getBeanDefinition(), mbd);
			Class<?> targetClass = predictBeanType(dbd.getBeanName(), tbd, typesToMatch);
			if (targetClass != null && !FactoryBean.class.isAssignableFrom(targetClass)) {
				return typeToMatch.isAssignableFrom(targetClass);
			}
		}

		// （从可能的typeToMatch中）推测这个beanName的具体Class类型
		Class<?> beanType = predictBeanType(beanName, mbd, typesToMatch);
		if (beanType == null) {
			return false;
		}

		// 如果上面推测到的Class是FactoryBean子类，name又不符合FactoryBean的形式
		if (FactoryBean.class.isAssignableFrom(beanType)) {
			if (!BeanFactoryUtils.isFactoryDereference(name)) {
				// If it's a FactoryBean, we want to look at what it creates, not the factory class.
				beanType = getTypeForFactoryBean(beanName, mbd);
				if (beanType == null) {
					return false;
				}
			}
		}
		else if (BeanFactoryUtils.isFactoryDereference(name)) {
			// Special case: A SmartInstantiationAwareBeanPostProcessor returned a non-FactoryBean
			// type but we nevertheless are being asked to dereference a FactoryBean...
			// Let's check the original bean class and proceed with it if it is a FactoryBean.
			beanType = predictBeanType(beanName, mbd, FactoryBean.class);
			if (beanType == null || !FactoryBean.class.isAssignableFrom(beanType)) {
				return false;
			}
		}

		ResolvableType resolvableType = mbd.targetType;
		if (resolvableType == null) {
			resolvableType = mbd.factoryMethodReturnType;
		}
		if (resolvableType != null && resolvableType.resolve() == beanType) {
			return typeToMatch.isAssignableFrom(resolvableType);
		}
		return typeToMatch.isAssignableFrom(beanType);
	}
```

回顾一下getByClassType，逻辑就是先由ClassType找到“最适合”的beanName，然后就是getBeanByName；下面分析下getBeanByName的过程，getBeanByName是在AbstractBeanFactory中实现的

```java
protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {
		// Step 1：转成原始的bean name
		final String beanName = transformedBeanName(name);
		Object bean;

		// Step 2：检索本地单例库是否已经实例化该name的Bean
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
		else {
			// Step 3.0 校验该bean是否是正在创建的prototype
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Step 3.1 校验这个beanName定义是否存在当前的容器中；
			// 如果不存在且存在父容器，则在父容器找；
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				String nameToLookup = originalBeanName(name);
				if (args != null) {
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

			// Step 3.2 如果不只是检查类型，将该beanName加到创建完的bean集合中
			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			// Step 3.3 实际创建逻辑
			try {
				// Step 3.3.1 生成该BeanName的RootBeanDefinition对象
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				// Step 3.3.2 校验该RootBeanDefinition，不能为抽象类
				checkMergedBeanDefinition(mbd, beanName, args);

				// Step 3.3.3 确保当前Bean依赖的bean对象都已经实例化了
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						getBean(dep);
					}
				}

				// Step 3.3.4 初始化。。。（Singleton，or Prototype or 自定义Scope）
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
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
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
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

		// Step 4：校验实例化的bean跟入参的classType类型是否一致，利用TypeConverter尝试转化类型
		if (requiredType != null && bean != null && !requiredType.isInstance(bean)) {
			try {
				return getTypeConverter().convertIfNecessary(bean, requiredType);
			}
			catch (TypeMismatchException ex) {
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```

doGetBean实现了getBeanByBeanName的整体逻辑，流程大致分成以下几个:
> 1. 转成原始的bean name
> 2. 检索本地单例库是否已经实例化该name的Bean
> 3. 2没找到，校验该bean name是否是正在创建的prototype
> 4. 校验这个beanName定义是否存在当前的容器中；如果不存在且存在父容器，则在父容器找；
> 5. 如果不只是检查类型，将该beanName加到创建完的bean集合中
> 6. 实际创建bean逻辑
> 7. 校验实例化的bean跟入参的classType类型是否一致，利用TypeConverter尝试转化类型
> 8. 返回该bean实例
其中第六步实际创建bean，又根据bean的是否为scope分成3个小流程。整体看又是统一用createBean方法获取实例化对象，又用getObjectForBeanInstance处理FactoryBean和普通bean的差异性，最终返回真实的bean对象。

```java
    @Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
		RootBeanDefinition mbdToUse = mbd;

		// Step 1：到这为止，bean的Class已经确定。如果之前没有Class就克隆一份Bean Definition，并将解析到的class配置好
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Step 2：处理bean上的look-up、replace标签
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Step 3：Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		// Step 4：实际创建bean
		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
		if (logger.isDebugEnabled()) {
			logger.debug("Finished creating instance of bean '" + beanName + "'");
		}
		return beanInstance;
	}
```

createBean逻辑：
1. 最后确认该bean对象的Class值:
2. 处理该bean上的look-up、replace标签
3. bean初始化前处理一遍BeanProcessor
4. doCreateBean

```java
        protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
			throws BeanCreationException {

		// Step 1：获取BeanWrapper，先从本地缓存找，没有则创建（createBeanInstance）
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
		Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
		mbd.resolvedTargetType = beanType;

		// Step 2：处理存在的MergedBeanDefinitionPostProcessor
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Step 3：如果允许bean提前暴露（为解决循环依赖的问题），则创建提前暴露的实例对象
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			addSingletonFactory(beanName, new ObjectFactory<Object>() {
				@Override
				public Object getObject() throws BeansException {
					return getEarlyBeanReference(beanName, mbd, bean);
				}
			});
		}

		// Step 4：填充实例的属性，（initializeBean）处理Aware，
        // BeanPostProcessor（postProcessBeforeInitialization，postProcessAfterInitialization），init-method，
		Object exposedObject = bean;
		try {
			populateBean(beanName, mbd, instanceWrapper);
			if (exposedObject != null) {
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

		// Step 5：尝试不允许提早暴露获取实例
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

		// Step 6：注册Bean的销毁逻辑
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

1. 获取BeanWrapper，先从本地缓存找，没有则创建【createBeanInstance用反射和构造函数，实现了类的对象实例化】
2. 处理存在的MergedBeanDefinitionPostProcessor
3. 如果允许bean提前暴露（为解决循环依赖的问题），则创建提前暴露的实例对象
4. 填充实例的属性【populateBean实现了依赖的注入（by name or by type）】
5. 尝试不允许提早暴露获取实例
6. 注册Bean的销毁逻辑



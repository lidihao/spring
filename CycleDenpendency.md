Spring 依赖注入之循环依赖

标签（空格分隔）： spring

---

#1.实体类:
    public class A {
    String name;
    B b;
    public B getB() {
        return b;
    }
    public void setB(B b) {
        this.b = b;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
    }

    public class B {
    String name;
    A a;

    public A getA() {
        return a;
    }

    public void setA(A a) {
        this.a = a;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }}
#2.配置文件
        
        <?xml version="1.0" encoding="UTF-8"?>
        <!--suppress ALL-->
        <beans xmlns="http://www.springframework.org/schema/beans"
               xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
               xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
        <bean id="A" class="com.hao.bean.A">
            <property name="name">
                <value>AClass</value>
            </property>
            <property name="b" ref="B"></property>
        </bean>
            <bean id="B" class="com.hao.bean.B">
                <property name="name" 
            value="BClass"></property>
                <property name="a" ref="A"></property>
            </bean>
        </beans>
当依赖注入时,A类会引用B类,B类又会引用A类,造成循环依赖.如果是构造器注入就会抛出异常,如果是setter注入且是单例的,则可以解决.
    解决方法:主要依赖DefaultSingletonBeanRegistry中的三个map:
    ---Map<String, Object> singletonObjects 用来存储已经创建完成的单例
bean;
    Map<String, ObjectFactory<?>> singletonFactories 用来存储暴露的ObjectFactory
    Map<String, Object> earlySingletonObjects 用来存储已经创建完成但没有依赖注入的Bean
#3.代码示例
doGetBean代码
第一步:



    protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {

		final String beanName = transformedBeanName(name);
		Object bean;

		// 这一步从缓存中拿Bean
		Object sharedInstance = getSingleton(beanName);      //1
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
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dependsOnBean : dependsOn) {
						if (isDependent(beanName, dependsOnBean)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dependsOnBean + "'");
						}
						registerDependentBean(dependsOnBean, beanName);
						getBean(dependsOnBean);
					}
				}

				// Create bean instance.
				if (mbd.isSingleton()) {
					****sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							try {
								return createBean(beanName, mbd, args);****2
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

		// Check if required type matches the type of the actual bean instance.
		if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
			try {
				return getTypeConverter().convertIfNecessary(bean, requiredType);
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type [" +
							ClassUtils.getQualifiedName(requiredType) + "]", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}


第二步:getSingleton(String beanName, boolean allowEarlyReference)


    protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    //从缓存真正创建完毕的缓存中拿SingletonBean
    		Object singletonObject = this.singletonObjects.get(beanName);
    		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
    			synchronized (this.singletonObjects) {
    //如果不存在则从实例化完毕还没依赖注入的缓存中拿SingletonBean
    				singletonObject = this.earlySingletonObjects.get(beanName);
    				if (singletonObject == null && allowEarlyReference) {
    //如果还不存在从暴露的ObjectFactory中getObject返回一个对象
    					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
    					if (singletonFactory != null) {
    						singletonObject = singletonFactory.getObject();
    				//将ObjectFactory移除,将对象放入earlySingletonObjects中
    					this.earlySingletonObjects.put(beanName, singletonObject);
    						this.singletonFactories.remove(beanName);
    					}
    				}
    			}
    		}
    		return (singletonObject != NULL_OBJECT ? singletonObject : null);
    	}


第三步:暴露ObjectFactory

1.getSingleton(String beanName, ObjectFactory<?> singletonFactory)

    public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    		Assert.notNull(beanName, "'beanName' must not be null");
    		synchronized (this.singletonObjects) {
    			Object singletonObject = this.singletonObjects.get(beanName);
    			if (singletonObject == null) {
    				if (this.singletonsCurrentlyInDestruction) {
    					throw new BeanCreationNotAllowedException(beanName,
    							"Singleton bean creation not allowed while the singletons of this factory are in destruction " +
    							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
    				}
    				if (logger.isDebugEnabled()) {
    					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
    				}
    				//将Bean放入singletonsCurrentlyInCreation中
    				beforeSingletonCreation(beanName);
    				boolean newSingleton = false;
    				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
    				if (recordSuppressedExceptions) {
    					this.suppressedExceptions = new LinkedHashSet<Exception>();
    				}
    				try {
    				//使用ObjectFactory的getObject方法,返回对象
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
    					afterSingletonCreation(beanName);
    				}
    				if (newSingleton) {
    					addSingleton(beanName, singletonObject);
    				}
    			}
    			return (singletonObject != NULL_OBJECT ? singletonObject : null);
    		}
    	}


2.直接看doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)



    protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
    		// Instantiate the bean.
    		BeanWrapper instanceWrapper = null;
    		if (mbd.isSingleton()) {
    			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    		}
    		if (instanceWrapper == null) {
    			instanceWrapper = createBeanInstance(beanName, mbd, args);
    		}
    		final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
    		Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
    
    		// Allow post-processors to modify the merged bean definition.
    		synchronized (mbd.postProcessingLock) {
    			if (!mbd.postProcessed) {
    				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
    				mbd.postProcessed = true;
    			}
    		}
    
    		// Eagerly cache singletons to be able to resolve circular references
    		// even when triggered by lifecycle interfaces like BeanFactoryAware.
    		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
    				isSingletonCurrentlyInCreation(beanName));
    		if (earlySingletonExposure) {
    			if (logger.isDebugEnabled()) {
    				logger.debug("Eagerly caching bean '" + beanName +
    						"' to allow for resolving potential circular references");
    			}
    			
    			//暴露ObjectFactory,注意beanName,mbd,bean都是final
    			addSingletonFactory(beanName, new ObjectFactory<Object>() {
    				@Override
    				public Object getObject() throws BeansException {
    					return getEarlyBeanReference(beanName, mbd, bean);
    				}
    			});
    		}
    
    		// Initialize the bean instance.
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
    				throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
    			}
    		}
    
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
    
    		// Register bean as disposable.
    		try {
    			registerDisposableBeanIfNecessary(beanName, bean, mbd);
    		}
    		catch (BeanDefinitionValidationException ex) {
    			throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
    		}
    
    		return exposedObject;
    	}

以上就是代码过程.
#4.文字解说
就拿A和B两个类作为例子:
    首先getBean("A")创建实例,从getSingleton(String beanName)中获取,在这个方法中





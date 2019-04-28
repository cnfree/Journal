# 记调试FF4J的全过程
关键字： `FF4J` `@Flip` `FeatureAutoProxy` `FeatureAdvisor` `AbstractAutoProxyCreator` `ProxyFactory`

本文旨在记录**Spring Bean**加载及调用的全过程

## FF4J产品功能开关配置
  1. 功能开关服务接口
        ```
            public interface ProductTagService {
            
                /**
                 * 产品Tag功能开关
                 * @param product
                 * @param person
                 * @return
                 */
                @Flip(name = "ProductTagDisabled:isUserEnabled", alterBean = "offlineProductTag")
                boolean isUserEnabled(ProductMenuInfo product, Person person);
            }
        ```
  2. 功能开关主实现
        ```
            @Service("onlineProductTag")
            @Primary
            public class OnlineProdcutTagServiceImpl implements ProductTagService {
                @Override
                public boolean isUserEnabled(ProductMenuInfo product, Person person) {
                    ...
                }
            }

        ```
  3. 功能开关alterBean实现
        ```
            @Service("offlineProductTag")
            public class OfflineProductTagServiceImpl implements ProductTagService {
            
                @Override
                public boolean isUserEnabled(ProductMenuInfo product, Person person) {
                    return true;
                }
            }
        ```

## 扫描Package
 1. 功能开关相关Bean的扫描解析过程
    1. component-scan **org.ff4j.aop**
    2. Spring初始化的时候会扫描org.ff4j.aop包，具体是在ClassPathBeanDefinitionScanner类的doScan(String... basePackages)方法
    3. ClassPathScanningCandidateComponentProvider的findCandidateComponents(String basePackage)方法扫描ClassPath下的org.ff4j.aop包
    4. 扫描实现是使用API ClassLoader.getResources("org.ff4j.aop)，在PathMatchingResourcePatternResolver类的doFindAllClassPathResources(String path)方法中
    5. ClassPathScanningCandidateComponentProvider的isCandidateComponent方法检测是否实现了 **@Component** 注解
    6. ClassPathBeanDefinitionScanner类的registerBeanDefinition方法会将扫描到的Component注册到容器 **DefaultListableBeanFactory** 中，通过DefaultListableBeanFactory.registerBeanDefinition方法 
    7. 以上过程都发生在**AbstractApplicationContext**的**refresh**方法的**obtainFreshBeanFactory**中
    ```
        prepareRefresh();
        
        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        }
    ```
    
 2. 功能开关相关Bean的注册过程
    1. 在上述代码里的registerBeanPostProcessors(beanFactory)中注册了所有的BeanProcessor，因此BeanProcessor优先于其他的Bean加载
 
    2. org.ff4j.aop.FeatureAutoProxy继承了AbstractAutoProxyCreator类，这个类实现了BeanPostProcessor接口
        ```
            public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
                    implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware
    
        ```
 
    3. AbstractAutoProxyCreator是Spring AOP实现的核心类，实现了代理创建的逻辑
 
    4. Spring AOP里的AbstractAutoProxyCreator类结构
        ![AbstractAutoProxyCreator][AbstractAutoProxyCreator]
 
    5. 在BeanProcessor之后的所有Bean实例化过程中都要被所有的BeanProcessor处理，参见类AbstractAutowireCapableBeanFactory
         ```
            public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
                        throws BeansException {
            
                Object result = existingBean;
                for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
                    result = beanProcessor.postProcessAfterInitialization(result, beanName);
                    if (result == null) {
                        return result;
                    }
                }
                return result;
            }
        
         ```
 
    6. 因此所有的Bean在初始化过程中都会被FeatureAutoProxy扫描
        ```
            protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, TargetSource targetSource) {
                // Do not used any AOP here as still working with classes and not objects
                if (!beanClass.isInterface()) {
                    Class<?>[] interfaces;
                    if (ClassUtils.isCglibProxyClass(beanClass)) {
                        interfaces = beanClass.getSuperclass().getInterfaces();
                    } else {
                        interfaces = beanClass.getInterfaces();
                    }
                    if (interfaces != null) {
                        for (Class<?> currentInterface: interfaces) {
                            Object[] r = scanInterface(currentInterface);
                            if (r != null) {
                                return r;
                            }
                        }
                    }
                }
                return DO_NOT_PROXY;
            }
        ```
 
    7. AbstractApplicationContext的finishBeanFactoryInitialization(beanFactory)中，会实例化所有被扫描到的@Component
         ```
            public void preInstantiateSingletons() throws BeansException {
                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("Pre-instantiating singletons in " + this);
                }
        
                // Iterate over a copy to allow for init methods which in turn register new bean definitions.
                // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
                List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);
        
                // Trigger initialization of all non-lazy singleton beans...
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
                ...
            }
        
         ```
 
    8. FeatureAutoProxy的父类**AbstractAutoProxyCreator**在方法**wrapIfNecessary**中将扫描到包含 **@Flip** 注解的Class注册到Map里，然后创建代理
        ```
             /*
              * Wrap the given bean if necessary, i.e. if it is eligible for being proxied.
              * @param bean the raw bean instance
              * @param beanName the name of the bean
              * @param cacheKey the cache key for metadata access
              * @return a proxy wrapping the bean, or the raw bean instance as-is
              */
             protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
                 if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
                     return bean;
                 }
                 if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
                     return bean;
                 }
                 if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
                     this.advisedBeans.put(cacheKey, Boolean.FALSE);
                     return bean;
                 }
         
                 // Create proxy if we have advice.
                 //扫描Bean是否包含@Flip
                 Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
                 if (specificInterceptors != DO_NOT_PROXY) {
                     this.advisedBeans.put(cacheKey, Boolean.TRUE);
                     //创建代理
                     Object proxy = createProxy(
                             bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
                     this.proxyTypes.put(cacheKey, proxy.getClass());
                     return proxy;
                 }
         
                 this.advisedBeans.put(cacheKey, Boolean.FALSE);
                 return bean;
             }
        ```
 
    9. 创建代理首先需要创建org.springframework.aop.Advisor,在AbstractAutoProxyCreator中有一个方法setInterceptorNames,该方法设置代理的拦截器组件，每一个拦截器会被转成一个Advisor
    
        ```
            /*
             * Resolves the specified interceptor names to Advisor objects.
             * @see #setInterceptorNames
             */
            private Advisor[] resolveInterceptorNames() {
                ConfigurableBeanFactory cbf = (this.beanFactory instanceof ConfigurableBeanFactory ?
                        (ConfigurableBeanFactory) this.beanFactory : null);
                List<Advisor> advisors = new ArrayList<Advisor>();
                for (String beanName : this.interceptorNames) {
                    if (cbf == null || !cbf.isCurrentlyInCreation(beanName)) {
                        Object next = this.beanFactory.getBean(beanName);
                        advisors.add(this.advisorAdapterRegistry.wrap(next));
                    }
                }
                return advisors.toArray(new Advisor[advisors.size()]);
            }
        ```
    
    10. 创建代理的过程就是创建一个ProxyFactory，传入Advisor和要代理的对象实例，然后通过ProxyFactory创建代理对象
    
        ```
            /*
             * Create an AOP proxy for the given bean.
             * @param beanClass the class of the bean
             * @param beanName the name of the bean
             * @param specificInterceptors the set of interceptors that is
             * specific to this bean (may be empty, but not null)
             * @param targetSource the TargetSource for the proxy,
             * already pre-configured to access the bean
             * @return the AOP proxy for the bean
             * @see #buildAdvisors
             */
            protected Object createProxy(
                    Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {
        
                if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
                    AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
                }
        
                ProxyFactory proxyFactory = new ProxyFactory();
                proxyFactory.copyFrom(this);
        
                if (!proxyFactory.isProxyTargetClass()) {
                    if (shouldProxyTargetClass(beanClass, beanName)) {
                        proxyFactory.setProxyTargetClass(true);
                    }
                    else {
                        evaluateProxyInterfaces(beanClass, proxyFactory);
                    }
                }
        
                Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
                proxyFactory.addAdvisors(advisors);
                proxyFactory.setTargetSource(targetSource);
                customizeProxyFactory(proxyFactory);
        
                proxyFactory.setFrozen(this.freezeProxy);
                if (advisorsPreFiltered()) {
                    proxyFactory.setPreFiltered(true);
                }
        
                return proxyFactory.getProxy(getProxyClassLoader());
            }
        ```
    11. ProxyFactory实际上是AopProxyFactory的代理，创建代理实际上是通过AopProxyFactory完成的
    
        ```
            public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {
            
                @Override
                public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
                    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
                        Class<?> targetClass = config.getTargetClass();
                        if (targetClass == null) {
                            throw new AopConfigException("TargetSource cannot determine target class: " +
                                    "Either an interface or a target is required for proxy creation.");
                        }
                        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
                            return new JdkDynamicAopProxy(config);
                        }
                        return new ObjenesisCglibAopProxy(config);
                    }
                    else {
                        return new JdkDynamicAopProxy(config);
                    }
                }
        ```
 
    12. 代理有两种，一种是JdkDynamicAopProxy，一种是ObjenesisCglibAopProxy，不管哪种，都持有AdvisedSupport，这个AdvisedSupport就是之前提到的ProxyFactory，它继承了AdvisedSupport，而ProxyFactory又持有Advisor(持有拦截器)和需要被代理的对象实例，因此两种代理就这样关联到拦截器和对象实例了。
    13. JdkDynamicAopProxy实现了JDK的动态代理接口InvocationHandler，同时也实现了AopProxy接口。AopProxy接口是用来创建代理对象的接口。
         ```
            public Object getProxy(ClassLoader classLoader) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
                }
                Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
                findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
                return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
            }
         ``` 
    14. 至此包含了@Flip注解的对象的代理就被创建出来，并作为Bean对象被暴露出来。
 
 3. 功能开关的执行过程 
    1. 首先明确一点，FF4J自定义了代理、拦截器通知实现，因此它并没有走标准的Spring Aop流程。Aop的几个概念中，它只有：
        1. 连接点JointPoint：被@Flip注解的方法
        2. 通知Advisor：FeatureAdvisor实现
        3. 目标对象：含有@Flip注解的接口的主实现Bean
    2. 调用@Flip注解方法，走的是JdkDynamicAopProxy代理invoke方法，在里面创建了一个ReflectiveMethodInvocation对象代理执行。
    3. ReflectiveMethodInvocation通过责任链模式来调用拦截器，这个模式基本是Spring各模块拦截器实现的标配了。 
       > ReflectiveMethodInvocation结构 
        ![ReflectiveMethodInvocation][ReflectiveMethodInvocation]
        ```
            public Object proceed() throws Throwable {
        		//	We start with an index of -1 and increment early.
        		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        			return invokeJoinpoint();
        		}
        
        		Object interceptorOrInterceptionAdvice =
        				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
        		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
        			// Evaluate dynamic method matcher here: static part will already have
        			// been evaluated and found to match.
        			InterceptorAndDynamicMethodMatcher dm =
        					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
        			if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
        				return dm.interceptor.invoke(this);
        			}
        			else {
        				// Dynamic matching failed.
        				// Skip this interceptor and invoke the next in the chain.
        				return proceed();
        			}
        		}
        		else {
        			// It's an interceptor, so we just invoke it: The pointcut will have
        			// been evaluated statically before this object was constructed.
        			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
        		}
        	}

        ```
    4. FeatureAdvisor此时隆重登场，它的作用就是判断功能开关是否开启，如果开启并且当前Bean不是注解上的alterBean，执行alterBean，否则调用ReflectiveMethodInvocation的proceed方法
        ```
            public Object invoke(final MethodInvocation mi) throws Throwable {
                 Flip ff4jAnnotation = getFF4jAnnotation(mi);
                 
                 // Method is annotated
                 if (ff4jAnnotation != null) {
                 	
                 	String alterBean    = ff4jAnnotation.alterBean();
                 	Class<?> alterClazz = ff4jAnnotation.alterClazz();
                 	boolean isFeatureToggled = check(ff4jAnnotation, mi);
                 	
                 	// Would like to skip if feature is Disable
                 	if (!Util.hasLength(alterBean) && !Util.isValidClass(alterClazz) && !isFeatureToggled) {
                 		return null;
                 	}
                 	
                 	// Feature is 'ON'
                 	if (isFeatureToggled) {
         	            // Do we use the alter bean defined in the annotation ?
         	            // 如果功能开关开启，当前执行Bean不是alterBean，调用alterBean，否则当前就是alterBean，直接执行mi.proceed()
         	            if (Util.hasLength(alterBean)
         	                    // Bean name exist
         	                    & appCtx.containsBean(alterBean)   
         	                    // Bean name is not the same as current
         	                    & !alterBean.equals(getExecutedBeanName(mi))) {
         	                return invokeAlterBean(mi, alterBean);
         	            }
         	            
         	            // Or else do we use the alter class defined in the annotation ?
         	            if (Util.isValidClass(alterClazz) 
         	                    // Alter class is not the same as current
         	                    & (alterClazz != getExecutedClass(mi))) {
         	                return invokeAlterClazz(mi, ff4jAnnotation);
         	            }
                 	}
                 }
                 // No feature toggle (no annotation nor feature OFF)
                 return mi.proceed();
             }

        ```
    5. 如果功能开关开启，会调用FeatureAdvisor.invoke两次，第一次执行PrimaryBean代理，第二次执行alterBean代理，因此会判断功能开关两次
    6. ReflectiveMethodInvocation.proceed()每被拦截器调用一次，拦截器计数减一。当计数为-1时，执行真正的目标对象方法
        ```
            /**
             * Invoke the joinpoint using reflection.
             * Subclasses can override this to use custom invocation.
             * @return the return value of the joinpoint
             * @throws Throwable if invoking the joinpoint resulted in an exception
             */
            protected Object invokeJoinpoint() throws Throwable {
                return AopUtils.invokeJoinpointUsingReflection(this.target, this.method, this.arguments);
            }
        ```
    7. 至此调用@Flip注解方法整个过程结束。
 
 
 [AbstractAutoProxyCreator]:img/AbstractAutoProxyCreator.png
 [ReflectiveMethodInvocation]:img/ReflectiveMethodInvocation.png   
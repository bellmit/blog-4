
[简书：单例模式讲解以及Spring中的单例实现，good](https://www.jianshu.com/p/d9ccf7c12865)
[Spring的单例模式底层实现学习笔记](https://www.cnblogs.com/zhaww/p/8467196.html)

那么Spring对单例的底层实现，到底是饿汉式单例还是懒汉式单例呢？
很多博文写到，这既不属于懒汉式也不属于饿汉式，是Spring框架对单例的支持是采用单例注册表的方式进行实现的。
但其实，单例注册表用来维护一个beanName->object的映射，方便之后的getBean。内核的实现思路，是double check lock方式。


以下为AbstractBeanFacotory中doGetBean的实现

```java
	/**
	 * Return an instance, which may be shared or independent, of the specified bean.

	 */
	@SuppressWarnings("unchecked")
	protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {
        //对传入的Bean name稍做处理，防止传入的Bean name名有非法字符(或则做转码) 
		String beanName = transformedBeanName(name);
		Object bean;

        //1. check缓存(单例注册表)中是否已经有该实例
		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
            ...
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
            ...
                // Create bean instance.
				if (mbd.isSingleton()) {
                    //2.核心方法：getSingleton()
					sharedInstance = getSingleton(beanName, () -> {
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
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
            ...
		}

		// Check if required type matches the type of the actual bean instance.
		if (requiredType != null && !requiredType.isInstance(bean)) {
            ...
		}
		return (T) bean;
	}

```

```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
	/** Cache of singleton objects: bean name to bean instance. */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	/** Cache of singleton factories: bean name to ObjectFactory. */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

	/** Cache of early singleton objects: bean name to bean instance. */
	private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

	/** Set of registered singletons, containing the bean names in registration order. */
	private final Set<String> registeredSingletons = new LinkedHashSet<>(256);

	/** Names of beans that are currently in creation. */
	private final Set<String> singletonsCurrentlyInCreation =
			Collections.newSetFromMap(new ConcurrentHashMap<>(16));

	/** Names of beans currently excluded from in creation checks. */
	private final Set<String> inCreationCheckExclusions =
			Collections.newSetFromMap(new ConcurrentHashMap<>(16));
    
	/**
	 * Return the (raw) singleton object registered under the given name.
	 * <p>Checks already instantiated singletons and also allows for an early
	 * reference to a currently created singleton (resolving a circular reference).
	 * @param beanName the name of the bean to look for
	 * @param allowEarlyReference whether early references should be created or not
	 * @return the registered singleton object, or {@code null} if none found
	 */
     //在1.中调用，check缓存(单例注册表)中是否已经有该实例
	@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		// Quick check for existing instance without full singleton lock
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
				synchronized (this.singletonObjects) {
					// Consistent creation of early reference within full singleton lock
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						singletonObject = this.earlySingletonObjects.get(beanName);
						if (singletonObject == null) {
							ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
							if (singletonFactory != null) {
								singletonObject = singletonFactory.getObject();
								this.earlySingletonObjects.put(beanName, singletonObject);
								this.singletonFactories.remove(beanName);
							}
						}
					}
				}
			}
		}
		return singletonObject;
	}


    	/**
	 * Return the (raw) singleton object registered under the given name,
	 * creating and registering a new one if none registered yet.
	 * @param beanName the name of the bean
	 * @param singletonFactory the ObjectFactory to lazily create the singleton
	 * with, if necessary
	 * @return the registered singleton object
	 */
     //在2.中调用，方法名都是getSingleton，但入参不一样
	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
		synchronized (this.singletonObjects) {
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}

				beforeSingletonCreation(beanName);
				boolean newSingleton = false;
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<>();
				}
				try {
                    //从singletonFactory中getObject();
                    //singletonFactory在入参定义
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				}
				catch (IllegalStateException ex) {
                    ...
				}
				catch (BeanCreationException ex) {
                    ...
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
					afterSingletonCreation(beanName);
				}
				if (newSingleton) {
                    //加入到singletonObjects这个map中
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
	}

	/**
	 * Add the given singleton object to the singleton cache of this factory.
	 * <p>To be called for eager registration of singletons.
	 * @param beanName the name of the bean
	 * @param singletonObject the singleton object
	 */
	protected void addSingleton(String beanName, Object singletonObject) {
		synchronized (this.singletonObjects) {
			this.singletonObjects.put(beanName, singletonObject);
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}

}
```
首先 BeanFactory中已经存储了我们从xml解析出来的相关信息放在BeanDefinitionMap（作用后期会讲）中，然后通过这个map获取到RootBeanDefinition（功能后期也会讲，在此理解为一个有属性值，构造方法和其他相关信息的Bean ） ，然后就有个判断if (mbd.isSingleton()) 如果是单例的就接着getSingleton的重载方法，传入的是mbd,


当从缓存中加载单例对象的时候singletonObjects这个map(用来存放缓存的单例)，并且只要创建一个单例对象就会把当前的单例对象存放在这个singletonObjects中，这样一来就保证了在getBean的时候这里面永远就只有一个，实现了我们在获取某个对象过得时候才会给她分配内存。既保证了内存高效利用，又是线程安全的

这样的话，我们就从实例中也获取到了Bean,也创建单例Bean的缓存，当下一次还需要这个单例Bean的时候就直接从缓存中获取了。

最后一点总结：\
Spring中创建单例的过程真的是非常的绕，但是逻辑还是非常清楚的，就是将我们需要的对象放在map中，下次需要的时候就直接从map中获取，只是spring在处理的时候，需要解决其他很多问题而已，到此单例模式就真的结束了。【‘


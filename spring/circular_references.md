[知乎：spring如何解决循环依赖](https://zhuanlan.zhihu.com/p/84267654)

[简书：spring如何解决循环依赖](https://www.jianshu.com/p/8bb67ca11831)

[知乎：阿里面试，Spring循坏依赖 好！](https://zhuanlan.zhihu.com/p/186212601)

spring三级缓存

DefaultSingletonBeanRegistry这个类中
```java
// 一级缓存，缓存正常的bean实例
/** Cache of singleton objects: bean name to bean instance. */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

// 二级缓存，缓存还未进行依赖注入和初始化方法调用的bean实例 半成品
/** Cache of early singleton objects: bean name to bean instance. */
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

// 三级缓存，缓存bean实例的ObjectFactory
/** Cache of singleton factories: bean name to ObjectFactory. */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

```

```java
public Object getSingleton(String beanName) {
    return getSingleton(beanName, true);
}

protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 先尝试中一级缓存获取
    Object singletonObject = this.singletonObjects.get(beanName);
    // 获取不到，并且当前需要获取的bean正在创建中
    // 第一次容器初始化触发getBean(A)的时候，这个isSingletonCurrentlyInCreation判断一定为false
    // 这个时候就会去走创建bean的流程，创建bean之前会先把这个bean标记为正在创建
    // 然后A实例化之后，依赖注入B，触发B的实例化，B再注入A的时候，会再次触发getBean(A)
    // 此时isSingletonCurrentlyInCreation就会返回true了

    // 当前需要获取的bean正在创建中时，代表出现了循环依赖（或者一前一后并发获取这个bean）
    // 这个时候才需要去看二、三级缓存
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // 加锁了
        synchronized (this.singletonObjects) {
            // 从二级缓存获取
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                // 二级缓存也没有，并且允许获取早期引用的话 - allowEarlyReference传进来是true
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                // 从三级缓存获取ObjectFactory
                if (singletonFactory != null) {
                    // 通过ObjectFactory获取bean实例
                    singletonObject = singletonFactory.getObject();
                    // 放入二级缓存
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    // 从三级缓存删除
                    // 也就是说对于一个单例bean，ObjectFactory#getObject只会调用到一次
                    // 获取到早期bean实例之后，就把这个bean实例从三级缓存升级到二级缓存了
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    // 不管从哪里获取到的bean实例，都会返回
    return singletonObject;
}

```

我们回到这个三级缓存的结构，二级缓存是是在getSingleton方法中put进去的，这跟我们之前分析的，创建bean实例之后放入，好像不太一样？那我们是不是可以推断一下，其实创建bean实例之后，是放入三级缓存的呢（总之实例创建之后是需要放入缓存的）？我们来跟一下bean实例化的代码，主要看一下上一篇时刻意忽略掉的地方：

```java
// 代码做了很多删减，只把主要的逻辑放出来的
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
    throws BeanCreationException {

    // 创建bean实例
    BeanWrapper instanceWrapper = createBeanInstance(beanName, mbd, args);
    final Object bean = instanceWrapper.getWrappedInstance();
    // beanPostProcessor埋点调用
    applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);

    // 重点是这里了，如果是单例bean&&允许循环依赖&&当前bean正在创建
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                      isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        // 加入三级缓存
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    Object exposedObject = bean;
    try {
        // 依赖注入
        populateBean(beanName, mbd, instanceWrapper);
        // 初始化方法调用
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(...); 
    }

    if (earlySingletonExposure) {
        // 第二个参数传false是不会从三级缓存中取值的
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            // 如果发现二级缓存中有值了 - 说明出现了循环依赖
            if (exposedObject == bean) {
                // 并且initializeBean没有改变bean的引用
                // 则把二级缓存中的bean实例返回出去
                exposedObject = earlySingletonReference;
            }
        }
    }

    try {
        // 注册销毁逻辑
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    catch (BeanDefinitionValidationException ex) {
        throw new BeanCreationException(...);
    }

    return exposedObject;
}

```

可是按我们自己的设计思路，明明只需要两级缓存就可以解决，spring却使用了三级缓存，难道是为了炫技么？

原来为了解决initializeBean可能替换bean引用的问题，spring就设计了这个三级缓存，他在第三级里保存了一个ObjectFactory，其实具体就是getEarlyBeanReference的调用，其中提供了一个getEarlyBeanReference的埋点方法，通过这个埋点方法，它允许开发人员把需要替换的bean，提早替换出来。
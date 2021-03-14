
首先来看一下aop的wiki解释：
>In computing, aspect-oriented programming (AOP) is a programming paradigm that aims to increase modularity by allowing the separation of cross-cutting concerns. It does so by adding additional behavior to existing code (an advice) without modifying the code itself, instead separately specifying which code is modified via a "pointcut" specification, such as "log all function calls when the function's name begins with 'set'". This allows behaviors that are not central to the business logic (such as logging) to be added to a program without cluttering the code core to the functionality. AOP forms a basis for aspect-oriented software development.

通过这段解释，我们可以看出AOP带来的好处
1. 模块化
2. 不修改源代码的前提下，增加下新功能。 这符合开闭原则。
3. 将业务代码和非业务代码分隔开。对非业务功能(e.g. 日志、权限控制、监控设定)等，统一管理。

### 切面aspect是什么？为什么会有切面？


OOP引入封装、继承、多态等概念来建立一种对象层次结构，用于模拟公共行为的一个集合。不过OOP允许开发者**定义纵向的关系**，但并不适合**定义横向的关系**，例如日志功能。日志代码往往横向地散布在所有对象层次中，而与它对应的对象的核心功能毫无关系对于其他类型的代码，如安全性、异常处理和透明的持续性也都是如此，这种散布在各处的无关的代码被称为横切（cross cutting），在OOP设计中，它导致了大量代码的重复，而不利于各个模块的重用。\
AOP技术恰恰相反，它利用一种称为"横切"的技术，剖解开封装的对象内部，**并将那些影响了多个类的公共行为封装到一个可重用模块，并将其命名为"Aspect"，即切面。** 所谓"切面"，简单说就是那些与业务无关，却为业务模块所共同调用的逻辑或责任封装起来，便于减少系统的重复代码，降低模块之间的耦合度，并有利于未来的可操作性和可维护性。\
aop并不是来替代oop的，而是对oop的一种补充和完善，两者可以很好地配合使用。


### 应用AOP之一：缓存
通常我们会将日志、权限控制、报警设定等放到切面中实现。但其实它的应用不止于此。比如我们可以给一些方法加一层缓存（缓存可以用guava来实现）。
```java
@Aspect
public CacheAspect{
  Cache cache;
  //切入点：AccountInfoService对象的getAccountInfo方法
  @Pointcut("execution(* org.enactus.zhangzhen.AccountService.getAccountInfo(..))")
  public void pointcut(){}

  @Around(value = "pointcut()")
  public Object around(ProceedingJointPoint pjp){
    AccountService accountService = (AccountService)pjp.getTarget();
    //缓存中没有，执行原方法
    if(null==cache.get(accountService.getAlias()){
      return pjp.proceed(pjp.getArgs());
    }
    //缓存中有，直接返回缓存中内容，不需要执行原方法。
    else{
      return cache.get(accountService.getAlias());
    }

  }

}
```

>参考\
>[掘金：spring使用详解](https://juejin.cn/post/6844903965398663182)\
>[博客园：aop详解](https://www.cnblogs.com/xrq730/p/4919025.html)\
>[知乎: aop详解](https://zhuanlan.zhihu.com/p/37497663)\
>[思否：SpringAOP实现原理](https://segmentfault.com/a/1190000037596406)


### 动态代理

代理模式：给某一个对象提供一个代理，并由代理对象来控制对真实对象的访问。代理模式是一种结构型设计模式。
+ 静态代理：所谓静态也就是在程序运行前就已经存在代理类的字节码文件，代理类和真实主题角色的关系在运行前就确定了。
+ 动态代理：而动态代理的源码是在程序运行期间由JVM根据反射等机制动态的生成，所以在运行前并不存在代理类的字节码文件。
  + 通过实现接口的方式 -> JDK动态代理
  + 通过继承类的方式 -> CGLIB动态代理

#### jdk动态代理

在java的动态代理机制中，有两个重要的类或接口，一个是 InvocationHandler(Interface)、另一个则是 Proxy(Class)，这一个类和接口是实现我们动态代理所必须用到的。首先我们先来看看java的API帮助文档是怎么样对这两个类进行描述的：
```java
InvocationHandler is the interface implemented by the invocation handler of a proxy instance. 

Each proxy instance has an associated invocation handler. When a method is invoked on a proxy instance, the method invocation is encoded and dispatched to the invoke method of its invocation handler.
```
每一个动态代理类都必须要实现InvocationHandler这个接口，并且每个代理类的实例都关联到了一个handler，当我们通过代理对象调用一个方法的时候，这个方法的调用就会被转发为由InvocationHandler这个接口的 invoke 方法来进行调用。我们来看看InvocationHandler这个接口的唯一一个方法 invoke 方法：
```java
Object invoke(Object proxy, Method method, Object[] args) throws Throwable

proxy: 指代我们所代理的那个真实对象
method: 指代的是我们所要调用真实对象的某个方法的Method对象
args: 指代的是调用真实对象某个方法时接受的参数
```
如果不是很明白，等下通过一个实例会对这几个参数进行更深的讲解。

接下来我们来看看Proxy这个类：
```java
Proxy provides static methods for creating dynamic proxy classes and instances, and it is also the superclass of all dynamic proxy classes created by those methods. 
```
Proxy这个类的作用就是用来动态创建一个代理对象的类，它提供了许多的方法，但是我们用的最多的就是 newProxyInstance 这个方法：
```java
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces,  InvocationHandler h)  throws IllegalArgumentException

Returns an instance of a proxy class for the specified interfaces that dispatches method invocations to the specified invocation handler.
```
这个方法的作用就是得到一个动态的代理对象，其接收三个参数，我们来看看这三个参数所代表的含义:
+ **loader**:　　一个ClassLoader对象，定义了由哪个ClassLoader对象来对生成的代理对象进行加载
+ **interfaces**:　　一个Interface对象的数组，表示的是我将要给我需要代理的对象提供一组什么接口，如果我提供了一组接口给它，那么这个代理对象就宣称实现了该接口(多态)，这样我就能调用这组接口中的方法了
+ **h**:　　一个InvocationHandler对象，表示的是当我这个动态代理对象在调用方法的时候，会关联到哪一个InvocationHandler对象上

Proxy.newProxyInstance返回值类型：$Proxy0
```java
Subject subject = (Subject)Proxy.newProxyInstance(handler.getClass().getClassLoader(), realSubject.getClass().getInterfaces(), handler);
```
可能我以为返回的这个代理对象会是Subject类型的对象，或者是InvocationHandler的对象，结果却不是，首先我们解释一下为什么我们这里可以将其转化为Subject类型的对象？原因就是**在newProxyInstance这个方法的第二个参数上，我们给这个代理对象提供了一组什么接口，那么我这个代理对象就会实现了这组接口，这个时候我们当然可以将这个代理对象强制类型转化为这组接口中的任意一个，因为这里的接口是Subject类型，所以就可以将其转化为Subject类型了。**

**同时我们一定要记住，通过 Proxy.newProxyInstance 创建的代理对象是在jvm运行时动态生成的一个对象，它并不是我们的InvocationHandler类型，也不是我们定义的那组接口的类型，而是在运行是动态生成的一个对象，并且命名方式都是这样的形式，以$开头，proxy为中，最后一个数字表示对象的标号。**

java动态代理是利用反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理。

而cglib动态代理是利用asm开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理。

Spring AOP会在两种方式下切换:
1. 如果目标对象实现了接口，默认情况下会采用JDK的动态代理实现AOP 
2. 如果目标对象实现了接口，可以强制使用CGLIB实现AOP 
3. 如果目标对象没有实现了接口，必须采用CGLIB库，spring会自动在JDK动态代理和CGLIB之间转换

利用拦截器(拦截器必须实现InvocationHanlder)加上反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理。是在程序运行的过程中，根据被代理的接口来动态生成代理类的class文件，并加载运行的过程。

之所以只支持实现了接口的类的代理。从原理上讲是因为JVM动态生成的代理类有如下特性：

**继承了Proxy类，实现了代理的接口，最终形式如下（HelloInterface为被代理类实现的接口）：**
```java
public final class $Proxy0 extends Proxy implements HelloInterface{

.......

}
```

然而由于java不能多继承，这里已经继承了Proxy类了，不能再继承其他的类，所以JDK的动态代理不支持对实现类的代理，只支持接口的代理。

从使用上讲，创建代理类时必须传入被代理类实现的接口。


### SpringAOP的实现原理
关于Spring AOP的概念我就不多说了，大家都知道是基于动态代理实现的，就是上面我们说的这些，那么具体是怎么实现的呢？

在代理模式中有三种核心的角色：委托类、代理类、接口，而gclib动态代理中“接口”是非必须的，因此我们关注Spring AOP中 委托类和代理类的实现。

**委托类**

回顾一下Aop的实现代码：需要在实现类上加上@Aspect的注解，还需要通过@Pointcut注解来申明“切点”，即委托类和委托方法的路径。\
有了这些信息就足够获取委托类了。这里充分用到Java反射，先找到包含@Aspect注解的类，然后找到该类下的@Pointcut注解，读取所定义的委托类和委托方法路径，就完全能拿到委托类对象。

**代理类**

因为我们使用的是动态代理，这里的代理类可以被替换成代理方法。同样，我们在@Aspect注解的类中，用@Around、@Before、@After修饰的方法，就是我们想要的代理方法。

**总结**
我们可以通过BeanFactoryPostProcessor的实现类，完成对所有BeanDefinition的扫描，找出我们定义的所有的切面类，然后循环里面的方法，找到切点、以及所有的通知方法，然后根据注解判断通知类型（也就是前置，后置还是环绕），最后解析切点的内容，扫描出所有的目标类。这样就获取了委托类 和 代理方法。

现在委托类 和 代理方法 都有了，我们知道在动态代理模式中，最终的目的是将委托类的方法执行，替换成代理类的方法执行。但是在Spring中，我们是感知不到代理类的，我们在代码中还是调用原委托类的方法，那么Spring框架是如何神不知鬼不觉地将委托类替换成代理类的呢？

**这就涉及到我们之前有关Ioc文章的内容了，在Bean的生命周期中，Bean在初始化前后会执行BeanPostProcessor的方法。可以把它理解成一个增强方法，可以将原始的Bean经过“增强”处理后加载到Ioc容器中。这就是一个天然的代理方法，原始的Bean就是委托类，在此处实现代理方法生成代理类，再将代理类加载进Ioc容器。**





>参考\
>[博客园: java动态代理](https://www.cnblogs.com/xiaoluo501395377/p/3383130.html)\
>[小旋锋：java动态代理详解](https://juejin.cn/post/6844903744954433544)\
>[知乎：jdk动态代理和cglib动态代理的区别](https://zhuanlan.zhihu.com/p/48736954)\
>[思否：SpringAOP实现原理](https://segmentfault.com/a/1190000037596406)


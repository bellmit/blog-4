[知乎：深入理解volatile](https://zhuanlan.zhihu.com/p/73561744)

[外文：happen before guarantee](http://tutorials.jenkov.com/java-concurrency/java-happens-before-guarantee.html)

可见性
可见性指当一个线程修改共享变量的值，其他线程能够立即知道被修改了。Java是利用volatile关键字来提供可见性的。 当变量被volatile修饰时，这个变量被修改后会立刻刷新到主内存，当其它线程需要读取该变量时，会去主内存中读取新值。而普通变量则不能保证这一点。

除了`volatile`关键字之外，`final`和`synchronized`也能实现可见性。

synchronized的原理是，在执行完，进入unlock之前，必须将共享变量同步到主内存中。

final修饰的字段，一旦初始化完成，如果没有对象逸出（指对象为初始化完成就可以被别的线程使用），那么对于其他线程都是可见的。
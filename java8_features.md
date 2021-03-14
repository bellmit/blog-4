[csdn: 尚硅谷java8新特性](https://blog.csdn.net/weixin_45225595/article/details/106203264)

[Java 8 Stream Tutorial](https://winterbe.com/posts/2014/07/31/java8-stream-tutorial-examples/)

[博客园：深入理解Java8中Stream的实现原理](https://www.cnblogs.com/sunshinekevin/p/11576893.html)

Stream上的所有操作分为两类：中间操作和结束操作，中间操作只是一种标记，只有结束操作才会触发实际计算。中间操作又可以分为无状态的(Stateless)和有状态的(Stateful)，无状态中间操作是指元素的处理不受前面元素的影响，而有状态的中间操作必须等到所有元素处理之后才知道最终结果，比如排序是有状态操作，在读取所有元素之前并不能确定排序结果；结束操作又可以分为短路操作和非短路操作，短路操作是指不用处理全部元素就可以返回结果，比如找到第一个满足条件的元素。之所以要进行如此精细的划分，是因为底层对每一种情况的处理方式不同。 为了更好的理解流的中间操作和终端操作，可以通过下面的两段代码来看他们的执行过程。

```java

     Stream.of("one", "two", "three", "four")
        .filter(e -> e.length() > 3)
        .peek(e -> System.out.println("Filtered value: " + e))
        .map(String::toUpperCase)
        .peek(e -> System.out.println("Mapped value: " + e))
        .collect(Collectors.toList());

    //result:
    /**
    *
    *Filtered value: three
    *Mapped value: THREE
    *Filtered value: four
    *Mapped value: FOUR
    */
```
[spi和api的区别](https://www.jianshu.com/p/7e85b8ed00e2)
[stack overflow](https://stackoverflow.com/questions/2954372/difference-between-spi-and-api)
API：Application Programming Interface
SPI：Service Provider Interface

More specifically, for Java libraries, what makes them an API and/or SPI?

the API is the description of classes/interfaces/methods/... that you call and use to achieve a goal

the SPI is the description of classes/interfaces/methods/... that you extend and implement to achieve a goal

Put differently, the API tells you what a specific class/method does for you and the SPI tells you what you must do to conform.

比如java定义了连接数据库的接口【java.sql.Driver】，我们需要实现接口，但是数据库开发商如mysql或者oracle都开发了对应的驱动包，来链接自己的数据库，这些驱动包都实现了java定义的这个接口，实现的功能都是链接数据库，但是mysql的驱动包链接的是mysql，oracle驱动包链接的是oracle，这样我们需要链接mysql就依赖mysql的驱动包，我们需要链接orcale就用oracle的驱动包。SPI流程就是提供接口，实现接口，当客户需要实现某种功能的时候，服务方调用客户方的接口实现类来完成某种功能。比如mysql的驱动（接口实现类）就是SPI服务，当java开发人员需要链接mysql的时候，服务方调用的是mysql开发人员开发的驱动(接口实现类)，来为java开发人员实现mysql的链接功能。

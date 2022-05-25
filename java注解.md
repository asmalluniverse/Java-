### 注解的作用
注解就是在代码中标注一个特殊的标记，这些标记我们可以再编译、类加载、运行时被读取，然后执行相应的处理。

### 常用的注解
1. 常用的注解有spring等框架中提供的；
2. java原生中的注解，例如@Overried和@Deprecated等大多是一些标记的作用。

### 元注解
还有一种叫做元Annotation（元注解）包含@Retention和@Target

1. @Retention标识设置的生命周期，RUNTIME(常用)  SOURCE CLASS ，如果我们设置成source和class级别，就需要继承AbstractProcessor实现process方法进行中自定义注解的处理。
2. @Target表示注解修饰的哪些地方，方法，类，还是成员变量，亦或者是包等等。

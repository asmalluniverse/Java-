### Java异常
指程序运行时(非编译时)所发生的非正常情况或错误，JVM会将出现的错误表示为一个异常并抛出。

### Java异常体系
Throwable：异常基类，java.lang.Throwable，派生Error(错误)和Exception(异常)两个子类。  
Error：表示程序运行期间出现非常严重的错误，且该错误不可恢复，由于是运行在JVM的错误，会导致程序终止。(JVM责任)  
Exception：程序运行本身异常，编译器可捕获。派生出RuntimeException(运行时异常)和CkeckedException(检查异常)  
RuntimeException：运行异常，编译器没有强制对其捕获处理，当出现这种异常，由JVM来处理。(程序责任)    
NullPointerException ---- 空指针异常  
ClassCastException ---- 类转换异常  
ArrayIndexOutOfBoundsException ---- 数组越界异常  
ArrayStoreException ---- 数组存储异常  
BufferOverflowException ---- 缓冲区溢出异常  
ArithmeticException ---- 算术异常  
CheckException：编译期可以检测的异常，如IOException,SQLException，java编译器强制去捕获此类型异常。(Java编译器) 非RuntimeException异常  
ClassNotFoundException：找不到指定class的异常；  
IOException：IO操作异常  

### 常见Error异常   
NoClassDefFoundError：找不到class定义的异常  
StackOverflowError：深递归导致栈被耗尽而抛出的异常  
OutOfMemoryError：内存溢出异常  

### Error&Exception区别  
Error：程序无法处理的系统错误，编译器不做检查  
Exception：程序可以处理的异常，捕获后可恢复  

总结：前者是无法处理的错误，后者是可以处理的异常。  

**ClassNotFoundException**   	                                                                      
从java.lang.Exception继承，是一个Exception类型	                                              
当动态加载Class的时候找不到类会抛出该异常	                                                  
一般在执行Class.forName()、ClassLoader.loadClass()或ClassLoader.findSystemClass()的时候抛出	  

**NoClassDefFoundError**    
从java.lang.Error继承，是一个Error类型  
当编译成功以后执行过程中Class找不到导致抛出该错误
由JVM的运行时系统抛出  

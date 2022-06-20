### 1.threadlocal 理解
他可以提供线程的局部变量，让每个线程都可以通过set/get来对这个局部变量进行操作，不会和其他线程的局部变量进行冲突，实现了线程的数据隔离

### 2.使用场景
1. 项目中有一个时间工具解析类，用于对时间进行格式化，格式化的实现使用的是SimpleDateFormat,但是它不是线程安全的，所以我们就使用ThreadLocal来让每个线程装载自己的SimpleDateFromat对象，已达到在格式化的时候，是线程安全的。

2. Srping提供了事务相关的操作，而我们知道事务得保证一组操作同时成功或失败，这意味着我们一次事务的所有操作需要在同一个数据库连接上，spring就是使用ThreadLocal来实现的。ThreadLocal存储的类型是一个Map，Map中的key是DataSource,value是Connection(Map结构可以解决多数据源的情况)，从而保证同一个线程获取的是一个Connection对象，即一次事务的所有操作在同一个连接上。

3. 内存泄露的问题
ThreadLocal本身并不存储值，它只是作为key来让线程从ThreadLocalMap获取value，ThreadLocaMap该结构本身是定义在Thead中，而ThreadLocal只是作为key，存储set到ThreadLocalMap的变量就是当前线程。  

**疑问点：**为什么不在ThreadLocal下定义Map,key是Thread,value是set进去的值，为啥是ThreadLocal作为key而不是Thread作为key呢？   
如果按照上面的做法，那么所有线程都会去访问ThreadLocal的Map，那么就是存在如下问题：  
1. 一个线程可以拥有多个私有变量如何处理，那是不是就要对set进去的key做一下特殊处理，用于唯一性校验。麻烦不
2. 并发足够大的情况下，意味着所有线程都是操作同一个map。map体积膨胀导致访问性能下降。
3. 何时销毁该Map，map中维护所有线程的私有变量，不知道何时才能销毁。

在回到原理上，我们知道Thread在创建的时候，会有栈的引用指向Thread对象，Thread对象内部维护这ThreadLocalMap的引用。而ThreadLocalMap的key是ThreadLocal，value是传入的Object对象  

	栈			                        堆
ThreadLocal Ref  ----->      ThreadLocal      ----->     Entry(key)    （弱引用）

Thread Ref       ------>      Thread     ----->ThreadLocalMap --->Entry(key,value)  （强引用）

```shell
强引用：只要把一个对象赋给一个引用变量，这个引用变量就是一个强引用，只要对象被置为null时，Gc时会被回收
软引用：需要集成SoftRerence实现，只有在内存不足，软引用指向的对象才会被回收
弱引用：需要继承WeakReference实现，只要发生Gc用引用指向的对象就会被回收
虚引用:它的作用主要是跟踪对象垃圾回收的状态，当回收时通知引用队列做一些通知类的工作。
```
 
**ThreadLocal内存泄露指的是** ：ThreadLocal对象被回收了，ThreadLocalMap中Entry的key没有了指向，但是Entry中仍然有Thread Ref -> Thread -> ThreadLocalMap -> Entry value ->Obeject,这一条引用一直存在，导致了内存泄露  

只有在ThreadLocal没有被回收，那么ThreadLocalMap Entry的key的指向就不会在GC是断开被回收，这样就没有内存泄露这一说法了。那如果Thread销毁了，那么ThreadLocalMap也会被销毁，那么非线程池的环境下，也不会有长期的内存泄漏的问题，而且ThreadLocal实现下还做了保护措施，如果操作ThreadLocal时，发现key为null，会将其清除掉，所以如果在线程池的环境下，如果还会调用ThreadLocal的set get  remover 方法，发现key为null会进行清除，不会存在内存泄露的问题。  

**所以存在长期内存泄露需要满足的条件: **
1. ThreadLocal被回收，
2. 线程被复用
3. 线程复用后不会在调用ThreadLocal的 set get remove 方法 

因此使用ThreadLocal的最佳实践，用完了记得手动remove掉。而ThreadLocal使用弱引用可以预防大多数内存泄露的情况，


### set()方法：
```java
public void set(T value) {
	//获取当前线程
Thread t = Thread.currentThread();
//根据当前线程获取一个ThreadLocalMap(相当于一个容器，用于保存threadlocal的引用 以及对应的值)
	ThreadLocalMap map = getMap(t);
if (map != null)
    map.set(this, value); //map不为空，设置key value
else
    createMap(t, value); //如果map为空，则创建map
}
	
	
void createMap(Thread t, T firstValue) {
//调用了ThreadLocalMap的构造方法并传递了当前threadlocal对象以及对应的 value
t.threadLocals = new ThreadLocalMap(this, firstValue);
}
	
	
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
	//先创建一个新的Entry 
    table = new Entry[INITIAL_CAPACITY];
		// 取threadlocal 对象的hashcode作为下标
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
		//在当前下标下设置key value 
    table[i] = new Entry(firstKey, firstValue);
		//seize 记录 entry 中的元数个数
    size = 1;
		//设置阈值
    setThreshold(INITIAL_CAPACITY);
}
``` 

到此为止，已经分析完 ThreadLocalMap为空的情况，下面分析不为空的情况，分析之前先说明下threadlocalMap中为什么为存在key 为空的情况
因为它使用的时弱引用，可以看到如下代码：

```java
public class ReferenceExample {

    static Object object = new Object();

    public static void main(String[] args) {

        //强引用
        Object strongReference = object;
        object = null;
        System.gc();
        System.out.println(strongReference);

        //弱引用  （threadlocal 中的threadlocalMap中的key可能为空的情况）
        WeakReference<Object> weakReference = new WeakReference<Object>(object);
        System.gc();
        System.out.println(weakReference.get());
    }
}

```
java.lang.Object@2b193f2d  
null  

从输出结果我们可以看出弱引用和强引用的区别，这也能说明threadlocalmap 中key 为空的情况   


```java
private void set(ThreadLocal<?> key, Object value) {
            Entry[] tab = table;
			//获取entry长度
            int len = tab.length;
			//计算hash值作为下标
            int i = key.threadLocalHashCode & (len-1);

			// 从当前计算的i向后循环， 循环进入的条件为 entry 不为null，遍历整个数组
            for (Entry e = tab[i];e != null;e = tab[i = nextIndex(i, len)]) {
			    //获取当前位置的key
                ThreadLocal<?> k = e.get();
				//如果存在key 直接替换，返回
                if (k == key) {
                    e.value = value;
                    return;
                }
				//如果key 为null 调用replaceStaleEntry方法
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
	    //没有找到，那直接创建一个新的entry保存元素，并清理key为null的entry
            tab[i] = new Entry(key, value);
            int sz = ++size;
			//判断是否需要进行扩容操作
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
	

//该方法做了两件事情，1.把当前的value保存到entry数组中  2.清理无效的key 
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;
	    //slotToExpunge 记录当前的下标位置
            int slotToExpunge = staleSlot;
	    //从当前位置向前循环找到数据中最前面一个为null的位置，记录当前的位置
            for (int i = prevIndex(staleSlot, len);(e = tab[i]) != null;i = prevIndex(i, len))
                if (e.get() == null)
                    slotToExpunge = i;

            //从当前位置向后循环
            for (int i = nextIndex(staleSlot, len);(e = tab[i]) != null;i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();

		//如果当前位置对应的key 和需要设置的key相同，则value的值保存到当前位置，并把该位置的老值放到循环起始位置
		
		//这里的意思就是前面的第一个for 循环(i--)往前查找的时候没有找到过期的，只有staleSlot
                    // 这个过期，由于前面过期的对象已经通过交换位置的方式放到index=i上了，
                    // 所以需要清理的位置是i,而不是传过来的staleSlot
                if (k == key) {
                    e.value = value;

                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;

                    // 向前查找没有找到为空的元数，即从当前位置开始进行清理key 为null的元素
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
			//进行清理过期数据
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

                如果当前位置的key为null，且向前没有查找到为空的元素，记录当前位置的下标
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            // 如果key没有找到直接设置key value
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            // 清理key 为null的元素 ，因为如果 slotToExpunge == staleSlot 说明向前和向后查找都没有为null 的元素，不需要清理
	    //如果进入到这里说明 if (k == null && slotToExpunge == staleSlot)走到了这个if条件中，向后循环的过程中找到为null的元素，
	    //所以此时需要从该位置进行清理工作
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }
		
cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);		


		private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.get() == null) {
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }
		//循环Log 2n次数
            } while ( (n >>>= 1) != 0);
            return removed;
        }
		
		
expungeStaleEntry(slotToExpunge)		


        //staleSlot 表示向前循环查到找的第一个为Null 的元素
        private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
``` 

### get()方法
```java
    public T get() {
	    //获取当前线程
        Thread t = Thread.currentThread();
		//根据当前线程获取一个ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
		//这里调用 setInitialValue方法，因为我再初始化threadloca的时候可以直接初始化value。而不是在set的时候进行设置值
        return setInitialValue();
    }	

	//可以看出这里的setInitialVlaue方法和set 方法一致
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }	
```    
    


### 为什么ThreadLocalMap 采用开放地址法来解决哈希冲突?
jdk 中大多数的类都是采用了链地址法来解决hash 冲突，为什么ThreadLocalMap 采用开放地址法来解决哈希冲突呢？首先我们来看看这两种不同的方式

1. 链地址法
这种方法的基本思想是将所有哈希地址为i的元素构成一个称为同义词链的单链表，并将单链表的头指针存在哈希表的第i个单元中，因而查找、插入和删除主要在同义词链中进行。列如对于关键字集合{12,67,56,16,25,37, 22,29,15,47,48,34}，我们用前面同样的12为除数，进行除留余数法
2. 开放地址法
这种方法的基本思想是一旦发生了冲突，就去寻找下一个空的散列地址(这非常重要，源码都是根据这个特性，必须理解这里才能往下走)，只要散列表足够大，空的散列地址总能找到，并将记录存入。

3. 链地址法和开放地址法的优缺点
```shell
开放地址法：
1.	容易产生堆积问题，不适于大规模的数据存储。
2.	散列函数的设计对冲突会有很大的影响，插入时可能会出现多次冲突的现象。
3.	删除的元素是多个冲突元素中的一个，需要对后面的元素作处理，实现较复杂。
链地址法：
1.	处理冲突简单，且无堆积现象，平均查找长度短。
2.	链表中的结点是动态申请的，适合构造表不能确定长度的情况。
3.	删除结点的操作易于实现。只要简单地删去链表上相应的结点即可。
4.	指针需要额外的空间，故当结点规模较小时，开放定址法较为节省空间。
``` 
### ThreadLocalMap 采用开放地址法原因
1. ThreadLocal 中看到一个属性 HASH_INCREMENT = 0x61c88647 ，0x61c88647 是一个神奇的数字，让哈希码能均匀的分布在2的N次方的数组里, 即 Entry[] table，关于这个神奇的数字google 有很多解析，这里就不重复说了
2. ThreadLocal 往往存放的数据量不会特别大（而且key 是弱引用又会被垃圾回收，及时让数据量更小），这个时候开放地址法简单的结构会显得更省空间，同时数组的查询效率也是非常高，加上第一点的保障，冲突概率也低


### set方法总结

1.	首先计算key的hash值作为下标，如果当前位置获取的entry为空，直接创建一个新的entry保存当前元素，map长度加一。
2.	如果当前位置的entry不为空，会进入一个for循环，从当前位置向后循环，循环的结束条件是entry为Null。
3.	循环过程中，如果新保存的key和当前index位置上的key相同，那么直接替换，结束循环
4.	如果当前index位置的key为null，则进行清理工作，会调用replaceStaleEntry方法，清楚无效的key,即threadlocalmap中key为null 的元素。
5.	如果没有找到相同的key向后循环也不存在为null的元素，那么在 循环结束后的i位置new一个entry放入map中

### replaceStateEntry方法总结

该方法主要做两件事情，1，把当前的value保存到entry数组中 2.清理无效的key
slotToExpunge参数为需要清理key为null的下标
1.	从当前位置（key为null的index）向前循环，直到找到为Null的元素，把当前i的位置的值赋值给slotToExpunge变量
2.	从当前位置向后循环，如果当前位置对应的key 和需要设置的key相同，则value的值保存到当前位置，并把该位置的老值放到循环起始位置（交换操作，后面会说为什么做交换），然后进行清理无效的key操作，并返回。
3.	如果向前没有找到为null的key，向后也没有找到相同的key且也没有为Null的key，那么就在null的位置（staleSlot入参的位置）新建一个entry放入到map中

为什么进行交换？
因为在删除无效元素的时候遇到null 就停止寻找了，你前面k==null
//的时候已经设置entry为null了，不移动的话，那么后面的元素就永远访问不了了  


参考博客：  
https://www.jianshu.com/p/dde92ec37bd1   
 https://blog.csdn.net/wanghao112956/article/details/102678591

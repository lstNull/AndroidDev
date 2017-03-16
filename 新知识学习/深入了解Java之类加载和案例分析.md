\r\n摘自http://itfeifei.win/2017/03/14/深入了解Java之类加载和案例分析/?utm_source=gank.io&utm_medium=email
##深入了解Java之类加载和案例分析

* 在讨论JVM内存区域分析之前，先来看一下Java程序具体执行的过程：
* ![过程图](http://i1.piimg.com/567571/0282ddefcac2c362.png)
* Java 程序的执行过程：Java 源代码文件（.Java文件）-> Java Compiler（Java编译器）->Java 字节码文件（.class文件）->类加载器（Class Loader）->Runtime Data Area（运行时数据）-> Execution Engine（执行引擎）。

1. 类加载器（ClassLoader）

### 把Java类的数据从Class文件加载到虚拟机内存中，然后对这部分数据进行验证、准备、解析、初始化，最终形成可以被虚拟机直接使用的Java类型。
#### 学习这个有什么好处呢？
* 1、有助于了解Java虚拟机的执行过程
* 2、程序能动态的控制类加载，比如热部署、热修复等

#### 类加载器（ClassLoader）的一些方法：
| 方法     | 说明    |
| :------------- | :------------- |
| getParent()      | 返回该类加载器的父类加载器      |
| loadClass(String name)    | 加载名称为 name的类      |
| findClass(String name)     | 查找名称为 name的类      |
| findLoadedClass(String name)      | 查找名称为 name的已经被加载过的类     |
| defineClass(String name, byte[] b, int off, int len)     | 把字节数组 b中的内容转换成 Java 类，返回的结果是 java.lang.Class类的实例。这个方法被声明为 final的     |

2. 类加载的分类

* Bootstrap Loader(启动类加载器) ：是用C++语言写的，是虚拟机自身的一部分，主要负责加载Java的核心类；
* Extended Loader(扩展类加载器) ：Extended Loader的父加载器为 Bootstrap Loader，该加载器使用Java写的，它用来加载Java的扩展库；
* AppClass Loader(系统类加载器)：AppClass Loader的父加载器为 Extended Loader，该加载器使用Java写的，它用来加载Java应用的类路径（CLASSPATH）上的类库。

3. 类加载过程

* 寻找jre目录，寻找jvm.dll，并初始化JVM，JVM启动后，运行Bootstrap Loader，该Bootstrap Loader(启动类加载器)自动加载Extended Loader（扩展类加载器）和AppClass Loader（系统类加载器），最后AppClass Loader加载CLASSPATH目录下定义的Class。

4. 双亲委派模型

* 如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把请求委托给父加载器去完成，依次向上，因此，所有的类加载请求最终都应该被传递到顶层的启动类加载器中，只有当父加载器在它的搜索范围中没有找到所需的类时，即无法完成该加载，子加载器才会尝试自己去加载该类。
***********************************************************
代码实现:
```
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        //检查该类是否已经加载过
        Class c = findLoadedClass(name);
        if (c == null) {
            //如果该类没有加载，则进入该分支
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    //当父类的加载器不为空，则通过父类的loadClass来加载该类
                    c = parent.loadClass(name, false);
                } else {
                    //当父类的加载器为空，则调用启动类加载器来加载该类
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                //非空父类的类加载器无法找到相应的类，则抛出异常
            }
            if (c == null) {
                //当所有父类加载器无法加载时，则调用自己findClass方法来加载该类
                long t1 = System.nanoTime();
                c = findClass(name); //用户可通过覆写该方法，来自定义类加载器
                //用于统计类加载器相关的信息
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            //对类进行link操作
            resolveClass(c);
        }
        return c;
    }
}
```

#### “双亲委派模型”有什么作用呢？

* 1、这样保证了每个类都只会加载一次
* 2、维护系统自身类的安全

5. 类加载的三种方式

* 1、通过JVM初始化加载
* 2、通过Class.forName()方法动态加载
* 3、通过ClassLoader.loadClass()方法动态加载

6. 案例分析

* 需要深入了解Java虚拟机内存的理论知识，我们用这个例子简单的来了解一下类的加载和内存分配的一些逻辑：
```
public class TestMain {
     public void TestMethod() {
         Sample testSample = new Sample(" test ");
         testSample.getName();
     }
 }
 public class Sample {
     private String name;
     public Sample(String name) {
         this.name = name;
     }
     public void getName() {
         Log.i("Sample", "name:" + name);
     }
 }
```
#### 过程分析
* public class TestMain

\r\n通过JVM初始化加载；通过Class.forName()方法动态加载；通过ClassLoader.loadClass()方法动态加载通过以上方法加载这个类的时候，会从Bootstrap Loader（启动类加载器），Extended Loader（扩展类加载器）从上往下加载，发现都加载不了，最后由AppClass Loader从CLASSPATH中找到加载TestMain这个类,读取这个文件中的二进制数据,接着对这部分数据进行验证、准备、解析、初始化，最后会把这个类的类信息、常量、静态变量、以及编译器编译后的代码等存入运行时数据区中的方法区。

* public void TestMethod()

\r\nTestMethod 也被放入方法区。

* Sample testSample = new Sample(" test ");

\r\n先看”=”右边的：“new Sample(“ test “)”也是一个类跟上面一样，加载这个类的相关信息到方法区中，然后就是就是要在运行时数据区中的堆进行分配内存，方法区的类信息和堆怎么对应呢？其实堆只是存放了这个Sample实例的一个指向方法区的引用地址testSample，他主要是存放在运行时数据的Java虚拟机栈的栈帧中的局部变量表中”=”其实不是赋值，而是testSample这个变量指向了堆中的Sample实例

* testSample.getName();

\r\ntestSample指向了堆中Sample的实例，然后根据这个实例持有的引用拿到了方法区这个类的相关信息，其中的一个方法getName();接着执行这个方法打印数据 ]

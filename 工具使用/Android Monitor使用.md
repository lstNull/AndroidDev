# 摘自:http://rkhcy.github.io/2017/02/14/Android%20Monitor/?utm_source=gank.io&utm_medium=email

## 最近项目上线，公司定制的机器硬件配置不是很好，需要长时间测试以及性能测试，Monitor能很好的帮我们监控CPU、内存、网络等状况，以便于更好的了解当前项目所存在的问题。

*Android Studio 内置了四种性能监测工具Memory Monitor、Network Monitor、CPU Monitor、GPU Monitor，我们可以使用这些工具监测APP的状态，该文简单介绍下这些工具的使用*

## Memory Monitor
* Memory Monitor工具主要是用来监测APP的内存分配情况，判断是否存在内存泄漏。连接设备，选择好要监测的APP，如图所示：
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/memory1.png)
* A：手动触发GC操作
* B：获取当前的堆栈信息，生成.hprof文件
* C：内存分配追踪工具，生成.alloc文件
* D：已使用内存
* E：剩余可用内存
* 通过与应用交互并在Memory Monitor中观察它是如何影响内存的使用，图表可以为你展示一些潜在的问题:
* 1.频繁的垃圾收集活动使应用运行缓慢。
* 2.应用耗尽内存导致app崩溃.
* 3.潜在的内存泄漏
* 正常情况下，上图中的D区域会随着时间的走势慢慢上升(就算你与APP没有任何交互)，直到E区域被用完，则会触发GC操作，释放内存，周而复始。如果你发现你的应用是静态的，但是E区域的内存很快就被用完了，即频繁的触发GC操作，这时你就应该引起重视，说不定你的代码中就存在着引起内存泄漏的隐患。

### Dump Java Heap
* 使用场景：定位内存泄漏
* 点击上图中的B按钮开始检测APP，此时APP会变得很卡，容易发生ANR，一段时间过后会生成.hprof文件，如下图所示
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/memory2.png)
* 这里的截图是我故意生成的一个能引起内存泄漏的例子，点击上图右上方的Analyzer Tasks按钮，若代码中存在内存泄漏隐患，在其下方会列出可能引起内存泄漏的Activity，如上图右下方的Leaked Activities，之后我们便可以结合左下方Reference Tree中指出的问题分析，如果你有源码的话还可以索引源码(右键->Jump to source)。实例代码如下：
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/memory3.png)
* 多次旋转屏幕，使得内存不断增加就容易引起内存泄漏。
上面的例子比较简单，可以直接通过Memory Monitor工具就能直接看出，在平常的开发中内存泄漏的问题往往没有这么简单，我们可以借助MAT工具分析。

### MAT
* MAT(Memory Analyzer Tool)，一个基于Eclipse的内存分析工具，是一个快速、功能丰富的JAVA heap分析工具，它可以帮助我们查找内存泄漏和减少内存消耗。使用内存分析工具从众多的对象中进行分析，快速的计算出在内存中对象的占用大小，看看是谁阻止了垃圾收集器的回收工作，并可以通过报表直观的查看到可能造成这种结果的对象。
* 在使用MAT工具前，我们需要将.hprof文件转换成标准的.hprof文件才能被识别，在Android Studio中可以通过以下操作转换
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/memory4.png)
* 之后用MAT打开，显示如下
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/memory5.png)
* 点击Histogram按钮
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/memory6.png)
* 如图所示，该图会列出内存中所有的对象的个数即其占用的内存大小，其次，我们可以输入指定的Activity名称来缩小定位范围
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/memory7.png)
* 如图所示，这里列出了MainActivity和其内部类MyThread的对象个数即占用的内存大小，接下来我们选择一个条目右键—>Merge Shortest Paths to GC Roots(查看一个对象到GC Roots是否存在引用链相连接)->Merge Shortest Paths to GC Roots(排除虚引用/弱引用/软引用等等)
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/memory8.png)
* 如上图所示，就是可能存在内存泄漏的地方，具体还是要结合代码分析。

### GC Roots
* 对象存活的判定:
* 当一个对象不会再被使用的时候，我们会说这对象已经死亡。对象何时死亡，写程序的人应当是最清楚的。如果计算机也要弄清楚这件事情，就需要使用一些方法来进行对象存活判定，常见的方法有引用计数（Reference Counting）和有可达性分析（Reachability Analysis）两种。

* 引用计数算法的大致思想是给对象中添加一个引用计数器，每当有一个地方引用它时，计数器值就加1；当引用失效时，计数器值就减1；任何时刻计数器为0的对象就是不可能再被使用的。
* Java语言里面没有选用引用计数算法来管理内存，其中最主要原因是它没有一个优雅的方案解决对象之间相互循环引用的问题：
* 当两个对象互相引用，即使它们都无法被外界使用时，它们的引用计数器也不会为0。

* 可达性算法的基本思路就是通过一系列的称为GC根节点（GC Roots）的对象作为起始点，从这些节点开始进行向下搜索，搜索所走过的路径成为引用链（Reference Chain），当一个对象到GC Roots没有任何引用链相连（用图论的话来说就是从GC Roots到这个对象不可达）时，则证明此对象是不可用的。

* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/memory9.png)
* 如上图，对象object 5、object 6、object 7虽然互相有关联，它们的引用并不为0，但是它们到GC Roots是不可达的，因此它们将会被判定为是可回收的对象。

### Allocation Tracker
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/memory1.png)
* 在内存图中点击C，启动追踪，再次点击停止追踪，随后自动生成一个alloc结尾的文件，这个文件就记录了这次追踪到的所有数据。
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/memory10.png)
* 如上图，我们可以看出这次操作中应用里的各个组件的分配次数与占用大小，若发现这两个数据有异常(分配过多，占用过大)，同样可以索引源码优化(前提是你有)，最下方的视图是以另一种较酷炫的方式呈现，感兴趣的可以具体结合文件操作。

## Network Monitor
* Network Monitor是用于显示app网络请求的状态，频繁的网络请求是耗电的重要原因
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/network_monitor.png)
* 如上图所示，Tx与Rx分别表示上下行的速度。

## GPU Monitor
* GPU Monitor工具可以将进行UI渲染工作所花的时间表现出来，它记录下渲染线程准备以及进行描绘的时间。
* Android系统每隔16ms发出VSYNC信号，触发对UI进行渲染，如果每次渲染都成功，这样就能够达到流畅的画面所需要的60fps，为了能够实现60fps，这意味着程序的大多数操作都必须在16ms内完成。

### Why 60fps?
* 我们通常都会提到60fps与16ms，可是知道为何会是以程序是否达到60fps来作为App性能的衡量标准吗？这是因为人眼与大脑之间的协作无法感知超过60fps的画面更新。

* 12fps大概类似手动快速翻动书籍的帧率，这明显是可以感知到不够顺滑的。24fps使得人眼感知的是连续线性的运动，这其实是归功于运动模糊的效果。24fps是电影胶圈通常使用的帧率，因为这个帧率已经足够支撑大部分电影画面需要表达的内容，同时能够最大的减少费用支出。但是低于30fps是无法顺畅表现绚丽的画面内容的，此时就需要用到60fps来达到想要的效果，当然超过60fps是没有必要的。

* 开发app的性能目标就是保持60fps，这意味着每一帧你只有16ms=1000/60的时间来处理所有的任务。

### Get GPU Trace
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/GPU_monitor.png)
* 如上图所示，就是GPU Monitor的样子，点击上图获取gfxtrace按钮后，出现弹出框让你选择要监测的指定线程
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/gpumonitor1.png)
* 选定后点击Trace
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/gpumonitor2.png)
* 过程中会加载指定的lib,同时手机会弹出Dialog
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/gpumonitor3.png)
* 环境需满足条件(手机root，安装GPU Debugging tools以及相关lib)，若不满足会提示：“Failed to connect to the graphics debugger”，假设这里已满足条件
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/gpumonitor4.png)
* 判断是否在获取trace的依据是随着你的操作，上图的值不断增长，点击stop获得这个过程中生成的gfxtrace。

## GPU Debugger
* GPU Debuger是检查OpenGL ES 2.0或3.1渲染app图形的情况，打开之前生成的gfxtrace
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/gpudebugger1.png)

  渲染上下文:
* 渲染上下文是执行OpenGL ES命令所需的。它用来收集渲染一张图片所需的状态，包括相关的缓存区、阴影、纹理等。许多游戏应用程序只有一个上下文。更高级的应用程序可以使用超过一个上下文。
如果你选择了一个上下文，在GPU Commands 面板中的调用方法将按照调用顺序排列。

  渲染时间线:
* 渲染时间线中的缩略图和GPU Commands 面板下的Frame一一对应，GPU Commands 面板显示了 OpenGL ES 生成每一帧的调用层级。

* 支持双buffer的apps渲染一帧的结尾函数是eglSwapBuffers()方法。

### Framebuffer Pane
* 在GPU Commands面板中选择一帧(生成这一个由多个Draw函数完成，在Draw函数中又有多个openGL ES方法)，Framebuffer 面板显示的内容取决于最后一个方法。
* ![图例](http://7xjvg5.com1.z0.glb.clouddn.com/framebuffer1.png)
* 对于上图红框中各项工具的具体作用可以查看官网的描述，本地测试的时候作用不够明显。
![](http://7xjvg5.com1.z0.glb.clouddn.com/framebuffer2.png)
![](http://7xjvg5.com1.z0.glb.clouddn.com/framebuffer3.png

### Textures Pane
* ![](http://7xjvg5.com1.z0.glb.clouddn.com/Textures%C2%A0Pane1.png)
* ![](http://7xjvg5.com1.z0.glb.clouddn.com/Textures%C2%A0Pane2.png)

### Geometry pane
* ![](http://7xjvg5.com1.z0.glb.clouddn.com/Geometry%C2%A0pane1.png)
* 几何面板的介绍同样本地的操作不够直观，可以直接查看官网介绍
* ![](http://7xjvg5.com1.z0.glb.clouddn.com/geometry2.png)
* ![](http://7xjvg5.com1.z0.glb.clouddn.com/geomery3.png)
* ![](http://7xjvg5.com1.z0.glb.clouddn.com/geomery4.png)

### GPU state pane
* 查看当前GPU的状态
![](http://7xjvg5.com1.z0.glb.clouddn.com/gpustatepane.png)

### Memory pane
* 查看所选方法的值在内存中的保存情况
![](http://7xjvg5.com1.z0.glb.clouddn.com/memorypane.png)

## CPU Monitor
* CPU Monitor可以对代码中的方法进行检测，同样可以生成一个Trace文件
* ![](http://7xjvg5.com1.z0.glb.clouddn.com/cpumonitor1.png)
* 生成的Trace文件如下图
* ![](http://7xjvg5.com1.z0.glb.clouddn.com/cpumonitor2.png)

* 通过方法的调用次数和所花时间来查看，通常判断方法是：
1. 如果方法调用次数不多，但每次调用却需要花费很长的时间的函数，可能会有问题。
2. 如果自身占用时间不长，但调用却非常频繁的函数也可能会有问题。

## Caputures
Android Monitor捕获到的文件都在该路径下
![](http://7xjvg5.com1.z0.glb.clouddn.com/captures1.png)

![](http://7xjvg5.com1.z0.glb.clouddn.com/captures2.png)

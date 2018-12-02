# 02-04、面试题总结-Runloop和Runtime

## Runloop

[深入理解 runloop](https://blog.ibireme.com/2015/05/18/runloop/)

### RunLoop 的概念：

一般来讲，一个线程一次只能执行一个任务，执行完成后线程就会退出。如果我们需要一个机制，让线程能随时处理事件但并不退出，通常的代码逻辑是这样的：
function loop() {
    initialize();
    do {
        var message = get_next_message();
        process_message(message);
    } while (message != quit);
}
这种模型通常被称作 Event Loop。 Event Loop 在很多系统和框架里都有实现，比如 Node.js 的事件处理，比如 Windows 程序的消息循环，再比如 OSX/iOS 里的 RunLoop。实现这种模型的关键点在于：如何管理事件/消息，如何让线程在没有处理消息时休眠以避免资源占用、在有消息到来时立刻被唤醒。

所以，RunLoop 实际上就是一个对象，这个对象管理了其需要处理的事件和消息，并提供了一个入口函数来执行上面 Event Loop 的逻辑。线程执行了这个函数后，就会一直处于这个函数内部 "接受消息->等待->处理" 的循环中，直到这个循环结束（比如传入 quit 的消息），函数返回。

OSX/iOS 系统中，提供了两个这样的对象：NSRunLoop 和 CFRunLoopRef。
CFRunLoopRef 是在 CoreFoundation 框架内的，它提供了纯 C 函数的 API，所有这些 API 都是线程安全的。
NSRunLoop 是基于 CFRunLoopRef 的封装，提供了面向对象的 API，但是这些 API 不是线程安全的。

### RunLoop 与线程的关系

线程和 RunLoop 之间是一一对应的，其关系是保存在一个全局的 Dictionary 里。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。RunLoop 的创建是发生在第一次获取时，RunLoop 的销毁是发生在线程结束时。你只能在一个线程的内部获取其 RunLoop（主线程除外）

总结一下
1、主线程的run loop默认是启动的。
2、对其它线程来说，run loop默认是没有启动的，如果你需要更多的线程交互则可以手动配置和启动，
   如果线程只是去执行一个长时间的已确定的任务则不需要。
3、在任何一个 Cocoa 程序的线程中，都可以通过以下代码来获取到当前线程的 run loop 。
   NSRunLoop *runloop = [NSRunLoop currentRunLoop];

## Runtime 机制

### 什么是 Runtime ？

runtime：指一个程序在运行（或者在被执行）的状态。也就是说，当你打开一个程序使它在电脑上运行的时候，那个程序就是处于运行时刻。在一些编程语言中，把某些可以重用的程序或者实例打包或者重建成为“运行库"。这些实例可以在它们运行的时候被连接或者被任何程序调用。

objective-c中runtime：是一套比较底层的纯C语言API, 属于1个C语言库, 包含了很多底层的C语言API。 在我们平时编写的OC代码中, 程序运行过程时, 其实最终都是转成了runtime的C语言代码。

### runtime 的应用：

1、动态创建一个类(比如KVO的底层实现)
2、动态地为某个类添加属性\方法, 修改属性值\方法
3、遍历一个类的所有成员变量(属性)\所有方法
实质上，以上的是通过相关方法来获取对象或者类的isa指针来实现的。

runtime 相关函数：
1.  增加
增加函数: class_addMethod
增加实例变量: class_addIvar
增加属性: @dynamic标签，或者class_addMethod，因为属性其实就是由getter和setter函数组成
增加Protocol: class_addProtocol (说实话我真不知道动态增加一个protocol有什么用,-_-!!)

2.  获取
获取函数列表及每个函数的信息(函数指针、函数名等等):class_getClassMethod method_getName ...
获取属性列表及每个属性的信息:class_copyPropertyList、property_getName
获取类本身的信息,如类名等：class_getName、class_getInstanceSize
获取变量列表及变量信息：class_copyIvarList
获取变量的值

3.  替换
将实例替换成另一个类：object_setClass
替换类方法的定义：class_replaceMethod

4.  其他常用方法：
交换两个方法的实现：method_exchangeImplementations.
设置一个方法的实现：method_setImplementation.

## OC 消息发送和转发机制

2.消息发送和转发机制，SEL 和 IMP。
答：
消息转发：http://blog.csdn.net/u014410695/article/details/48650965

消息发送

-------------------------------------------

在Objective-C中，使用对象进行方法调用是一个消息发送的过程，
Objective-C采用“动态绑定机制”，所以所要调用的方法直到运行期才能确定。

objc_msgSend函数负责像某对象发送一个消息。定义如下
id objc_msgSend(id self, SEL op, ...)

在OC里面我们调用对象方法［Receiver message］的这种模式，实际是通过调用objc_msgSend（Receiver，message,…）函数来找到方法的实现入口。objc_msgSend 实现原理是通过对象对应的 objc_object 的 isa 找到该类对应的 objc_class 结构体。通过依次遍历 objc_cache，objc_method_list 里面的方法找到方法实际入口，如果没有找到，则跳到父类寻找，以此类推如果最终都没有找到就会发生下面介绍的消息转发机制。


消息转发

-------------------------------------------
当我们像一个对象发送消息［Receiver message］，Receiver 没有实现该消息，即[Receiver respondsToSelector:SEL]返回为NO情况下，其实系统不会立刻出现cash，这时Runtime system会对message进行转发。转发之后，如果该消息依然没有被执行就会出现 Cash！Runtime System为我们提供了三种解决这种给对象发送没有实现消息方案。 
消息转发机制基本上分为三个步骤： 
1. 动态方法解析 
2. 备用接收者 
3. 完整转发 
我们可以通过控制这三个步骤其中一环来解决这一个问题

1. 动态方法解析
对象在接收到未知的消息时，首先会调用所属类的类方法
+resolveInstanceMethod:(实例方法)  或者
+resolveClassMethod:(类方法)。
在这个方法中，我们有机会为该未知消息新增一个”处理方法”“。不过使用该方法的前提是我们已经 实现了该”处理方法”，我们可以在运行时通过class_addMethod函数动态添加未知消息到类里面，让原来没有处理这个方法的类具有了处理这个方法的能力。

2. 备用接收者
如果在动态方法解析无法处理消息，则Runtime会继续调以下方法：
- (id)forwardingTargetForSelector:(SEL)aSelector
如果一个对象实现了这个方法，并返回一个非nil的对象，则返回的对象会作为消息的新接收者，且消息会被分发到这个对象。当然这个对象不能是self自身，否则就是出现无限循环。当然，如果我们没有指定相应的对象来处理aSelector，则应该调用父类的实现来返回结果。 
使用这个方法通常是在对象内部，可能还有一系列其它对象能处理该消息，我们便可借这些对象来处理消息并返回，这样在对象外部看来，还是由该对象亲自处理了这一消息。 

3. 完整转发
如果在上一步还不能处理未知消息，则唯一能做的就是启用完整的消息转发机制了。 
此时会调用以下方法：
`- (void)forwardInvocation:(NSInvocation *)anInvocation`
由于NSInvocation的初始化需要有一个方法签名NSMethodSignature，所以我们还需要实现下面的方法，该方法的返回值用于初始化NSInvocation的。
`-(NSMethodSignature*)methodSignatureForSelector:(SEL)aSelector`


Runtime中方法的动态绑定让我们写代码时更具灵活性，如我们可以把消息转发给我们想要的对象，或者随意交换一个方法的实现等。不过灵活性的提 升也带来了性能上的一些损耗。毕竟我们需要去查找方法的实现，而不像函数调用来得那么直接。当然，方法的缓存一定程度上解决了这一问题。

特别是当我们需要在一个循环内频繁地调用一个特定的方法时，通过这种直接调用IMP减少查找过程的方式可以提高程序的性能。NSObject类提供了methodForSelector:方法，让我们可以获取到方法的指针，然后通过这个指针来调用实现代码。我们需要将methodForSelector:返回的指针转换为合适的函数类型，函数参数和返回值都需要匹配上。


## OC方法的IMP

一、什么是IMP
IMP 是 ”implementation” 的缩写，它是 Objetive-C 方法(method)实现代码块的地址，可像C函数一样直接调用。通常情况下我们是通过 [object method:parameter] 或 objc_msgSend() 的方式向对象发送消息，然后 Objective-C 运行时(Objective-C runtime)寻找匹配此消息的IMP，然后调用它；但有些时候我们希望获取到IMP进行直接调用。

二、Objetive-C中的Method结构
在Objecitve-C中，在类中对每一个方法有一个在运行时构建的数据结构，在Objective-C 2.0中，此结构对用户不可见，但仍在内部存在。
struct objc_method
{
      SEL method_name;
      char * method_types;
      IMP method_imp;
};
typedef objc_method Method;
每个方法有3个属性
1、方法名：方法名为此方法的签名，有着相同函数名和参数名的方法有着相同的方法名。
2、方法类型：方法类型描述了参数的类型。
3、IMP:  IMP即函数指针，为方法具体实现代码块的地址，可像普通C函数调用一样使用IMP。

由于Method的内部结构不可见，所以不能通过method->method_name的方式访问其内部属性，只能Objective-C运行时提供的函数获取。
SEL method_getName(Method method);
IMP method_getImplementation(Method method);
const char * ivar_getTypeEncoding(Ivar ivar);

三、IMP 可以实现效果：Method Swizzling

Method Swizzling：
![](http://oy7b0gogl.bkt.clouddn.com/BDF85D9C-3596-4AEC-BE91-44B4E92FB4B9.png)


IMP：
1、有返回值的 IMP
![](http://oy7b0gogl.bkt.clouddn.com/228035BD-80A4-440D-B3AC-9D7DF1DADD6C.png)


还有一个地方我们需要注意，如果这样直接调用IMP的话就会发生经典的EXC_BAD_ACCESS错误，我们定义的IMP指针是一个有返回值的类型，而其实我们获取的viewDidLoad这个方法是没有返回值的，所以我们需要新定义一个和IMP相同类型的函数指针比如VIMP，把他的返回值定位Void，这样如果你修改的方法有返回值就用IMP，没有返回值就用VIMP。

2、没有返回值的 IMP
![](http://oy7b0gogl.bkt.clouddn.com/8A433F6E-7D18-4150-9661-A6E3CB894308.png)
![](http://oy7b0gogl.bkt.clouddn.com/BA6A944E-FF49-4991-8377-A36FC1CAAA39.png)


## OC isa 指针

一、isa 指针
要认识什么是 isa 指针，我们得先明确一点：在 Objective-C 中，任何类的定义都是对象。
类和类的实例（对象）没有任何本质上的区别。任何对象都有isa指针。

打开文件objc.h 能看到类的定义：
![](http://oy7b0gogl.bkt.clouddn.com/1A961A37-1FE7-40C0-94F3-114514642BB0.jpg)

可以看出:
Class 是一个 objc_class 结构类型的指针，id 是一个 objc_object 结构类型的指针.


我们再来看看 objc_class 的定义：
![](http://oy7b0gogl.bkt.clouddn.com/FB7D642E-0B58-4820-86AA-6CD6FD5BBE21.jpg)

isa：是一个 Class 类型的指针
每个实例对象有个 isa 的指针，他指向对象的类，而 Class 里也有个 isa 的指针，指向 meteClass(元类)。
元类保存了类方法的列表。当类方法被调用时，先会从本身查找类方法的实现，如果没有，元类会向他父类查找该方法。同时注意的是：元类（meteClass）也是类，它也是对象。元类也有isa指针,它的isa指针最终指向的是一个根元类(root meteClass)。根元类的 isa 指针指向本身，这样形成了一个封闭的内循环。

super_class：父类，如果该类已经是最顶层的根类,那么它为NULL。
version：类的版本信息,默认为0
info：供运行期使用的一些位标识。
instance_size：该类的实例变量大小
ivars：成员变量的数组


再来看看各个类实例变量的继承关系：
![](http://oy7b0gogl.bkt.clouddn.com/7D9A301B-F827-4826-9CCC-6E734136E9B4.png)


总结：
1、每一个对象本质上都是一个类的实例。其中类定义了成员变量和成员方法的列表。对象通过对象的isa指针指向类。
2、每一个类本质上都是一个对象，类其实是元类（meteClass）的实例。元类定义了类方法的列表。类通过类的isa指针指向元类。
3、所有的元类最终继承一个根元类，根元类isa指针指向本身，形成一个封闭的内循环。

## Runloop & Runtime 面试题

### 什么是 runloop？

百度大神孙源：[深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/#base)

所以，RunLoop 实际上就是一个对象，这个对象管理了其需要处理的事件和消息，并提供了一个入口函数来执行上面 Event Loop 的逻辑。线程执行了这个函数后，就会一直处于这个函数内部 “接受消息->等待->处理” 的循环中，直到这个循环结束（比如传入 quit 的消息），函数返回。

### 什么是 Runtime？

### 是否能给编译后的类添加属性？运行时的呢？（runtime基础知识）

不可以在编译后的类添加属性，因为已经确定了实例变量数组的大小以及指针的偏移
运行时是可以的，但是也要在申请内存地址之前

### 消息发送和转发机制，SEL和IMP。


### runloop的基础知识。经典题目：timer准不准？mode都有几种？scrollView滑动的时候timer失效吗？怎么处理？

[RunLoop 总结：RunLoop的应用场景（一）](https://blog.csdn.net/u011619283/article/details/53433243)

这个也是偏基础的。timer不是一定准的，因为受到runloop的影响。

Mode一共5种，API提供给用户的两种，DefaultMode和CommonMode。UIInitializationRunLoopMode这个mode在第一次启动后就不用了。

苹果文档中提到的 Mode 有五个，分别是：

1. NSDefaultRunLoopMode 这是大多数操作中使用的模式。
2. NSConnectionReplyMode	该模式用来监控NSConnection对象。你通常不需要在你的代码中使用该模式。
3. NSModalPanelRunLoopMode	Cocoa使用该模式来标识用于modal panel（模态面板）的事件。
4. NSEventTracking（UITrackingRunLoopMode） Cocoa使用该模式来处理用户界面相关的事件。
5. NSRunLoopCommonModes 这是一组可配置的通用模式。将input sources与该模式关联则同时也将input sources与该组中的其它模式进行了关联。对于Cocoa应用，该模式缺省的包含了default，modal以及event tracking模式。

iOS 中公开暴露出来的只有 NSDefaultRunLoopMode 和 NSRunLoopCommonModes。 NSRunLoopCommonModes 实际上是一个 Mode 的集合，默认包括 NSDefaultRunLoopMode 和 NSEventTrackingRunLoopMode。

滑动的时候是失效的，解决方法就是讲timer加入CommonMode。

### 如果要使用 runtime 添加实例属性，往里传入的参数都有哪些？

这个只要思考我们@property的时候都有啥就明白了。
：名称，类型，参数权限（readwrite啊，nonatomic啊，strong啊之类的）和持有这个变量的类的类名。

面试官提醒：如果要更严谨一点，加个identifier。
这个可以详细看runtime的associate那块。

### 谈谈你对 Objective-C 的动态绑定理解？ 

[深入Objective-C的动态特性](https://onevcat.com/2012/04/objective-c-runtime/)

### 我们说的 oc 是动态运行时语言是什么意思?

主要是将数据类型的确定由编译时，推迟到了运行时。这个问题其实浅涉及到两个概念，运行时和多态。

简单来说：
运行时机制：使我们直到运行时才去决定一个对象的类别，以及调用该类别对象指定方法。
多态：不同对象以自己的方式响应相同的消息的能力叫做多态。
意思就是：假设生物类(life)都用有一个相同的方法-eat；那人类属于生物，猪也属于生物，都继承了life后，实现各自的eat，但是调用是我们只需调用各自的eat方法。也就是不同的对象以自己的方式响应了相同的消息(响应了 eat 这个选择器)。

## Timer

[深入学习iOS定时器](https://www.jianshu.com/p/10c3df0dbfef)

#### 定时器的几个类方法底层分别是怎么实现的（[NSTimer timerWithTimeInterval:repeats: block:]等）

>在介绍RunLoop时已经提到过:NSTimer 其实就是 CFRunLoopTimerRef。他们之间是 toll-free bridged 的。一个 NSTimer 注册到 RunLoop 后，RunLoop 会为其重复的时间点注册好事件。例如 10:00, 10:10, 10:20 这几个时间点。RunLoop为了节省资源，并不会在非常准确的时间点回调这个Timer。Timer 有个属性叫做 Tolerance (宽容度)，标示了当时间点到后，容许有多少最大误差。
如果某个时间点被错过了，例如执行了一个很长的任务，则那个时间点的回调也会跳过去，不会延后执行。就比如等公交，如果 10:10 时我忙着玩手机错过了那个点的公交，那我只能等 10:20 这一趟了。

#### 为什么 timer 加入 CommonMode 就能解决？

CommonMode其实是一个占位Mode，把timer加入CommonMode后能同时支持DefaultMode和TrackingMode，这样就能避免scrollView对timer的影响了。

#### NSTimer-->runloop-->runloop mode


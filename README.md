
ANR问题总结

什么是ANR
ANR，是“Application Not Responding”的缩写，即“应用程序无响应”。

与Java Crash或者Native Crash不同，ANR并不会导致程序崩溃，如果用户愿意等待，大多数ANR在一段时间后都是可以恢复的。但对于用户而言，打开一个窗口就要黑屏8秒，或者按下一个按钮后10秒程序没有任何响应显然是不可接受的。为了便于开发者Debug自己程序中响应迟缓的部分，Android提供了ANR机制。ActivityManagerService(简称 AMS)和
WindowManagerService(简称 WMS)会监测应用程序的响应时间，如果应用程序主线程(即 UI 线程)在超时时间内对输入事件没有处理完毕，或者对特定操作没有执行完毕，就会出现 ANR。



二、出现ANR的场景



发生ANR的三个源头：



主线程：只有应用程序进程的主线程响应超时才会产生ANR；

②超时时间：产生ANR的上下文不同，超时时间也会不同，但只要在这个时间上限内没有响应就会ANR；

输入事件/特定操作：手动或者系统执行的动作
输入事件包括下面三类：
按键事件分发超时(Key Dispatch Timeout)。
Android默认是5秒。输入事件是指按键、触屏等设备输入事件，华为修改为8s
广播事件接收超时(Broadcast Timeout)
Broadcast在10秒内没有执行完成。主线程在执行BroadcastReceiver的onReceive函数时10秒内没有执行完毕，华为修改为20s
③服务超时(Service Timeout)


Service 20秒内没有执行完成。主线程在执行Service的各个生命周期函数时20秒内没有执行完毕。



但是，不是所有的ANR都会给用户有提示框，只有当手机与用户有互动的时候才会弹出提示框。



 



三、ANR log 结构




 
  
  文件名称


  
  
  说明


  
 
 
  
  system_app_anr


  
  
  anr时各个进程的CPU占比、进程堆栈；


  
 
 
  
  event log、kernel log、main log、radio log、system log；


  
  
  启动应用/界面log，kernel log，APP层log，射频log，framework log


  
 
 
  
  devinfo


  
  
  手机的IMEI/SN号；


  
 
 
  
  dumpsys


  
  
  系统当前状态文件，包含内存情况、CPU频率等；


  
 
 
  
  finger


  
  
  明确指出当前对应system_app_anr压缩包名；有时一份log上来会有很多 system_app_anr，通过finger文件可以确定提单的是指哪一个anr文件；


  
 
 
  
  trace


  
  
  包含进程堆栈，一般system_app_anr 已经包含trace文件


  
 



 ANR分析流程：





四、线程状态



Android对标准POSIX本地线程状态进行了进一步的封装，使之可以更准确地描述虚拟机内部的状态。参见下表：




 
  
  线程状态


  
  
  含义


  
 
 
  
  SUSPENDED


  
  
  线程暂停，可能是由于输出Trace、GC或debug被暂停


  
 
 
  
  NATIVE


  
  
  正在执行JNI本地函数


  
 
 
  
  MONITOR


  
  
  线程阻塞，等待获取对象锁


  
 
 
  
  WAIT


  
  
  执行了无限等待的wait函数


  
 
 
  
  TIMED_WAIT


  
  
  执行了带有超时参数的wait、sleep或join函数


  
 
 
  
  VMWAIT


  
  
  正在等待VM资源


  
 
 
  
  RUNNING/RUNNABLE


  
  
  线程可运行或正在运行


  
 
 
  
  INITALIZING


  
  
  新建，正在初始化，为其分配资源


  
 
 
  
  STARTING


  
  
  新建，正在启动


  
 
 
  
  ZOMBIE


  
  
  线程死亡，终止运行，等待父线程回收它


  
 
 
  
  UNKNOWN


  
  
  未知状态


  
 



 



五、ANR原因



造成ANR的原因可以分成两大类，系统原因与应用原因。



系统原因是指在手机开发过程中，由于Kernel、FrameWork、驱动存在问题，导致系统工作不稳定，最终在应用层表现出来的ANR。



应用原因是指应用程序主线程死锁、阻塞或者性能低下导致ANR。应用自身为避免发生ANR，应当在程序开发中注意避免将耗时的操作放在主线程，耗时操作包括：



1、数据库操作。 



2、初始化的数据和控件太多。



3、频繁的创建线程或者其它大对象。



4、加载过大数据和图片。



5、对大数据排序和循环操作。



6、过多的广播和滥用广播。



7、大对象的传递和共享。



8、访问网络。



9、锁住主线程。



 



六、分析技巧



一、主线程执行高耗时操作



ANR最简单的一种情况，主线程执行了耗时操作，系统指定时间内未返回触发ANR。解决方案也很多：



1、  设计上保证主线程中不会执行过于耗时的操作



2、   优化代码尽量避免高耗时



3、  将无法避免的耗时操作（如I/O、外部网络端数据的下载、大量的计算）移动到独立线程中（需要主要线程间的同步，以及考虑是否要采用进度条等UI给予用户提示



4、  处理过程发生了某种异常导致陷入死循环或者其它导致不能正常返回的情况



打开对应的ANR或者trace.txt日志，分析主线程的调用堆栈，对于这类问题基本都可以明确的看到耗时操作，如寻找是不是可能出现死循环，是否卡在I/O操作中等等可能引起长时间执行而不返回的情况。



二、条件等待不满足



本质上是前一种情况的变种，虽然将耗时操作移动到了独立线程中，但可能由于逻辑上的原因，必须等待这些耗时操作的完成（如下一步的计算以来前一步的计算结果等），解决方案依然同前述情况一致。



三、死锁



作为多线程的环境的顽疾，可以轻松的导致程序假死，对于应用来说就是ANR，但是好在死锁基本上都会完美的反应到对应的ANR日志中，分析起来也不算太困难，困难的主要是在复杂的流程中如何去解决（如Android的关键服务WindowManagerService、ActivityManagerService）。



死锁类问题都可以明显看到被阻塞(Blocked)的线程，其中需要等待的锁，以及持有这个锁的线程id都会明确的打印出来。



四、跨进程调用



跨进程调用引起的ANR很类似于最先介绍的“主线程执行高耗时操作”，只是耗时操作从自身进程换到了其它进程中，可能由于服务进程中执行的操作就很耗时、或者服务进程繁忙来不及处理又或是服务进程自身陷入了死锁等情况导致同步跨进程调用一直未能返回，引发ANR。



简易快速分析方法——利用一般情况下服务端的Binder派生类方法名和代理类的方法名一致这一点，直接在服务端进程去搜同类名的方法，例如先前的例子中是MediaPlayer::reset()方法，那么直接搜索reset()方法，即可搜索到到对应的调用堆栈，当然有可能不只一处，所以搜到后还需要继续看代码确认下是否对应的服务端实现，确认后就和一般的ANR问题分析解决方法一致了。




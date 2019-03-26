# Android-崩溃

## 崩溃现场采集的信息

### 崩溃信息

- 进程名、线程名，崩溃的进程是前台进程、后台进程还是UI线程

- 崩溃堆栈和类型（Java崩溃、Native崩溃、ANR）。注意查看堆栈栈顶，看看崩溃在系统代码里还是在自己代码里。还可能需要关键线程的日志，比如下面这段崩溃，除了崩溃现场的堆栈外，也希望知道主线程当前的调用栈。

  ```java
  Process Name: 'com.sample.crash'
  Thread Name: 'MyThread'
  
  java.lang.NullPointerException
      at ...TestsActivity.crashInJava(TestsActivity.java:275)
  ```

### 系统信息

- logcat，包括应用、系统的运行日志。系统的event logcat会记录App运行时的一些情况，记录在/system/etc/event-log-tags中。以下为常见日志表示的一些原因。

  ```Java
  system logcat:
  10-25 17:13:47.788 21430 21430 D dalvikvm: Trying to load lib ... 
  event logcat:
  10-25 17:13:47.788 21430 21430 I am_on_resume_called: 生命周期
  10-25 17:13:47.788 21430 21430 I am_low_memory: 系统内存不足
  10-25 17:13:47.788 21430 21430 I am_destroy_activity: 销毁 Activty
  10-25 17:13:47.888 21430 21430 I am_anr: ANR 以及原因
  10-25 17:13:47.888 21430 21430 I am_kill: APP 被杀以及原因
  ```

  

- 机型、系统、厂商、CPU、ABI、Linux 版本等

- 设备状态（是否root、是否模拟器）

### 内存信息

- 系统剩余内存。可直接读取/proc/meminfo，当系统可用内存很小（小于MemTotal 10%）时，OOM、大量GC、系统频繁自杀拉起等问题会很容易出现。

- 应用使用内存（Java内存、RSS（Resident Set Size）、PSS（Proportional Set Size））。PSS和RSS通过/proc/self/smap计算，可进一步得到apk、dex、so等更详细的分类统计。

- 虚拟内存。可通过/proc/self/status得到，通过/proc/self/maps得到具体分布情况。OOM、tgkill等问题是虚拟内存不足导致的。

  ```Java
  Name:     com.sample.name   // 进程名
  FDSize:   800               // 当前进程申请的文件句柄个数
  VmPeak:   3004628 kB        // 当前进程的虚拟内存峰值大小
  VmSize:   2997032 kB        // 当前进程的虚拟内存大小
  Threads:  600               // 当前进程包含的线程个数
  ```

  32位进程，32位CPU，虚拟内存达到3GB可能会引起内存申请失败的问题。

  如果是64位CPU，虚拟内存一般在3~4GB之间。

### 资源信息

应用崩溃可能由于资源泄漏导致的。

- 文件句柄fd。通过/proc/self/limits获得文件句柄限制，一般单个进程允许打开的最大文件句柄数为1024。如果超过800个就比较危险，需要将所有fd及对应文件名输出到日志，进一步排查是否出现了有文件或者线程的限制。（TODO: 如何输出？）

  ```Java
  opened files count 812:
  0 -> /dev/null
  1 -> /dev/log/main4 
  2 -> /dev/binder
  3 -> /data/data/com.crash.sample/files/test.config
  ...
  ```

  

- 线程数。当前线程数可通过/proc/self/status得到，过多的线程会对虚拟内存和句柄带来压力。如果线程数超过400个会比较危险，需要将线程id及对应线程名输出到日志进行排查。

  ```Java
   threads count 412:               
   1820 com.sample.crashsdk                         
   1844 ReferenceQueueD                                             
   1869 FinalizerDaemon   
   ...  
  ```

- JNI。可通过DumpReferenceTables统计JNI的引用表，进一步分析JNI泄露问题。

### 应用信息

- 崩溃场景：崩溃发生在哪个Actiivty或Fragment，发生在哪个业务中。
- 关键操作路径。注意是记录关键的用户操作路径。
- 其他自定义信息。根据各自应用关心的重点进行打点。

### 其他

- 磁盘空间

- 电量

- 网络信息

  .....

数据采集时注意用户隐私、加密、和脱敏。

## 崩溃分析

### 确定重点

- 确认严重程度：优先解决Top崩溃或对业务有重大影响的崩溃。对未来需求预判，可能被干掉的功能可不处理。

- 关注崩溃基本信息。

  - Java崩溃
  - Native崩溃：观察signal、code、fault addr等内容，及崩溃时Java的堆栈。[signal的含义](https://www.mkssoftware.com/docs/man5/siginfo_t.5.asp "Title")。比较常见的是有 SIGSEGV （由于空指针、非法指针造成）和 SIGABRT（因为ANR和调用abort() 退出所导致）。
  - ANR。先看主线程堆栈（是否因为锁等待导致），再看ANR日志中iowait、CPU、GC、system server等信息，进一步确定是IO问题、C
  - PU竞争问题还是大量GC导致卡死。

- Logcat

  - 特别留意Warning和error的日志
  - 观察系统行为和设备状态（am_anr、am_kill等）
  - 当从崩溃日志无法看出问题原因或得不到有用信息时，建议查看相同崩溃点下的更多崩溃日志。

- 各个资源情况

  结合崩溃基本信息，查看是否内存信息和资源信息有关。内存和线程相关的信息都要特别注意。

### 查找共性

查找某一类崩溃是否存在共性（机型、系统、ROM、厂商、ABI等，或者不同现象下的相同条件下才出现的崩溃）。

### 尝试复现

在稳定复现的路径上，可以增加日志或debug、GDB等各种手段和工具进行分析。

## 疑难案例分析-系统崩溃

### 查找可能的原因

确定系统版本、是否某个厂商ROM的问题，找出一些怀疑的点。

### 尝试规避

查看可疑的代码调用（不恰当的API，或更换其他实现方式）

### Hook解决

Java Hook和Native Hook。

- 案例一：BadTokenException

一个Toast相关的系统崩溃，只在Android 7.0中，Toast显示时窗口Token已经无效。

Android 7.0下的崩溃堆栈：

```Java
android.view.WindowManager$BadTokenException: 
	at android.view.ViewRootImpl.setView(ViewRootImpl.java)
	at android.view.WindowManagerGlobal.addView(WindowManagerGlobal.java)
	at android.view.WindowManagerImpl.addView(WindowManagerImpl.java4)
	at android.widget.Toast$TN.handleShow(Toast.java)
```

Android 8.0的源码：

```Java
try {
  mWM.addView(mView, mParams);
  trySendAccessibilityEvent();
} catch (WindowManager.BadTokenException e) {
  /* ignore */
}
```

关键在于寻找Hook点，Toast里有个叫mTN的变量，类型为Handler，代理它就可捕获。



- 案例二：finalize的TimeoutExecption

另外，对于系统的另一种的TimeoutExecption崩溃，可参照[关于解决"TimeoutExceptiion"的方案](https://github.com/AndroidAdvanceWithGeektime/Chapter02)

```Java
java.util.concurrent.TimeoutException: 
         android.os.BinderProxy.finalize() timed out after 10 seconds
at android.os.BinderProxy.destroy(Native Method)
at android.os.BinderProxy.finalize(Binder.java:459)
```

## 参考文章

[极客时间-张绍文-崩溃优化（下）](https://time.geekbang.org/column/article/70966)


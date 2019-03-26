# 可见性、原子性、有序性问题

为了合理利用CPU的性能，计算机做了如下优化：

- CPU增加了缓存，以均衡与内存的速度差异
- 操作系统增加了进程、线程，分时复用CPU，均衡CPU与I/O设备的速度差异
- 编译程序优化指令执行次序，使缓存更合理地利用

## 缓存导致的可见性问题

可见性：一个线程对共享变量的修改，另外一个线程能立刻看到。

- 单核CPU时代，所有线程操作同一个CPU缓存，因此一个线程对缓存的写，对另一个线程来说一定是可见的。

![image-20190327011349355](https://ws4.sinaimg.cn/large/006tKfTcgy1g1gp76r8wjj30gz0ck77m.jpg)

- 多核CPU时代，每个CPU都有自己的缓存，多个线程在不同CPU上运行时，操作的是不同的CPU缓存。

![image-20190327011515015](https://ws4.sinaimg.cn/large/006tKfTcgy1g1gp8o236lj30ez0cg788.jpg)

## 线程切换带来的原子性问题

原子性：一个或者多个操作在 CPU 执行的过程中不被中断的特性

如执行count += 1，需要三条CPU指令，CPU指令能保证的原子操作是CPU指令级别的。

- 将变量count从内存加载到CPU寄存器
- 在寄存器中执行+1
- 将结果写入内存（由于缓存机制，这里写入的内存可能是CPU缓存而不是主内存）

![image-20190327012453077](https://ws1.sinaimg.cn/large/006tKfTcgy1g1gpipanx2j30i60cuaer.jpg)

若按图中的执行顺序执行，count的最终结果是1而不是我们预想的2。

## 编译优化带来的有序性问题

有序性：程序按照代码的先后顺序执行。编译器为了优化性能，有时会改变程序中语句的先后顺序。

如Java中的双重检查单例：

```Java
public class Singleton {
  static Singleton instance;
  static Singleton getInstance(){
    if (instance == null) {
      synchronized(Singleton.class) {
        if (instance == null)
          instance = new Singleton();
        }
    }
    return instance;
  }
}

```

编译器会对代码进行编译优化，因此在执行new操作时，实际执行路径为：

- 分配一块内存M
- 将M的地址赋值给instance变量
- 最后在内存M上初始化Singleton对象

因此，当线程A执行了getInstance()方法，在执行完指令2时线程切换到B。此时线程B也执行getInstance() 方法，B线程在执行第一个判断时发现instance != null ，所以直接return instance。但是instance实际上是没有初始化过的，这个时候访问instance的成员变量就可能触发空指针异常。

![image-20190327013725395](https://ws3.sinaimg.cn/large/006tKfTcgy1g1gpvq6gu9j30kg0d5jtd.jpg)


# Java如何解决可见性和有序性问题

- 导致可见性的原因是缓存  —> 按需禁用缓存
- 导致有序性的原因的编译优化 —> 按需禁用编译优化

禁用缓存和编译优化的具体方法为：volatile、synchronized、final以及Happens-Before规则。

## Happens-Before规则

- 定义：前面一个操作对后续操作是可见的。Happens-Before约束了编译器的优化行为，虽然允许编译器优化，但要求编译器优化后一定遵守Happens-Before原则。（下面Happens-Before = HB）

```Java
class VolatileExample {
  int x = 0;
  volatile boolean v = false;
  public void writer() {
    x = 42;
    v = true;
  }
  public void reader() {
    if (v == true) {
      // 这里 x 会是多少呢？
    }
  }
}

```

假设在线程A运行writer方法，在线程B运行reader方法，上述代码在Java 1.5前运行，x可能是0，可能是42，在Java 1.5版本后运行，就是42。

### 程序顺序性原则

在一个线程中，按程序顺序，前面的操作HB于后续的任意操作。即：单线程中，程序前面对某个变量的修改一定对后续操作可见。

### volatile变量规则

对一个volatile变量的写操作，HB于后续对这个变量的读操作。

### 传递性

如果A HB于 B，B HB于C，则A HB于C。

结合volatile变量规则对示例代码进行分析：

1、 x=42 HB于 v=true（程序顺序性原则）

2、写变量v=true HB于 读变量v==true （volatile变量规则）

因此，x=42 HB于 读变量v==true。也就是说，在线程B能看到 x=42。这是Java 1.5版本对volatile语义的增强。

### 管程中锁的规则

一个锁的解锁HB于后续对这个锁的加锁。在Java中，synchronized就是对管程的实现。在同步块的加锁和释放锁都是编译器实现的。

### 线程start() 规则

线程A启动子线程B后，子线程B能看到主线程在启动线程B前的操作。

### 线程join() 规则

线程A等待子线程B完成（主线程A通过调用子线程B的join() 方法实现），当子线程B完成后（即主线程A中B线程的join方法返回），主线程能看到子线程的操作。所谓看到，即共享变量的操作。

即 线程A中调用了线程B的join方法，线程B中的任意操作HB于该join操作的返回。

## final

final修饰的变量，说明的是这个变量初始化后就不会变化，可以不断优化。但在Java 1.5前构造函数的错误重排可能会看到final变量的值会变化，在Java 1.5后对final类型的变量重排进行了约束。现在只需保证构造函数没有逸出就不会出问题。

逸出的意思是指，在一个类的构造方法中对外暴露了this指针，此时this指向的内存空间可能并未初始化，有可能在类外部通过this读取对象中的成员变量未被赋值。导致这个问题出现的原因是由于编译器会进行编译优化，在执行构造方法时this所指向的内存空间还没有完全初始化，但是这时候却在这个对象外部使用了这个未初始化完成的对象的成员变量。如：

```Java
final int x;
// 错误的构造函数
public FinalFieldExample() { 
  x = 3;
  y = 4;
  // 此处就是讲 this 逸出，
  global.obj = this;
}

```

FinalFieldExample是构造方法，编译器进行编译优化时，会先执行global.obj = this; 然后在对象外部使用了这个this去读取x的值(global.obj.x)，由于编译优化，此时的x并未赋值为3，因此这个时候读取的x的值仍为0。
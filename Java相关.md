- Java内存模型

  并发场景中，缓存导致可见性的问题，线程切换导致原子性的问题，编译优化导致有序性的问题常常违背直觉。解决可见性和有序性导致的问题引出了Java内存模型

  volatile不是Java的特产，C里面就有，原始意义是禁用CPU缓存。声明一个变量为volatile，是告诉编译器对这个变量的读写不能使用CPU缓存，必须从内存中读取或写入。

  ```java
  class VolatileExample {
    int x = 0;
    volatile boolean v = false;
    public void writer() {
      x = 42;
      v = true;
    }
    public void reader() {
      if (v == true) {
        // 这里x会是多少呢？
      }
    }
  }
  ```

  A线程调writer，B线程调reader，此时x是多少？Java1.5以前的版本可能是0，可能是42；1.5之后的版本则是42。是0的原因是CPU缓存导致的可见性问题，1.5后对volatile语义增强，通过Happens-Before规则

- Happens-Before规则

  它表达的意思是：前面一个操作的结果对后续操作是可见的。正式的说法是：Happens-Before约束了编译器的优化行为，允许优化但是优化后一定要遵守Happens-Before规则

  1. 程序的顺序性规则

     在一个线程中，按程序顺序， 前面的操作Happens-Before于后续的任意操作。程序前面对某个变量的修改一定是对后续操作是可见的

  2. volatile变量规则

     对一个volatile变量的写操作Happens-Before于后续对这个变量的读操作

  3. 传递性

     如果A Happens-Before B，且B Happens-Before C，那么A Happens-Before C

     对于上面的代码示例

     - x=42 Happens-Before v=ture，规则1
     - v=true Happens-Before 读v=ture，规则2
     - 根据规则3，则x=42 Happens-Before 读v=true，也即A线程设置x=42对B线程可见

  4. 管程中锁的规则

     所谓管程是一种同步原语，在Java中的实现就是synchronized。这条规则是指对一个锁的解锁Happens-Before于后续对这个锁的加锁

     管程中的锁在Java中是隐式实现的，在进入同步块之前会自动加锁，代码快执行完会自动释放锁，都是由编译器完成的

     ```java
     synchronized (this) { //此处自动加锁
       // x是共享变量,初始值=10
       if (this.x < 12) {
         this.x = 12; 
       }  
     } //此处自动解锁
     ```

     x初始是10，A加锁写12，解锁后。B线程进入时，能看到A对x的操作，即B线程能看到x=12

  5. 线程start规则

     主线程A启动子线程B后，子线程B能看到主线程在启动子线程B前的操作。换句话说，如果线程A调用线程B的start（线程A中启动线程B），那么该start操作Happens-Before于子线程B的任意操作

     ```java
     Thread B = new Thread(()->{
       // 主线程调用B.start()之前
       // 所有对共享变量的修改，此处皆可见
       // 此例中，var==77
     });
     // 此处对共享变量var修改
     var = 77;
     // 主线程启动子线程
     B.start();
     ```

  6. 线程join规则

     主线程A等待子线程B完成（主线程A调用子线程B的join），当子线程B完成后（主线程A中join返回），主线程能看到子线程的操作，所谓看到是指对共享变量的操作。换句话说，在线程A中，调用线程B的join并成功返回，那么线程B中的任意操作Happens-Before于该join操作的返回

     ```java
     Thread B = new Thread(()->{
       // 此处对共享变量var修改
       var = 66;
     });
     // 例如此处对共享变量修改，
     // 则这个修改结果对线程B可见
     // 主线程启动子线程
     B.start();
     B.join()
     // 子线程所有对共享变量的修改
     // 在主线程调用B.join()之后皆可见
     // 此例中，var==66
     ```

- final关键字

  final修饰变量，是告诉编译器：这个变量生而不变，随便优化。1.5之前优化错了，构造函数的错误重排导致线程可能看到final变量的值变化。1.5之后Java内存模型对final变量重排进行约束，只要保证构造函数没有逸出。

  所谓逸出，只构造函数将this赋值给了全局变量global.obj。线程通过global.obj读取x是可能读到0的

  ```java
  final int x;
  // 错误的构造函数
  public FinalFieldExample() { 
    x = 3;
    y = 4;
    // 此处就是讲this逸出，
    global.obj = this;
  }
  ```

- synchronized关键字

  - 修饰静态方法

    默认锁的是当前类的Class对象

  - 修饰非静态方法

    默认锁的是当前实例对象this

  - 修饰代码块

    需要一个对象来锁

  锁和受保护资源的关系：1：N，可以一个锁保护多个资源，但是不能多个锁保护一个资源，多个锁之前没有可见性保证

  加锁的本质是在锁对象的时候在对象头中写入当前线程id。synchronized(new obj())这种代码加锁无用，首先JVM逃逸分析后会直接进优化掉，因为new出来的对象只能在一个地方使用，其他线程不能对它解锁，其次哪怕不优化每个线程进来都是不同的锁

- 原子性的本质：操作的中间状态不可见，因此解决原子性问题就是保证中间状态对外不可见
- 管程：管理共享变量以及对共享变量的操作过程，让其支持并发
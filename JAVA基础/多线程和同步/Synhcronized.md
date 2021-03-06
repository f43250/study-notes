# Synchronized

## 位置与区别
| 修饰区域 | 含义 |
| ---- | ---- |
| 修饰静态方法/代码块`类对象` | 整个静态方法，作用的对象是这个类的所有对象 |
| 修饰普通方法或者代码块`this` | 作用的对象是调用这个方法的对象,不同线程调用同一个对象的同步方法会被锁住

## synchronized 和 volatile 可以保证原子性吗?
 - volatile: 告诉JVM当前变量的值是不确定的,从主存中读取,只能保证可见性,不能保证原子性.
   - 可见性: 每个线程改变自己虚拟机栈的变量副本后,强制写入主存(线程共享),然后让其余线程虚拟机栈的副本失效(告诉过期),让其余线程读取此变量时
    强制从主存中读取(同时刷新自己虚拟机栈的CPU高速缓存副本).
 - synchronized: 
   - 原子性:加锁后其他线程无法获取锁,就算cpu时间片用完,由于是可冲入锁,下一个时间片还是只有被他自己获取.
   - 有序性:由于只能被单线程访问,只能顺序执行
   - 可见性:
     - 线程加锁前:清空工作内存的变量值,从而使其他线程读取变量需要冲主存中读取
     - 线程解锁前:讲共享变量最新值刷入主存.

## Java 对象头构成
 - 对象头
   - markword
   - Class对象指针
 - 实例数据
 - 对齐填充
## Synchronized锁升级过程
![锁升级过程](/静态资源/锁升级过程.png )

- 偏向锁开关:使用JVM参数 -XX:+UseBiasedLocking 
  - 偏向锁打开状态: 无锁状态->偏向锁(一个线程执行)->轻量级锁(两个线程交替执行)->重量级锁(多个线程竞争)
  - 偏向锁关闭状态: 无锁不可偏向->轻量级锁(两个线程交替执行)->重量级锁

- 无锁->偏向锁: 在对象头markdword里面设置当前线程Id
- 偏向->轻量级锁:CAS+自旋(+自适应自旋)修改markword
- 轻量级锁->重量级:自旋尝试获取锁,失败达到默认10次阈值.
![锁升级过程](/静态资源/Markword.png )

## 偏向锁撤销
- 原获得偏向锁的线程同步代码块执行完了，那么这个时候会把对象头设置成无锁状态并且争抢锁的线程可以基于CAS重新偏向当前线程 
- 如果原获得偏向锁的线程的同步代码块还没执行完,这个时候会把原获得偏向锁的线程升级为轻量级锁后继续执行同步代码块 
## 批量锁撤销和批量重偏向
- 批量重偏向:
  1. 当对某个类的对象偏向锁撤销20次，则偏向锁认为，后面的锁需要重新偏向新的线程（批量重偏向）
- 批量锁撤销: 
  1. 当某个类的对象的偏向锁累计被撤销到阈值40次（从40次开始），则偏向锁认为偏向锁撤销过于频繁，则后面的对象包括新生成的对象（标识为101和001）如果需要使用锁，则直接轻量级锁，不在使用偏向锁（即禁用了偏向锁）

## hashcode方法
1. 当一个对象已经计算过identity hash code，它就无法进入偏向锁状态；当一个对象当前正处于偏向锁状态，并且需要计算其identity hash code的话，则它的偏向锁会被撤销，并且锁会膨胀为重量锁；重量锁的实现中，ObjectMonitor类里有字段可以记录非加锁状态下的mark word，其中可以存储identity hash code的值。或者简单说就是重量锁可以存下identity hash code。

## Synchronized和Lock区别
- 存在层面: 
  1. Syncronized 是Java 中的一个关键字，Lock 是 Java 中的一个接口
- 锁的释放条件:
  1. Syncronized:获取锁的线程执行完同步代码后，自动释放；2. 线程发生异常时，JVM会让线程释放锁；
  2. Lock:Lock 必须在 finally 关键字中释放锁，不然容易造成线程死锁
- 锁的获取:
  1. 在 Syncronized 中，假设线程 A 获得锁，B 线程等待。如果 A 发生阻塞，那么 B 会一直等待。在 Lock 中，会分情况而定，
  2. Lock 中有尝试获取锁的方法，如果尝试获取到锁，则不用一直等待 
- 锁的状态:
  1. Synchronized 无法判断锁的状态
  2. Lock 可以判断
- 锁的类型:
  1. Synchronized 是可重入，不可中断，只可公平锁
  2. Lock 锁则是 可重入，可判断，可公平锁也可非公平
- 锁的性能:
  1. Lock 锁可以提高多个线程进行读的效率(使用 readWriteLock)

  2. 在竞争不是很激烈的情况下，Synchronized的性能要优于ReetrantLock，但是在资源竞争很激烈的情况下，Synchronized的性能会下降几十倍，但是ReetrantLock的性能能维持常态；

  3. ReetrantLock 提供了多样化的同步，比如有时间限制的同步，可以被Interrupt的同步（synchronized的同步是不能Interrupt的）等。
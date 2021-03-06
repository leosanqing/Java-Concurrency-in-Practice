# Java 内存模型

<!-- TOC -->

- [Java 内存模型](#java-内存模型)
    - [Java 内存间的操作](#java-内存间的操作)
    - [Java 内存模型中的 8 个原子操作](#java-内存模型中的-8-个原子操作)
    - [Java 内存模型规定在执行上述 8 个操作时的规则](#java-内存模型规定在执行上述-8-个操作时的规则)
        - [有关变量拷贝过程的规则](#有关变量拷贝过程的规则)
        - [有关加锁的规则](#有关加锁的规则)
    - [原子性、可见性与有序性](#原子性可见性与有序性)
        - [原子性](#原子性)
        - [可见性](#可见性)
        - [有序性](#有序性)
            - [Happens-Before 规则](#happens-before-规则)

<!-- /TOC -->

## Java 内存间的操作

Java 中，线程、工作内存、主内存三者的交互关系如下：

![Java内存结构.png](./pic/Java内存结构.png)

通过上图可以发现，Java 线程对变量的所有操作（读取、赋值等）都必须在工作内存中进行，不能直接读写主内存中的变量。即使是 volatile 变量，也是从工作内存中读取的，只是它有特殊的操作顺序规定，使得看起来像是直接在主内存中读写。

**一个变量从主内存拷贝到工作内存，再从工作内存同步回主内存的过程：**

```
|主内存| -> read -> load -> |工作内存| -> user -> |Java线程| -> assign -> |工作内存| -> store -> write -> |主内存|
```

> 注意：read 和 load，store 和 write 不一定是连续执行的，中间可以插入其他命令



## Java 内存模型中的 8 个原子操作

- `lock`：作用于主内存，把一个变量标识为一个线程独占状态。
- `unlock`：作用于主内存，释放一个处于锁定状态的变量。
- `read`：作用于主内存，把一个变量的值从主内存传输到线程工作内存中，供之后的 load 操作使用。
- `load`：作用于工作内存，把 read 操作从主内存中得到的变量值放入工作内存的变量副本中。
- `use`：作用于工作内存，把工作内存中的一个变量传递给执行引擎，虚拟机遇到使用变量值的字节码指令时会执行。
- `assign`：作用于工作内存，把一个从执行引擎得到的值赋给工作内存的变量，虚拟机遇到给变量赋值的字节码指令时会执行。
- `store`：作用于工作内存，把工作内存中的一个变量传送到主内存中，供之后的 write 操作使用。
- `write`：作用于主内存，把 store 操作从工作内存中得到的变量值存入主内存的变量中。



## Java 内存模型规定在执行上述 8 个操作时的规则

### 有关变量拷贝过程的规则

- 不允许 read 和 load，store 和 write 单独出现
- 不允许线程丢弃它最近的 assign 操作，即工作内存变化之后必须把该变化同步回主内存中
- 不允许一个线程在没有 assign 的情况下将工作内存同步回主内存中，也就是说，只有虚拟机遇到变量赋值的字节码时才会将工作内存同步回主内存
- 新的变量只能从主内存中诞生，即不能在工作内存中使用未被 load 和 assign 的变量，一个变量在 use 和 store 前一定先经过了 load 和 assign

### 有关加锁的规则

- 一个变量在同一时刻只允许一个线程对其进行 lock 操作，但是可以被一个线程多次 lock（锁的可重入）
- 对一个变量进行 lock 操作会清空这个变量在工作内存中的值，然后在执行引擎使用这个变量时，需要通过 assign 或 load 重新对这个变量进行初始化
- 对一个变量执行 unlock 前，必须将该变量同步回主内存中，即执行 store 和 write 操作
- 一个变量没有被 lock，就不能被 unlock，也不能去 unlock一个被其他线程 lock 的变量



## 原子性、可见性与有序性

### 原子性

- 给用户提供了字节码指令：monitorenter 和 monitorexit 来隐式的使用 lock 和 unlock。
- 这两个字节码反映到 Java 代码中就是同步块：synchronized。

### 可见性

- 当一条线程修改了共享变量的值，其他线程可以立即得知这个修改。
- 实现方式：在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值的方式实现，依赖主内存作为传输媒质。
- 可以保证可见性的关键字：
  - `volatile`：通过volatile的特殊规则；
  - `synchronized`：通过“对一个变量执行 unlock 操作前，必须将该变量同步回主内存”这条规则。
  - `final`：被 final 修饰的字段，一旦完成了初始化，其他线程就能看到它，并且它也不会再变了。
    - 即只要不可变对象被正确的构建出来（没有发生 this 引用溢出），它就是线程安全的。

### 有序性

- 如果在本线程内观察，所有操作都是有序的。
  - 即 Java 内存模型会保证重排序后的执行，在线程内看起来和串行的效果是一样的。
- 如果在一个线程观察另一个线程，所有操作都是无序的。
- 可以保证有序性的关键字：
  - volatile：本身禁止指令重排序；
  - synchronized：通过保证线程的串行执行来保证有序性，因为“线程内表现为串行的语义”。
- 如果两个操作之间缺乏 Happens-Before 规则，那么 JVM 可对它们任意地排序。

#### Happens-Before 规则

- 操作的顺序：
	- **程序顺序规则：** 如果代码中操作 A 在操作 B 之前，那么同一个线程中 A 操作一定在 B 操作前执行，即在本线程内观察，所有操作都是有序的。
	- **传递性：** A 先于 B ，B 先于 C 那么 A 必然先于 C。
- 锁和 volatile：
	- **监视器锁规则：** 监视器锁的解锁操作必须在同一个监视器锁的加锁操作前执行。
	- **volatile 变量规则：** 对 volatile 变量的写操作必须在对该变量的读操作前执行，这样才能保证时刻读取到这个变量的最新值。
- 线程和中断：
	- **线程启动规则：** `Thread#start()` 方法一定先于该线程中执行的操作。
	- **线程结束规则：** 线程的所有操作先于线程的终结。
	- **中断规则：** 假设有线程 A，其他线程 interrupt A 的操作先于检测 A 线程是否中断的操作，即对一个线程的 interrupt() 操作和 interrupted() 等检测中断的操作同时发生，那么 interrupt() 先执行。
- 对象生命周期相关：
	- **终结器规则：** 对象的构造函数执行先于 finalize() 方法。


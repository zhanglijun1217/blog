---
title: Lock接口用法和与synchronized的比较
copyright: true
date: 2019-07-22 23:17:05
tags:
	- Lock
	- ReentrantLock
categories:
	- 并发编程
	- 并发基础
---

# 关于锁的一些知识
1. 可重入锁

如果锁具有可重入性，则称为可重入锁。synchronized和ReentrantLock都是可重入锁。其实可重入锁实际上表明了锁的分配机制：是基于线程的分配还是基于方法调用的分配。
比如代码：

<!--more-->

```java
class MyClass {
    public synchronized void method1() {
        method2();
    }
     
    public synchronized void method2() {
         
    }
}
```
在method1获取到锁之后，再去调用method2是可以重入的，此时锁的对象也是当前调用方法的对象。如果是不可重入的，那么这里在调用method2时，还要去申请获取自身持有的锁。
2. 可中断锁

java中，synchronized是不可中断锁，而Lock是可中断锁。synchronized在未获取到锁时，只能等待，不能响应中断；而Lock接口的lockInterruptibly()方法即体现了lock的可中断性。

3. 公平锁和非公平锁

公平锁即尽量以请求锁的顺序来获取锁，比如多个线程等待锁，这个锁释放时，等待时间最久的线程（最先请求的线程）会获取到锁，这就是公平锁。
而非公平锁因为线程的竞争和被调度是不公平的。比如synchronized是非公平锁，无法保证获取锁的顺序。

而ReentrantLock和ReentrantReadWriteLock是可以根据构造函数的参数来设置是公平锁还是非公平锁。
```java
ReentrantLock lock = new ReentrantLock(true); // true代表是公平锁
```
ReentrantLock也实现了isFair等判断是否为公平锁的方法。

当然公平锁的性能因为要排序所以会在高并发下比非公平锁差。

4. 读写锁

这里指的是维护了两个锁，一个读锁和一个行锁。java中提供了ReentrantReadWriteLock。和数据库的S和X锁一样，读锁和读锁可以共享，不阻塞读操作。
而写锁是独占的，不共享。

```java
public class ReadWriteLockTest {

    public static void main(String[] args) {
        ReadWriteLockDemo readWriteLockDemo = new ReadWriteLockDemo();


        for (int i = 0; i < 100; i++) {
            try {
                Thread.sleep(200);
            }catch (Exception e) {

            }
            if (i %3 == 0) {
                // 写线程：
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        readWriteLockDemo.setNumber(999);
                    }
                }).start();
            } else {
                // 读线程
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        readWriteLockDemo.get();
                    }
                }).start();
            }

        }
    }
}

class ReadWriteLockDemo {

    private int number;

    private ReadWriteLock readWriteLock = new ReentrantReadWriteLock();


    public void setNumber(int number) {
        // 获取写锁
        Lock lock = readWriteLock.writeLock();
        lock.lock();
        try {
            this.number = number;
            System.out.println(Thread.currentThread().getName() + "修改了number");
        } finally {
            lock.unlock();
        }
    }

    public void get() {
        Lock lock = readWriteLock.readLock();
        // 这里注意因为lock也可能报错 应该放在try的外边，防止没有加锁而去调用unlock方法导致异常
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + "读取到了number：" + number);
        } finally {
            lock.unlock();
        }
    }
}
```

5. 关于使用Lock接口的写法

```
Lock lock = ...;
lock.lock();
try{
    //处理任务
}catch(Exception ex){
     
}finally{
    lock.unlock();   //释放锁
}
```
- Lock接口需要手动解锁，finally中执行unlock不多说
- lock.lock这个方法要放在try之外，这里原因是如果lock失败，执行finally中unlock方法时会报错。当然也可以finally中判断下再unlock。

# Lock接口使用

Lock接口的出现是因为synchronized的一些缺陷：
- 被synchronized阻塞的线程等待时无法中断或者超时释放，如果占用锁的线程因为io很耗时，就会很影响性能。
- synchronized不支持读写锁这种场景隔离。任何线程的操作都会等待独占锁，也牺牲了性能。
- synchronized并不能知道当前线程是否获取到了独占锁，而Lock接口也提供了API去做判断。

当然Lock接口需要用户自己手动执行unlock，否则容易造成死锁。而synchronized关键字是不需要手动释放锁的。

## Lock中的方法
- lock方法：同步获取锁，如果其他线程已经获取锁，则进行等待。
- trylock方法：有返回值，会尝试获取锁，不会阻塞，会立即返回，拿不到锁会返回false。
- tryLock(long time, TimeUnit timeunit) ：和tryLock一样，但会阻塞对应的时间。
- lockInterruptibly()方法：当通过这个方法获取锁时，如果线程正在等待获取锁，那么这个线程能响应中断。比如线程A获取到了锁，线程B在等待，对线程B调用interrupt能中断B的等待过程，抛出InterruptException。

这里注意因为lockInterruptibly()方法抛出了异常，调用端要处理抛出的异常中断：
```java
public void method() throws InterruptedException {
    lock.lockInterruptibly();
    try {  
     //.....
    }
    finally {
        lock.unlock();
    }  
}
```

这里如果对获取到锁的线程interrupt，是否抛出异常，取决于业务逻辑，比如在Thread.sleep就会响应中断（这个中断和lockInterruptibly没关系，是sleep自身的响应），而LockSupport并不会响应中断。
```

    private Lock lock = new ReentrantLock();

    public static void main(String[] args) throws InterruptedException{
        LockInterruptiblyTest lockInterruptiblyTest = new LockInterruptiblyTest();
        MyTask task1 = new MyTask(lockInterruptiblyTest, "task1线程");
        MyTask task2 = new MyTask(lockInterruptiblyTest, "task2线程");
        task1.start();
        Thread.sleep(200); // 保证让task1线程先获取到锁
        task2.start();
        task1.interrupt(); // task1获取到锁在执行业务逻辑并不会响应interrupt中断抛出异常
        task2.interrupt();// task2在等待锁时抛出异常

    }

    public void biz() throws InterruptedException{
        lock.lockInterruptibly();
        try {
            for (int i = 0; i < 10 ; i++) {
                System.out.println(Thread.currentThread().getName() + "第" + i + "次执行业务逻辑");
//                TimeUnit.MILLISECONDS.sleep(2000); //注意本身线程休眠就是响应interrupt中断方法的 这里模拟不要用sleep 会影响观察
                LockSupport.parkNanos(1000 * 1000 * 200); // LockSupport不响应中断
            }
        } finally {
            lock.unlock();
            System.out.println(Thread.currentThread().getName() + "finally释放了锁");
        }
    }


    static class MyTask extends Thread {
        private LockInterruptiblyTest lockInterruptiblyTest;

        public MyTask(LockInterruptiblyTest test, String name) {
            setName(name);
            this.lockInterruptiblyTest = test;
        }

        @Override
        public void run() {
            try {
                lockInterruptiblyTest.biz();
            } catch (InterruptedException e) {
                System.out.println(Arrays.toString(e.getStackTrace()));
                System.out.println(Thread.currentThread().getName() + "等待锁时被interrupt中断");
            }

        }
    }
```

- newCondition()方法：Lock接口还提供了条件Condition，对线程等待、唤醒更加详细和灵活。

在Lock创建出的condition会配合await() 和 signal/signalAll()等方法来实现线程通讯，且Lock能拥有多个condition，即多个等待队列，对比在配合synchronized使用的wait()、nnotify()/notifyAll()只能等待一个条件，非常灵活。

同时在生产者/消费者模型中，Conditio也可与避免synchronized和wait/notify产生的虚假唤醒问题。这里不做过多篇幅。


# synchronized和Lock的比较
这里总结下他们之间的区别：
- synchronized是内置的语言实现，是一个关键字，而Lock是一个接口，提供了多种实现。（Lock接口虽然只有ReentrantLock一个实现，但是接口更加灵活，且读写锁虽然没有直接实现Lock接口，但也算Lock的实现。
- Lock需要手动去调用unlock方法解锁，而synchronized发生异常或执行完自动释放锁。
- Lock功能更加丰富，提供了非阻塞的获取锁、带有时间的获取锁、等待锁线程响应中断、判断是否占有锁、condition条件锁、公平锁等功能。
- Lock接口提供了读写锁，对锁使用分了场景，提高了性能。
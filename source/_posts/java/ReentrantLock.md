---
title: ReentrantLock
comments: false
toc: false
date: 2020-02-28 11:28:50
categories: java
tags:
---

java除了使用关键字synchronized外，还可以使用ReentrantLock实现独占锁的功能。而且ReentrantLock相比synchronized而言功能更加丰富，使用起来更为灵活，也更适合复杂的并发场景

## 简介

ReentrantLock常常对比着synchronized来分析，我们先对比着来看然后再一点一点分析。

* synchronized是独占锁，加锁和解锁的过程自动进行，易于操作，但不够灵活。ReentrantLock也是独占锁，加锁和解锁的过程需要手动进行，不易操作，但非常灵活。

* synchronized可重入，因为加锁和解锁自动进行，不必担心最后是否释放锁；ReentrantLock也可重入，但加锁和解锁需要手动进行，且次数需一样，否则其他线程无法获得锁。

* synchronized不可响应中断，一个线程获取不到锁就一直等着；ReentrantLock可以相应中断。

ReentrantLock好像比synchronized关键字没好太多，我们再去看看synchronized所没有的，一个最主要的就是ReentrantLock还可以实现公平锁机制。

> 公平锁就是在锁上等待时间最长的线程将获得锁的使用权。通俗的理解就是谁排队时间最长谁先执行获取锁。

## 使用

### 简单使用

在这里我们定义了一个ReentrantLock，然后在test方法中分别lock和unlock，如下:

``` java
public class ReentrantLockTest {

    private final static Lock lock = new ReentrantLock();

    private static void test() {
        try {
            lock.lock();
            System.out.println(Thread.currentThread().getName() + "获取了锁");
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            System.out.println(Thread.currentThread().getName() + "释放了锁");
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        new Thread(ReentrantLockTest::test, "线程A").start();
        new Thread(ReentrantLockTest::test, "线程B").start();
    }
}
```

### 公平锁实现

对于公平锁的实现要结合着可重入特性。公平锁的含义上面已经说了，就是谁等的时间最长，谁就先获取锁

``` java
public class ReentrantLockTest {

    private final static Lock lock = new ReentrantLock(true);

    private static void test() {
        for (int i = 0; i < 2; i++) {
            try {
                lock.lock();
                System.out.println(Thread.currentThread().getName() + "获取了锁");
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                System.out.println(Thread.currentThread().getName() + "释放了锁");
                lock.unlock();
            }
        }
    }

    public static void main(String[] args) {
        new Thread(ReentrantLockTest::test, "线程A").start();
        new Thread(ReentrantLockTest::test, "线程B").start();
        new Thread(ReentrantLockTest::test, "线程C").start();
        new Thread(ReentrantLockTest::test, "线程D").start();
    }
}
```

> new一个ReentrantLock的时候参数为true，表明实现公平锁机制
>
>这里我们多定义几个线程ABCDE，然后在test方法中循环执行了两次加锁和解锁的过程，结果顺序是ABCDE，ABCDE

### 非公平锁实现

非公平锁就是随机的获取，谁运气好，cpu时间片轮到哪个线程，哪个线程就能获取锁，和上面公平锁的区别很简单，就在于先new一个ReentrantLock的时候参数为false，当然我们也可以不写，默认就是false.

> 上面代码的lock改变后运行发现顺序是随机的了

### 响应中断

响应中断就是一个线程获取不到锁，不会傻傻的一直等下去，ReentrantLock会给予一个中断回应，在这里我们举一个死锁的案例:

``` java
public class ReentrantLockTest {

    private final static Lock lock1 = new ReentrantLock();
    private final static Lock lock2 = new ReentrantLock();

    private static class Test implements Runnable {
        private final Lock firstLock;
        private final Lock secondLock;

        public Test(Lock firstLock, Lock secondLock) {
            this.firstLock = firstLock;
            this.secondLock = secondLock;
        }

        @Override
        public void run() {
            try {
                firstLock.lockInterruptibly();
                TimeUnit.MILLISECONDS.sleep(50);
                secondLock.lockInterruptibly();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                firstLock.unlock();
                secondLock.unlock();
                System.out.println(Thread.currentThread().getName() + "获取到了资源正常结束!");
            }
        }
    }

    public static void main(String[] args) {
        Thread t1 = new Thread(new Test(lock1, lock2));
        Thread t2 = new Thread(new Test(lock1, lock2));
        t1.start();
        t2.start();
        t1.interrupt();
    }
}
```

在这里我们定义了两个锁firstLock和secondLock。然后使用两个线程t1和t2构造死锁场景。正常情况下，这两个线程相互等待获取资源而处于死循环状态。但是我们此时t1中断，另外一个线程t2就可以获取资源，正常地执行了。

### 限时等待

通过tryLock方法来实现，可以选择传入时间参数，表示等待指定的时间，无参则表示立即返回锁申请的结果：true表示获取锁成功，false表示获取锁失败。

``` java
 private static class Test implements Runnable {
        private final Lock firstLock;
        private final Lock secondLock;

        public Test(Lock firstLock, Lock secondLock) {
            this.firstLock = firstLock;
            this.secondLock = secondLock;
        }

        @Override
        public void run() {
            try {
                if (!firstLock.tryLock()) TimeUnit.MILLISECONDS.sleep(50);
                if (!secondLock.tryLock()) TimeUnit.MILLISECONDS.sleep(50);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                firstLock.unlock();
                secondLock.unlock();
                System.out.println(Thread.currentThread().getName() + "获取到了资源正常结束!");
            }
        }
    }
```

> 一个线程获取firstLock时候第一次失败，那就等50毫秒之后第二次获取，就这样一直不停的调试，一直等到获取到相应的资源为止,如此其他线程就有可能获取到锁了。如此上述的死锁也就无需中断线程。

当然，我们可以设置tryLock的超时等待时间tryLock(long timeout,TimeUnit unit)，也就是说一个线程在指定的时间内没有获取锁，那就会返回false，就可以再去做其他事了。

---
layout:     post
title:      利用synchronized实现的简单读写锁
subtitle:   读写锁简化版
date:       2018-01-03
author:     wangshenbo
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - JAVA
    - 读写锁
---

## 什么是读写锁
J.U.C并发包中提供了ReentrantReadWriteLock读写锁实现，多个线程读取相同资源的线程同时进行读操作不会有阻塞问题，解决共享资源读取效率问题，适用于多读少写的场景，在传统排它锁实现中，多个读取共享资源的线程会相互阻塞导致读效率低。读写锁可以描述为：当某个单独的线程要对资源进行写操作时，必须要求没有其他的线程在对该资源进行读操作或写操作，同时当某个线程要对资源进行读操作时，必要要求没有线程正在对共享资源进行写操作，但可允许读操作，即读写锁的获取条件如下：

1. 读锁获取条件：没有线程正在对共享资源进行写操作，即写锁未被其它线程获取
2. 写锁获取条件：没有线程正在对共享资源进行读或写操作，即写锁、读锁都未被其它线路获取

## 读写锁重入问题

1. 读读重入：一个线程获取了读锁，该线程能够再获取读锁
2. 读写重入：一个线程获取了读锁，如果系统中无其它线程获取了写锁，该线程能够再获取写锁，否则不可能获取写锁
3. 写读重入：一个线程获取了写锁，该线程再去获取读锁能够成功
4. 写写重入：一个线程获取了写锁，该线程再去获取写锁能够成功


## 利用synchronized实现读写锁

维护了四个读写锁属性，然后通过synchronized关键字更新这四个状态，由于更新状态时只有一个线程能够获取到锁，从而保证各个属性能被同步更新

* writeAccess：写计数，大于0代表写锁被占有
* writeOwner：占有写锁的线程
* writeWaitors：请求写锁但还未获取到写锁的线程数，获取读锁的前提是writeWaitors==0，读写锁一般用在多读少写的场景下可避免写锁饥饿
* readAccessMap：占有读锁的线程与相应的计数

```
public class SimpleReadWriteLock {
    private int writeAccess;
    private int writeWaitors;
    private Map<Thread, Integer> readAccessMap;
    private Thread writeOwner;

    public SimpleReadWriteLock() {
        writeAccess = 0;
        writeWaitors = 0;
        readAccessMap = new HashMap<Thread, Integer>();
        writeOwner = null;
    }

    /**
     * 判断是否可以获取读锁
     * 
     * @return
     */
    private boolean isAcquireReadLock() {
        // 当前线程已经占有写锁，可以获取读锁
        if (writeAccess > 0 && writeOwner == Thread.currentThread()) {
            return true;
        }
        
        // 其它线程占有写锁或有其它待获取写锁的请求，不可以获取读锁
        if (writeAccess > 0 || writeWaitors > 0) {
            return false;
        }
        
        // 无线程占有写锁，可以获取读锁
        return true;
    }

    /**
     * 获取读锁
     * @throws InterruptedException
     */
    public synchronized void readLock() throws InterruptedException {
        while (!isAcquireReadLock()) {
            this.wait(); // 等待
        }

        Integer readAccess = readAccessMap.get(Thread.currentThread());
        if (readAccess == null) {
            readAccess = 1;
        } else {
            readAccess += 1;
        }

        readAccessMap.put(Thread.currentThread(), readAccess);
    }

    /**
     * 释放读锁
     */
    public synchronized void unReadLock() {
        if (readAccessMap.containsKey(Thread.currentThread()) || readAccessMap.get(Thread.currentThread()) <= 0) {
            throw new IllegalStateException("未获取到读锁，无须操作");
        }
        
        // 读锁计数减1，如果为0，则移除Map中的key
        int readAccess = readAccessMap.get(Thread.currentThread()) - 1;
        if (readAccess <= 0) {
            readAccessMap.remove(Thread.currentThread());
        } else {
            readAccessMap.put(Thread.currentThread(), readAccess);
        }
        
        this.notifyAll();
    }

    /**
     * 判断是否可以获取写锁
     * 
     * @return
     */
    private boolean isAcquireWriteLock() {
        // 只有当前线程获取了读锁，则该线程可以获取写锁
        if (readAccessMap.size() == 1 && readAccessMap.get(Thread.currentThread()) > 0) {
            return true;
        }
        
        // 当前线程占有了写锁，则该线程可以获取写锁
        if (writeAccess > 0 && writeOwner == Thread.currentThread()) {
            return true;
        }
        
        // 读/写锁未被其它线程占用，则该线程可以获取写锁
        if (readAccessMap.isEmpty() && writeAccess == 0) {
            return true;
        }
        
        return false;
    }

    /**
     * 获取写锁
     * 
     * @throws InterruptedException
     */
    public synchronized void writeLock() throws InterruptedException {
        writeWaitors += 1;
        while (!isAcquireWriteLock()) {
            this.wait();
        }
        writeWaitors -= 1;
        writeAccess += 1;
        writeOwner = Thread.currentThread();
    }

    /**
     * 释放写锁
     */
    public synchronized void unWriteLock() {
        if (writeAccess <= 0 || writeOwner != Thread.currentThread()) {
            throw new IllegalStateException("未获取写锁，无须操作");
        }
        // 写计算减1
        writeAccess -= 1;
        if (writeAccess == 0) { // 写计数为空则清空writeOwner
            writeOwner = null;
        }
        this.notifyAll();
    }
 }
```


[TOC]

### 1 线程同步辅助类

- 除了基本的线程同步机制（Synchronized、Lock）外，JDK 还提供了 5种线程同步辅助类。用于多线程间的同步
- 其中 Semaphore、CountDownLatch 类已在 [AQS 应用 —— 手写 Semaphore、CountDownLatch](https://blog.csdn.net/kangsa998/article/details/97970564) 一文中分析过
- 本文主要分析 CyclicBarrier、Phaser、Exchanger 3个线程同步辅助类

### 2 CyclicBarrier 解析

- 特性分析：某个考场，只有所有的考生都考完试离开后，**下一波考生(不同线程)**才可以进入这个考场。并且前后两波考生切换过程中，**监考老师**(独立的线程)可以先做一件事（获取这一波考生分数最高的人的名字或统计成绩）
- 这一特性可用于 分治任务中：将任务拆分使得多个线程去做，所有线程完成之后，有独立线程在统计
- 具体使用与源码分析留待以后讨论。。

### 3 Phaser 解析

- 特性分析：某场考试分为**多个阶段**，只有当所有考生都完成一个阶段的试题后，**这些考生(同一批线程！)**才可以同时做下一个阶段的试题。并且在这阶段的切换过程中，监考老师也可以先做一件事情(当一个阶段的结束后会调用**onAdvance() 方法**)
- 结束后调用 onAdvance() 方法这一特性，也可用于分治任务中
- 多阶段特性可以用于 一个任务分为多个阶段性任务中
- 具体使用与源码分析留待以后讨论。。

### 4 Exchanger 解析

- 特性分析：**只有一个**生产者和消费者，并且两个线程都有一个同步点。当两个线程都达到同步点时，交换数据

```jav
//生产者
public class Producer implements Runnable {
    private List<String> buffer;
    private final Exchanger<List<String>> exchanger;
    public Producer(List<String> buffer, Exchanger<List<String>> exchanger) {
        this.buffer = buffer;
        this.exchanger = exchanger;
    }
    public void run() {
        int cycle = 1;
        //进行10次数据交换
        for (int i=0; i<10; i++) {
            //生产数据
            for (int j=0; j<10;j++) {
                String message = "EVENT " + (i*10 + j);
            }
            buffer.add(message);
            buffer = exchange(buffer);//这是生产者的同步点，到达同步掉后，只有被线程被中断或对应的消费者线程也到达它的同步点，才会被唤醒并交换数据！
        }
    }
}
//消费者
public class Consumer implements Runnable {
    private List<String> buffer;
    private final Exchanger<List<String>> exchanger;
    public Consumer(List<String> buffer, Exchanger<List<String>> exchanger) {
        this.buffer = buffer;
        this.exchanger = exchanger;
    }
    public void run() {
        int cycle = 1;
        //进行10次数据交换
        for (int i=0; i<10; i++) {
            buffer = exchange(buffer);//这是消费者的同步点，到达同步掉后，只有被线程被中断或对应的消费者线程也到达它的同步点，才会被唤醒并交换数据！
            consumeBuffer();//消费掉得到的数据
        }
    }
}
```
### 参考
Java 7 并发编程实战手册




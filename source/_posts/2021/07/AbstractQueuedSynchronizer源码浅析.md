---
title: AbstractQueuedSynchronizer 源码浅析
date: 2021/07/05
categories: 源码浅析
urlName: AbstractQueuedSynchronizer
---
本学习笔记主要参考Doug Lea论文及源代码编写而成。主要分为AQS的设计和实现、AQS源码解读。AQS的设计和实现中阐述了要解决的三个核心问题：锁的状态、线程的阻塞和唤醒、阻塞队列。AQS源码解读主要涉及`acquire`、`addWaiter`、`acquireQueued`、`shouldParkAfterFailedAcquire`、`cancelAcquire方法`，文中提出了目前仍然存在的疑问和困惑。

## AQS设计
要设计一个锁，必须解决两个问题：
1. 如何申请锁？
2. 如何释放锁？
这里引用Doug Lea论文中的伪代码来说明上述两个问题。
```
while (synchronization state does not allow acquire) {
    enqueue current thread if not already queued;
    possibly block current thread;
}
```
如果当前线程不能获取锁的状态，并且不在等待获取锁的队列中的话，那么就把当前线程加入到队列中；如果可能的话，会阻塞当前线程。
```
update synchronization state;
if (state may permit a blocked thread to acquire)
    unblock one or more queued threads;
```
当前线程释放锁时，首先更新锁的状态；如果允许阻塞线程去获取锁的状态，那么唤醒已经入队的阻塞的一个或者多个线程。
为了能够支持上述伪代码中的申请锁和释放锁，那么需要以下几件事情：
- 原子的维护锁的状态（synchronization state）
- 阻塞和唤醒线程机制
- 维护等待锁的线程队列

### 锁的状态
通过getState、setState和compareAndSetState来访问和更新锁的状态（state）。

### 线程的阻塞和唤醒
通过`LockSupport.park`和`LockSupport.unpark`来阻塞线程和唤醒线程。
`LockSupport.unpark`可以多次调用，不过只有一次起作用，即只颁发了一个许可（permit），多次调用等同于一次调用。
先调用`LockSupport.unpark`，再调用`LockSupport.park`则不会阻塞线程。
`LockSupport.park`通过`Thread.interrupt`的支持，实现自身的相对时间和绝对时间的超时机制。

### 阻塞队列
AQS框架的核心就是维护一个FIFO的阻塞队列。因为此队列是先进先出（FIFO）的，那么也就意味着AQS框架也不支持基于优先级的同步了。
AQS采用CLH队列作为阻塞队列的优势：
- 入队（lock-freedom）和出队（obstruction-freedom）都很快。
- 只需要比较head==tail，就能够快速检查是否有线程等待。
- release status是分散的，避免了内存竞争。
AQS队列在CLH队列的基础上做了一些改进：
- AQS队列的节点需要显示的去唤醒他的successor，所以Node节点包含next，指向下一个节点，但是不保证原子性（因为没有合适的技术来确保双链表的更新能够达到lock-freedom并发级别）。
- Node节点中的status是用来控制阻塞（blocking），而不是自旋（spinning）的。AQS中的Node.status不通过判断release状态来决定是否获取锁，而是通过tryAcquire来决定是否要出队，而且只有Node.pred==head，才有资格去调用tryAcquire，避免了竞争。
- 与其他语言比较，AQS的队列不用操心内存释放问题。

## AQS 实现
AQS伪代码的实现版本（此版本的伪代码不支持超时和中断）
申请锁：
```
if (!tryAcquire(arg)) {
    node = create and enqueue new node;
    pred = node's effective predecessor;
    while (pred is not head node || !tryAcquire(arg)) {
        if (pred's signal bit is set) 
            park();
     else
            compareAndSet pred's signal bit to true;
        pred = node's effective predecessor; 
    }
    head = node;
}
```
释放锁：
```
if (tryRelease(arg) && head node's signal bit is set) { 
    compareAndSet head's signal bit to false;
    unpark head's successor, if one exists;
}
```

## AQS源码解读
AQS通过模板方法来实现由子类去控制锁的申请和释放。AQS提供了`tryAcquire`、`tryRelease`、`tryAcquireShared`和`tryReleaseShared`供子类去实现，即由子类决定如何获取和释放锁。`tryAcquireShared`和`tryReleaseShared`是共享锁的实现方法。

### acquire 方法
```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
1. 调用tryAcquire方法，如果成功，那么获取锁。
2. 如果不成功，则将此线程入队（addWaiter），并且尝试自旋获取锁（acquireQueued）。
3. 此处需要注意的是addWaiter方法仅仅是入队操作，acquireQueued才是让线程自旋去获取锁。

### addWaiter 方法
```
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // 尝试快速入队，如果失败则自旋入队（enq）
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}

private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
可以看到`addWaiter`中的尝试快速入队和`enq`中的自旋入队的区别在于，`enq`中做了`tail`是否为`null`的初始化，即判断队列是否为空。快速入队假设的情况就是大部分情况下，阻塞队列已经不为空了，所以可以省去这一部分的判断。

### acquireQueued 方法
```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 前面提到了只有node的前驱节点是head节点时，
            // node才有资格调用tryAcquire
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 如果获取锁失败，则检查是否需要阻塞（park）当前线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
此处的`cancelAcquire`方法，个人认为不会执行，因为调用`acquireQueued`的`acquire`是非中断的，而`acquireInterruptibly`中的`doAcquireInterruptibly`调用的`cancelAcquire`才会执行。

### shouldParkAfterFailedAcquire 方法
```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```
1. 如果`pred.waitStatus`已经是`Node.SIGNAL`了，那么直接`park`当前线程。
2. 如果`pred.waitStatus > 0`，那么前驱节点已经被取消了，那么跳过所有被取消的前驱节点。
   *这个地方有个疑问，就是`node.prev = pred = pred.prev`为什么没用CAS确保原子操作？个人理解，因为每个节点只会遍历前驱节点，而且只会跳过取消的前驱节点，当前驱节点不是取消状态时，则不会跳过，所以相当于多个线程在操作阻塞队列这个链表上不同区间段的某一段链表，各个线程并没有交叉操作，即不会产生资源竞争。*
3. 如果都不是上述情况，那么更新前驱节点`status`为`Node.SIGNAL`，即表示后置节点需要唤醒。设置完成后，则再次进入自旋，当自旋再次进入此方法时，那么执行第一步中的判断，已经为`true`，则阻塞当前线程。

### cancelAcquire 方法
```
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;
    node.thread = null;
    // Skip cancelled predecessors
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;
    // predNext is the apparent node to unsplice. CASes below will
    // fail if not, in which case, we lost race vs another cancel
    // or signal, so no further action is necessary.
    Node predNext = pred.next;
    // Can use unconditional write instead of CAS here.
    // After this atomic step, other Nodes can skip past us.
    // Before, we are free of interference from other threads.
    node.waitStatus = Node.CANCELLED;
    // If we are the tail, remove ourselves.
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
            // 注意此处为何又对next指针做CAS？
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node);
        }
        node.next = node; // help GC
    }
}
```
`cancelAcquire`中对`next`指针的赋值采用了CAS来保证原子操作，不太明白，因为会导致不同的`next`？

## 参考资料
1. [The java.util.concurrent Synchronizer Framework](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)
2. AbstractQueuedSynchronizer 源代码(JDK 1.8)








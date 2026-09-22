# dba和其他细节

# 《DBA实战手记》薛晓刚著 读书笔记
- 进度: 210/474

# 其他一些数据库方面的经典错误案例

## rr没有解决幻读（幻读固然需要解决,但是应尽量避免在一个事务里,让写语句像"夹心饼干"的利一样\夹在两条"一模一样"的奥奥(读语句)中间,解决问题并不高明,造成幻读的这种写法本不应该被提倡)
<video src="https://github.com/user-attachments/assets/639875d6-2edf-44df-a20e-d197644cb5e3" controls width="800">
</video>

## rr没有解决幻读是因为更新了undo log版本链，而非因为第二次读的时候重新生成了read view
<video src="https://github.com/user-attachments/assets/0eadbb00-d5bb-4f0a-8e2c-b634dc6a54d4" controls width="800">
</video>

<video src="https://github.com/user-attachments/assets/daae2ea3-9547-4684-9c84-f1b58e37b773" controls width="800">
</video>

<video src="https://github.com/user-attachments/assets/0b9bf8ee-2997-4ca6-a38d-851946de4462" controls width="800">
</video>

## rc的read view
<video src="https://github.com/user-attachments/assets/06cb4b37-71ee-492c-9ace-f594d362a88f" controls width="800">
</video>

## 元数据锁
<video src="https://github.com/user-attachments/assets/f5919505-905a-437b-8013-196456476f1d" controls width="800">
</video>




# 源码赏析
## AQS
- 1.addWaiter为什么要把enq(node)单独拆出来一个方法,是为了优先处理一次大多数的pred不为null的场景吗?感觉代码风格像个do...while...
try-once + spin-fallback

- 2.cancelAcquire里,没有成功设置成next链, 才会unparkSuccessor.

- 3.next链靠不住, prev链永远是最先更新的(更新prev-更新tail-更新next,这么一个顺序).

- 4.cancelAcquire是因为node.next = node这句话发生得比较晚, 所以,按照next找的话会多找.

- 5.cancelAcquire如果cas失败就不会执行到compareAndSetNext来更新next链的(随缘即可),因为它一开始就node.waitStatus = Node.CANCELLED;了,可以依靠后面的新任务addWaiter后通过acquireQueued加入队列里面的方法shouldParkAfterFailedAcquire方法来惰性逻辑标记删除(最终也是依赖gc来回收的). 

- 6.unparkSuccessor直接唤醒下一个节点.

- 7.接上一点:有至少以下途径会让pred.next不等于predNext, 从而cas不成功
  - shouldParkAfterFailedAcquire来更新前序节点的SIGNAL状态和prev/next链从而跳过所有前序cancelled状态的节点
  - cancelAcquire连续取消两个连续的节点时
队列：head → A → B → C → D(tail)，B 和 C 同时被取消：
线程1 取消 B：  想改 A.next
线程2 取消 C：  pred 跳到 A，记下 predNext = A.next = B
              准备 CAS(A.next, 期望=B, 新值=D)
此刻线程1 抢先把 A.next 改了
              → 线程2 执行 CAS 时，A.next 已经 ≠ B
              → pred.next ≠ predNext → CAS 失败

- 8.设置为head的时候, 就会把它的thread设置为null. 
private void setHead(Node node) {
        head = node;
        node.thread = null;
        node.prev = null;
    }
只要不是当前节点head,就会把thread包装入队.

- 9.LockSupport.park(this);中断信号在锁获取过程中被“延迟处理”，而不是被忽略。虽迟但到.
- 10.if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0); 相当乐观, SIGNAL能清就清, 别人已经清了SIGNAL 或者 新入队的节点通过shouldParkAfterFailedAcquire明确说要等我唤醒于是又把我标记为了SIGNAL, 那也没关系, 尽力而为
- 11.tryAcquireNanos的tryAcquire用短路, acquireInterruptibly的用分支...
- 12.FairSync的tryAcquire和nonfairTryAcquire的包含的通用方法, 也不抽出来...
- 13.偏向锁撤销是很麻烦的, 所以它要延迟开启.
- 14.匿名偏向 -> 带有线程id的偏向. 就好似, 就如同
- 15.bulk revoke 该类的其他某对象. 撤销19次...或许有一天
- 16.bulk rebias 该类超过撤销阈值, 后续跳过偏向\直接升级为轻量级锁.
- 17.NonfairSync: 我要抢三次才作罢.





## ThreadPoolExecutor
- 1. Are workers subject to culling?的恐怖...
- 2. 怎么样先增加到max workers, 再加任务到workQueue? -> make offer return false(假满) 然后等worker加到极限了再realOffer入队
- 3. 怎样不拒绝任务入队? 重写offer, 里面调用put阻塞
- 4. 滑动窗口最大值的恐怖...
- 5. 围圈报数的恐怖...

- 6. addWorker如果走到addWorkerFailed(w) 
- - -> tryTerminate(); 
- - -> interruptIdleWorkers(ONLY_ONE) 
- - -> if (!t.isInterrupted() && w.tryLock())就中断线程t.interrupt();
- - -> runWorker里的getTask里的 workQueue.take() 响应中断
- - -> Runnable r从workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) : workQueue.take()发现中断异常
- - -> 捕获 InterruptedException：中断异常没有向上传播，只是在 catch 里把 timedOut 重置，然后回到 for 循环顶部
- - -> 继续回到循环, 重新检查状态, 如果此时：
SHUTDOWN 且队列空 就走 decrementWorkerCount 于是 返回 null 导致 退出使得其返回null
- - -> runWorker的 
while死循环条件 “(task = getTask()) != null”
被“getTask()返回null”打破, 
任务执行完,
走到completedAbruptly = false;
- - -> finally块走到 processWorkerExit(w, completedAbruptly); 第二个参数传入false
- - -> 继续走到 tryTerminate();循环往复
- - -> workerCount == 0 → 进入 TIDYING → TERMINATED，池子真正关闭

terminate workers one by one to avoid concurrency...的恐怖

- 7. interruptIdleWorkers
内部会跳过"已经被中断"的线程（!t.isInterrupted()），
只中断真正空闲、且能 tryLock 成功的 worker。

- 8. 如果是onlyOne的情况, 
if (onlyOne) break; 保证同一时刻只有一个线程能进来发起中断，避免多个线程并发调用 tryTerminate() 时重复、扎堆地中断
一次只推一个，配合级联，既高效又不会误伤正在执行任务的 worker

- 9. addWorker-添加并启动工作线程
两步走:
添加 和 启动

- 10. SHUTDOWN了, 只有当任务为null, 且workQueue不为空的时候
->必须同时满足这三个条件, 
addWorker才能走下去

- 11. addWorker时为什么要加锁mainLock
- - ->防止此时线程池shutdown或者shutdownNow

- 12. runWorker这函数很有意思: 
- - 一方面, w.unlock(); // allow interrupts 即, 可以随时打断我(SHUTDOWN而非STOP状态时,只要能拿到锁,就中断w.tryLock()); 
- - 另一方面, 只要我开始做任务了, 就不能被打断了w.lock();

- 13. Worker本身也很有意思
- - setState(-1); // inhibit interrupts until runWorker

- 14. Worker的tryAcquire是最巧妙的了
- - 经典的不可重入锁, 非0即1, 只判断compareAndSetState(0, 1)

- 15. shutdown(状态变为SHUTDOWN)和shutdownNow(状态变为STOP)
- - shutdown(状态变为SHUTDOWN): interruptIdleWorkers();拿到w.tryLock()再中断它, 保证存量任务做完
- - shutdownNow(状态变为STOP): interruptWorkers();不管三七二十一,上去就中断所有, 压根不关心是否空闲, 无差别地全部处理, 马上叫停, t.interrupt();

- 16. runWorker里面判断线程池是否STOP时, 会存在一个时间缝隙, 会存在一个竞态条件: 
- - if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();

- - 如果没有第二个判断 (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP)), 那么worker就会误以为是"脏中断", 于是, 它会清除中断标志位, 继续执行任务, 导致该叫停的任务没有停下来
- - 一言以蔽之, recheck是为了防止线程"误以为是脏中断但其实不是"

- 17. getTask -> compareAndDecrementWorkerCount(c)
- - boolean timedOut = false; // Did the last poll() time out? 取任务超时
- - boolean timed = allowCoreThreadTimeOut || wc > corePoolSize; 非核心

- 18. getTask只判断 a.线程池状态 和 b.线程数量, 并不指定某个线程是否核心, addWorker时是核心\但后面可能会decrement掉

- 19.  (wc > 1 || workQueue.isEmpty())当是独苗线程时, 如果队列有任务, 则也不能decrement

- 20.  take()的实现
- - ArrayBlockingQueue: notEmpty.await(); 等待notEmpty.signal();唤醒的Condition类 出队则notFull.signal();单锁吞吐低
- - LinkedBlockingQueue: notEmpty.await();经典双锁

- 20.  tryTerminate TIDYING -> TERMINATED

- 21. processWorkerExit如果if (runStateLessThan(c, STOP))如果是 不正常移除 或 是正常移除线程导致没有worker了, 就再补回来一个工作线程


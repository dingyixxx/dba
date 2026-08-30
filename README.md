# dba和其他细节

# 《DBA实战手记》薛晓刚著 读书笔记

# 其他一些数据库方面的经典错误案例

## rr没有解决幻读
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

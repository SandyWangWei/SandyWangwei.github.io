# BlockingQueue笔记

------

BlockingQueue是为了实现线程安全的阻塞队列而设计的，在取元素期间，若当前队列为空会阻塞直到有新的元素添加进来；在添加 元素期间，若当前队列满了会阻塞到有空的位置。BlockingQueue提供了四种接口：

> * 抛异常
> * 同步返回具体数值
> * 调用时阻塞当前线程
> * 阻塞超时

BlockingQueue的实现有5种：ArrayBlockingQueue、LinkedBlockingQueue、DelayQueue、PriorityBlocingQueue和SynchronousQueue。ArrayBlockingQueue&LinkedBlockingQueue是我们最长使用。

BlockingQueue的成员函数主要是以下几种：
#### 1. Insert
     add add(e)     //插入元素，true表示插入成功，如果队列满了则抛出IllegalStateException，如果插入为null也会抛出NPE
     offer offer(e) //插入元素，true表示插入成功，false表示队列满了
     put put(e)     //插入元素，队列满了则线程阻塞等待
     offer(e, time, unit)  //插入元素，队列满了之后会等待直到传入的超时时间
#### 2.Remove
     remove remove() //移除元素，true表示移除成功
     poll poll()     //取队列顶部元素
     take take()     //获取顶部元素，队列为空则阻塞线程
     poll(time, unit)//取队列顶部元素，队列为空则最多等待超市时间time
   </tr>
#### 3.Examine
     drainTo(Collection<? super E> c) //转移当前队列元素到新的队列，返回为转移的元素数量
     peek()      //获取队列顶部元素，但是不会移除
     element()   //获取队列顶部元素，但是不会移除，与peek不同地方在于队列为空时候抛出NoSuchElementException

------

## LinkedBlockingQueue

LinkedBlockingQueue主要是用链表实现的阻塞队列，是一种FIFO队列，其实针对LinkedBlockingQueue的用法非常简单，因为线程安全问题内部都帮你解决了，并且实现了插入移除的线程等待，先看下它的用法：
```java
class Producer implements Runnable {
    private final LinkedBlockingQueue queue;
    Producer(LinkedBlockingQueue q) { queue = q; }
    public void run() {
      try {
        while (true) { queue.put(produce()); }
      } catch (InterruptedException ex) { ... handle ...}
    }
    Object produce() { ... }
  }
 
  class Consumer implements Runnable {
    private final LinkedBlockingQueue queue;
    Consumer(LinkedBlockingQueue q) { queue = q; }
    public void run() {
      try {
        while (true) { consume(queue.take()); }
      } catch (InterruptedException ex) { ... handle ...}
    }
    void consume(Object x) { ... }
  }
 
  class Setup {
    void main() {
      LinkedBlockingQueue q = new LinkedBlockingQueue();
      Producer p = new Producer(q);
      Consumer c1 = new Consumer(q);
      Consumer c2 = new Consumer(q);
      new Thread(p).start();
      new Thread(c1).start();
      new Thread(c2).start();
    }
```

这里只是以LinkedBlockingQueue为例，这样的用法试用于任何一种实现了BlockingQueue的队列。LinkedBlockingQueue实现了一种以链表形式存储的队列结构，其成员变量如下：

```java
    /**
     * 链表的结点类.
     */
    static class Node<E> {
        E item;
        Node<E> next;
        Node(E x) { item = x; }
    }
    /** 容量，默认为Integer.MAX_VALUE */
    private final int capacity;
    /** 当前元素数量 */
    private final AtomicInteger count = new AtomicInteger();
    /** 链表头结点 */
    transient Node<E> head;
    /** 链表尾结点 */
    private transient Node<E> last;
    /** take, poll等取操作的锁 */
    private final ReentrantLock takeLock = new ReentrantLock();
    /** takes操作等待 */
    private final Condition notEmpty = takeLock.newCondition();
    /** put、offer等方法的锁 */
    private final ReentrantLock putLock = new ReentrantLock();
    /** puts操作等待 */
    private final Condition notFull = putLock.newCondition();

```

Condition是条件变量，条件变量的实例化是通过一个Lock对象上调用newCondition()方法来获取的，这样，条件就和一个锁对象绑定起来了。因此，Java中的条件变量只能和锁配合使用，来控制并发程序访问竞争资源的安全。

LinkedBlockingQueue是如何实现阻塞队列的呢，我们以put为例：
```java
public void put(E e) throws InterruptedException {
        //如果传入的元素为null，抛出NPE
        if (e == null) throw new NullPointerException();
        // 预设值为-1
        int c = -1;
        Node<E> node = new Node<E>(e);
        //获取入队锁
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            //即使count没有锁保护也能在等待时使用，主要的原因在于count值只能在这个阶段减少
            //如果当前的队列满了，则等待其他线程唤醒，这里的唤醒主要是在take()方法中
            while (count.get() == capacity) {
                notFull.await();
            }
            //等待结束放入新结点
            enqueue(node);
            //count值增加
            c = count.getAndIncrement();
            //如果发现此时队列有空位置，唤醒入队条件变量
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            //结束之后释放入队锁
            putLock.unlock();
        }
        //如果当前队列count为0，则唤醒出队条件变量
        if (c == 0)
            signalNotEmpty();
    }
```

这里在调用put方法插入元素主要是依靠入队锁putLock和入队条件变量notFull来实现多线程插入与阻塞队列的，这里的条件变量notFull唤醒则是在take、poll等取元素的线程中调用。至于最后一点为什么要在当前队列count为0的时候唤醒出队条件呢？

我们来设定一个场景，当前队列为空，此时put/take未开始执行，这时候首先调用take会发线程阻塞住，一直停留在notEmpty.await()，当我们插入数据的线程调用了put之后，一旦调用count.getAndIncrement()，当前的count变为1；

```java
    while (count.get() == 0) {
         notEmpty.await();
    }
    x = dequeue();
    c = count.getAndDecrement();
    if (c > 1)
         notEmpty.signal();
```

这时候take中的出队条件满足，取出了该元素，但是此时获取到的c是0，并没有唤醒出队条件变量notEmpty。因此需要put来执行唤醒。

其他的take、offer等方法的逻辑类似就不多加赘述，关于其他类型的BlockingQueue后续有时间也会总结下。

###参考文章

    https://www.cnblogs.com/java-zhao/p/5135958.html

# 什么是JUC
java.util.concurrent在并发编程中使用的工具类
# 线程和进程
进程是要给应用程序。  
线程是一个进程中的一个执行单元。  
一个进程可以有多个线程。  

# Synchronized和Lock区别
1. Synchronized内置的Java关键字，Lock是一个Java类
2. Synchronized无法判断获取锁的状态，Lock 可以判断是否获取到锁
3. Synchronized会自动释放锁，Lock必须手动释放锁！如果没有释放锁会造成`死锁 `
4. Synchronized 线程1(获得锁，阻塞)，线程2(一直等待)，Lock锁就不会一直等待。
5. Synchronized可重入锁，不可以中断的，非公平；Lock可重入锁，可以判断锁，非公平（可以自己设置，默认非公平）
6. Synchronized适合锁少量的代码同步问题，Lock适合锁大量的同步代码块。

```java
//简单的Lock代码
public class User {
    private int num = 50;
    Lock lock = new ReentrantLock();//默认不公平锁 true 公平  false 不公平
    public void sava(String name){
        lock.lock();
        try {
            if(num>0){
                System.out.println(name+" 购买了,当前剩余数量："+(num--));
            }
        }catch (Exception e){
            e.getMessage();
        }finally {
            lock.unlock();
        }
    }
    public int getNum() {
        return num;
    }
    public void setNum(int num) {
        this.num = num;
    }
}
//测试
public static void main(String[] args) {
        User user = new User();
        new Thread(() -> {
            for (int i = 0; i < 60; i++) {
                user.sava("小王");
            }
        }).start();

        new Thread(() -> {
            for (int i = 0; i < 60; i++) {
                user.sava("小李");
            }
        }).start();

        new Thread(() -> {
            for (int i = 0; i < 60; i++) {
                user.sava("小猪");
            }
        }).start();
    }
//结果
小王 购买了,当前剩余数量：50
小猪 购买了,当前剩余数量：49
小猪 购买了,当前剩余数量：48
小猪 购买了,当前剩余数量：47
小猪 购买了,当前剩余数量：46
......
```

# 生产者和消费者问题

> synchronized版本（以下代码还是存在线程安全问题）

```java
public class Test {
    public static void main(String[] args) {
        Data data = new Data();
        new Thread(() -> {
            for (int i = 0; i <10 ; i++) {
                try {
                    data.increment();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        },"A").start();
        new Thread(() -> {
            for (int i = 0; i <10 ; i++) {
                try {
                    data.decrement();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        },"B").start();
    }
}
//判断等待，业务，通知
class Data{
    private int number = 0;
    //+1
    public synchronized void increment() throws Exception{
        if(number != 0){
            //等待
            this.wait();
        }
        number ++ ;
        System.out.println(Thread.currentThread().getName()+"==>>"+number);
        //通知其他线程我加完了
        this.notify();
    }
    //-1
    public synchronized void decrement() throws Exception{
        if(number == 0){
            //等待
            this.wait();
        }
        number--;
        System.out.println(Thread.currentThread().getName()+"==>>"+number);
        //通知其他线程我减完了
        this.notifyAll();
    }
}
```

> 以上代码在只要2个线程（A,B）的情况不会出现问题，如果在加（C,D）两个线程会出现是问题

```JAVA
A==>>6
C==>>7
A==>>8
D==>>7
C==>>8
```

`线程也可以被唤醒，而不会被通知，中断或者超时，这种情况被称为虚假唤醒，所以这里应该把if判断改为while循环`

`因为if判断只判断一次，而while判断会等待`

![img](https://cdn.jsdelivr.net/gh/my-zhb/CDN/img/20201214171351.png)

> Lock版本（其中lock替代synchronized ，Condition取代了对象监视器）

```java
public class Test {
    public static void main(String[] args) {
        Data data = new Data();
        new Thread(() -> {
            for (int i = 0; i <10 ; i++) {
                data.increment();
            }
        },"A").start();
        new Thread(() -> {
            for (int i = 0; i <10 ; i++) {
                data.decrement();
            }
        },"B").start();
        new Thread(() -> {
            for (int i = 0; i <10 ; i++) {
                data.increment();
            }
        },"C").start();
        new Thread(() -> {
            for (int i = 0; i <10 ; i++) {
                data.decrement();
            }
        },"D").start();
    }
}
//判断等待，业务，通知
class Data{
    private int number = 0;
    //创建lock锁（lock替代了synchronized）
    Lock lock = new ReentrantLock();
    //创建监视器（Condition替代了同步监视器）
    Condition condition = lock.newCondition();

    //+1
    public  void increment(){
        lock.lock();
        try {
            while (number != 0){
                //等待
                condition.await();
            }
            number ++ ;
            System.out.println(Thread.currentThread().getName()+"==>>"+number);
            //通知其他线程我加完了
            condition.signalAll();
        }catch (Exception e){
            e.getMessage();
        }finally {
            //必须关闭不然会出现死锁
            lock.unlock();
        }
    }
    //-1
    public  void decrement(){
        lock.lock();
        try {
            while (number == 0){
                //等待
                condition.await();
            }
            number -- ;
            System.out.println(Thread.currentThread().getName()+"==>>"+number);
            //通知其他线程我加完了
            condition.signalAll();
        }catch (Exception e){
            e.getMessage();
        }finally {
            lock.unlock();
        }
    }
}
```

> Condition 如何精准的通知和唤醒线程（A->B->C->D）

通过创建多个Condition 来实现精准的通知和唤醒

```JAVA
public class Test {
    public static void main(String[] args) {
        Data data = new Data();
        new Thread(() -> {
            for (int i = 0; i <10 ; i++) {
                data.condition1();
            }
        },"A").start();
        new Thread(() -> {
            for (int i = 0; i <10 ; i++) {
                data.condition2();
            }
        },"B").start();
        new Thread(() -> {
            for (int i = 0; i <10 ; i++) {
                data.condition3();
            }
        },"C").start();
    }
}

class Data{
    private int number = 1;

    Lock lock = new ReentrantLock();
    Condition condition1 = lock.newCondition();
    Condition condition2 = lock.newCondition();
    Condition condition3 = lock.newCondition();

    public void condition1(){
        lock.lock();
        try {
            while (number!=1){
                condition1.await();
            }
            number++;
            System.out.println(Thread.currentThread().getName()+"===>>>AAAAA");
            condition2.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }

    public void condition2(){
        lock.lock();
        try {
            while (number!=2){
                condition2.await();
            }
            number++;
            System.out.println(Thread.currentThread().getName()+"===>>>BBBBB");
            condition3.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }

    public void condition3(){
        lock.lock();
        try {
            while (number!=3){
                condition3.await();
            }
            number=1;
            System.out.println(Thread.currentThread().getName()+"===>>>CCCCC");
            condition1.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
}
```



# 8锁现象

> 此处synchronized锁的是方法的调用者，sendSms()和call()两个方法用的是同一把锁,谁先拿到谁先执行

`TimeUnit.SECONDS.sleep(1)不管放在方法里还是线程A和线程B之间 它的执行结果都是一样的`

1. 一个对象，两个同步方法块，睡眠放在线程A和线程B之间

```java
public class Test1 {
    public static void main(String[] args) {
        Phone phone = new Phone();
        new Thread(() -> {
            phone.sendSms();
        },"A").start();
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        new Thread(() -> {
            phone.call();
        },"B").start();
    }
}

class Phone{
    //synchronized 锁的是方法的调用者
    //两个方法用的是同一把锁,谁先拿到谁先执行
    public synchronized void sendSms(){
        System.out.println("发短信");
    }
    public synchronized void call(){
        System.out.println("打电话");
    }
}
```

2. 一个对象，两个同步方法块，睡眠放在同步方法里

```java
public class Test1 {
    public static void main(String[] args) {
        Phone phone = new Phone();
        new Thread(() -> {
            phone.sendSms();
        },"A").start();
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        new Thread(() -> {
            phone.call();
        },"B").start();
    }
}

class Phone{
    //synchronized 锁的是方法的调用者
    //两个方法用的是同一把锁,谁先拿到谁先执行
    public synchronized void sendSms(){
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("发短信");
    }
    public synchronized void call(){
        System.out.println("打电话");
    }
}
//结果
发短信
打电话
```

>hello()方法没有synchronized关键字，所以不受影响正常输出

3. 一个对象，两个同步方法块，一个普通方法（随机）

```java
public class Test2 {
    public static void main(String[] args) {
        Phone2 phone = new Phone2();
        new Thread(() -> {
            phone.sendSms();
        },"A").start();

        new Thread(() -> {
            phone.hello();
        },"B").start();
    }
}
class Phone2{
    public synchronized void sendSms(){
        System.out.println("发短信");
    }
    public synchronized void call(){
        System.out.println("打电话");
    }
    //hello没有synchronized关键字所以不受影响
    public void hello(){
        System.out.println("hello");
    }
}
```

> 因为是2个对象调用方法，锁的不是同一个对象，sendSms()睡了1秒，所以先输出打电话，在输出发短信，如果没有睡眠，就是随机的。

4. 两个对象，两个同步方法块（随机）

```java
public class Test2 {
    public static void main(String[] args) {
        Phone2 phone1 = new Phone2();
        Phone2 phone2 = new Phone2();
        new Thread(() -> {
            phone1.sendSms();
        },"A").start();

        new Thread(() -> {
            phone2.call();
        },"B").start();
    }
}
class Phone2{
    public synchronized void sendSms(){
        System.out.println("发短信");
    }
    public synchronized void call(){
        System.out.println("打电话");
    }
}
```

> 因为sendSms()和call()方法都是static修饰的，所以类一加载就有了，实际这里锁的是class模版

5. 增加两个静态的同步方法，只有一个对象（先输出发短信）

```java
public class Test3 {
    public static void main(String[] args) {
        Phone3 phone = new Phone3();
        new Thread(() -> {
            phone.sendSms();
        },"A").start();

        new Thread(() -> {
            phone.call();
        },"B").start();
    }
}
class Phone3{
    //synchronized 锁的是对象
    //因为是static修饰的，所以类一加载就有了，实际这里锁的是class模版
    public static synchronized void sendSms(){
        System.out.println("发短信");
    }
    public static synchronized void call(){
        System.out.println("打电话");
    }
}
```

6. 两个对象，两个静态的同步方法 （随机）

```java
public class Test3 {
    public static void main(String[] args) {
        Phone3 phone1 = new Phone3();
        Phone3 phone2 = new Phone3();
        new Thread(() -> {
            phone1.sendSms();
        },"A").start();

        new Thread(() -> {
            phone2.call();
        },"B").start();
    }
}
class Phone3{
    //synchronized 锁的是对象
    //因为是static修饰的，所以类一加载就有了，实际这里锁的是class模版
    public static synchronized void sendSms(){
        System.out.println("发短信");
    }
    public static synchronized void call(){
        System.out.println("打电话");
    }
}
```

> 如果同一个对象调用对象的一个非静态同步方法与同步方法，此时会出现两把锁，一个是类锁，一个是对象调用者锁

7. 一个静态的同步方法，一个非静态的同步方法（随机）

```java
public class Test3 {
    public static void main(String[] args) {
        Phone3 phone = new Phone3();
        new Thread(() -> {
            phone.sendSms();
        },"A").start();

        new Thread(() -> {
            phone.call();
        },"B").start();
    }
}
class Phone3{
    //static修饰的方法锁的实际上是class模版
    public static synchronized void sendSms()   {
        System.out.println("发短信");
    }
    //正常情况下锁的是方法的调用者
    public synchronized void call(){
        System.out.println("打电话");
    }
}
//结果
发短信
打电话
```

8. 两个不同的对象，一个静态同步方法块，一个非静态同步方法块（随机）

```java
public class Test3 {
    public static void main(String[] args) {
        Phone3 phone1 = new Phone3();
        Phone3 phone2 = new Phone3();
        new Thread(() -> {
            phone1.sendSms();
        },"A").start();

        new Thread(() -> {
            phone2.call();
        },"B").start();
    }
}
class Phone3{
    public static synchronized void sendSms()   {
        System.out.println("发短信");
    }
    public synchronized void call(){
        System.out.println("打电话");
    }
}
//结果
结果随机
```

> 总结

1. new,this 具体的一个对象（对象锁）
2. static 唯一的（class锁，类锁）

# 集合类不安全

> List

并发情况下往ArrayList写入数据

```java
public class ListTest {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        for (int i = 1; i <= 10 ; i++) {
            new Thread(() -> {
                list.add(UUID.randomUUID().toString().substring(0,5));
                System.out.println(list);
            },String.valueOf(i)).start();
        }
    }
}
//会出现 java.util.ConcurrentModificationException 并发修改异常
```

解决方法

1. `List<String> list = new Vector<>()`不建议用，因为底层是synchronized修饰的效率低
2. `List<String> list = Collections.synchronizedList(new ArrayList<>())`
3. `List<String> list = new CopyOnWriteArrayList<>();juc下面的CopyOnWriteArrayList方法`
   1. `CopyOnWrite` 写入时复制，简称COW 计算机设计领域的一种优化策略。
   2. 多个线程调用的时候，list读取的时候，固定的，写入会出现覆盖，所以写入数据前会先复制原有数据，避免造成数据问

```java
public class ListTest {
    public static void main(String[] args) {
        List<String> list = new CopyOnWriteArrayList<>();
        for (int i = 1; i <= 10 ; i++) {
            new Thread(() -> {
                list.add(UUID.randomUUID().toString().substring(0,5));
                System.out.println(list);
            },String.valueOf(i)).start();
        }
    }
}
```

> Set (与list差不多)

解决办法

1. `Collections.synchronizedSet(new HashSet<>())`
2. `Set<String> set = new CopyOnWriteArraySet<>()`

```java
public class SetTest {
    public static void main(String[] args) {
        Set<String> set = new CopyOnWriteArraySet<>();
        for (int i = 1; i <= 10 ; i++) {
            new Thread(() -> {
                set.add(UUID.randomUUID().toString().substring(0,5));
                System.out.println(set);
            },String.valueOf(i)).start();
        }
    }
}
```

`HashSet底层其实就是HashMap的key`

> Map

解决办法

1. `Collections.synchronizedMap(new HashMap<>())`
2. ` Map<String,String> map = new ConcurrentHashMap<>()`

```java
public class MapTest {
    public static void main(String[] args) {
        Map<String,String> map = new ConcurrentHashMap<>();
        for (int i = 1; i <= 100 ; i++) {
            new Thread(() -> {
                map.put(Thread.currentThread().getName(),UUID.randomUUID().toString().substring(0,5));
                System.out.println(map);
            },String.valueOf(i)).start();
        }
    }
}
```

`Map默认的两个参数为加载因子0.75f，初始容量1<<4（16）`

# Callable

## 什么是Callable

`Callable`接口类似于`Runnable`，因为它们都是为其实例可能由另一个线程执行的类设计的。 然而，`Runnable`不返回结果，也不能抛出被检查的异常，`Callable`能返回结果也可以抛出异常。

## 怎么启动Callable

通过FutureTask，FutureTask的本质就是Runnable，因为Runnable实现了FutureTask

FutureTask.get()方法可能会造成阻塞，因为这里会去等待线程的执行，可以通过异步的方式解决

```java
public class Test {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        //FutureTask的本质就是Runnable,FutureTask是一个适配器，来适配Callable
        MyThread myThread = new MyThread();
        FutureTask myThreadFutureTask = new FutureTask(myThread);
        
        new Thread(myThreadFutureTask,"A").start();
        //这里会有缓存，所以执行结果之会输出一次
        new Thread(myThreadFutureTask,"B").start();
        //这里通过FutureTask.get()方法可以获取到Callable的返回值
        Object o = myThreadFutureTask.get();
        System.out.println(o);
    }
}

class MyThread implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        System.out.println("call()");
        return 10;
    }
}
//执行结果
call()
10
```



# 常用的辅助类

## CountDownLatch

定义：可以直接理解为一个减法计数器

用法：`CountDownLatch`用给定的*计数*初始化。 `await`方法阻塞，直到由于`countDown()`方法的调用而导致当前计数达到零，之后所有等待线程被释放，并且任何后续的`await`调用立即返回。这是一个一次性的现象 - 计数无法重置。 如果您需要重置计数的版本，请考虑使用`CyclicBarrier`。

```java
public class CountDownLatchDemo {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(3);
        for (int i = 1; i <=3 ; i++) {
            new Thread(() -> {
                //-1操作
                countDownLatch.countDown();
                System.out.println(Thread.currentThread().getName()+"出去了");
            },String.valueOf(i)).start();
        }
        //阻塞，直到线程执行完毕
        countDownLatch.await();
        System.out.println("出去完了，可以关门了");
    }
}
//执行结果
1出去了
3出去了
2出去了
出去完了，可以关门了
```

理解：

​	` countDownLatch.countDown()`每次调用减1

​	`countDownLatch.await()`等待计数器归零，然后才往下继续执行。

## CyclicBarrier

定义：可以理解为一个加法计数器

用法：CyclicBarrier支持一格可选的`Runnable`命令，每个屏障点运行一次，在派对在的最后一格线程到达之后，但在任何线程释放之前，在任何一方继续进行之前，此屏障点操作对更新共享状态很有用。

```java
public class CyclicBarrierDemo {

    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(10,() -> {
            System.out.println("小明有10元钱，可以打游戏了");
        });
        for (int i = 1; i <= 10 ; i++) {
            final int temp = i;
            new Thread(() -> {
                System.out.println("小明现在"+temp+"元,还不能打游戏");
                try {
                    //cyclicBarrier.await()会一直阻塞直到值为10的时候，才会去执行， System.out.println("小明有10元钱，可以打游戏了");这个线程
                    //如果这里循环9次，它会一直阻塞在这里。
                    cyclicBarrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
//执行结果
小明现在1元,还不能打游戏
小明现在4元,还不能打游戏
小明现在3元,还不能打游戏
小明现在2元,还不能打游戏
小明现在6元,还不能打游戏
小明现在5元,还不能打游戏
小明现在7元,还不能打游戏
小明现在8元,还不能打游戏
小明现在9元,还不能打游戏
小明现在10元,还不能打游戏
小明有10元钱，可以打游戏了
```

## Semaphore

定义：相当于信号量

用法：一个个计数信号量。 在概念上，信号量维持一组许可证。 如果有必要，每个`acquire()`都会阻塞，直到许可证可用，然后才能使用它。 每个`release()`添加许可证，潜在地释放阻塞获取方。 但是，没有使用实际的许可证对象; Semaphore只保留可用数量的计数，并相应地执行

```java
public class SemaphoreDemo {
    //例子：打游戏，一共2台电脑,但是有4个人想玩
    public static void main(String[] args) {
        //这个参数表示最大2个线程数
        Semaphore semaphore = new Semaphore(2);
        for (int i = 1; i <= 4; i++) {
            new Thread(() -> {
                try {
                    //获得
                    semaphore.acquire();
                    System.out.println(Thread.currentThread().getName()+"号小朋友，抢到了电脑可以完游戏了");
                    Thread.sleep(5000);
                    System.out.println(Thread.currentThread().getName()+"号小朋友，玩了5秒钟，被他妈妈叫走了");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    //释放
                    semaphore.release();
                }
            },String.valueOf(i)).start();
        }
    }
}
//执行结果
1号小朋友，抢到了电脑可以完游戏了
3号小朋友，抢到了电脑可以完游戏了
3号小朋友，玩了5秒钟，被他妈妈叫走了
1号小朋友，玩了5秒钟，被他妈妈叫走了
2号小朋友，抢到了电脑可以完游戏了
4号小朋友，抢到了电脑可以完游戏了
2号小朋友，玩了5秒钟，被他妈妈叫走了
4号小朋友，玩了5秒钟，被他妈妈叫走了
```

理解：

​	`semaphore.acquire()`会获取线程，如果线程满了，它会等待线程释放，然后在去获取。（相当于减1）

​	` semaphore.release()`释放线程,然后唤醒等待的线程。（相当于加1）

作用：

​	多个共享资源互斥的时候使用

​	并发限流，保证最大线程数

# 读写锁

> ReadWriteLock

ReadWriteLock维护一堆关联的locks，一个用于只读操作，一个用于写入操作，read lock可以有多个线程同时进行，write lock只有一个。

读锁也叫`共享锁`（一次可以多个线程共享）

写锁也叫`独占锁`,`排它锁`（一次只能一个线程独有）

```java
public class ReadWriteLockDemo {

    public static void main(String[] args) {
        MyCache myCache = new MyCache();

        //写入
        for (int i = 1; i <= 10 ; i++) {
            final int temp = i;
            new Thread(() -> {
                myCache.put(temp+"",temp+"");
            },String.valueOf(i)).start();
        }

        //读取
        for (int i = 1; i <= 10 ; i++) {
            final int temp = i;
            new Thread(() -> {
                myCache.get(temp+"");
            },String.valueOf(i)).start();
        }
    }
}

class MyCache{
    private volatile Map<String,Object> map = new HashMap<>();
    //获取读写锁
    private ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    public void put(String key,Object value){
        //获取写锁
        readWriteLock.writeLock().lock();
        try {
            System.out.println(Thread.currentThread().getName()+"线程,写入"+key);
            Object o = map.put(key,value);
            System.out.println(Thread.currentThread().getName()+"线程,写入完毕");
        }catch (Exception e){
            e.getMessage();
        }finally {
            //关闭写锁
            readWriteLock.writeLock().unlock();
        }
    }
    public void get(String key){
        //获取读锁
        readWriteLock.readLock().lock();
        try {
            System.out.println(Thread.currentThread().getName()+"线程,读取"+key);
            Object o = map.get(key);
            System.out.println(Thread.currentThread().getName()+"线程,读取完毕");
        }catch (Exception e){
            e.getMessage();
        }finally {
            //关闭锁
            readWriteLock.readLock().unlock();
        }
    }
}
```

# 阻塞队列

`BlockingQueue`常用的实现类：`ArrayBlockingQueue`,`LinkedBlockingQueue`,`SynchronousQueue`

![image-20201216175639630](https://cdn.jsdelivr.net/gh/my-zhb/CDN/img/20201216175649.png)

`什么情况下会使用阻塞队列：多线程并发处理，线程池`

`队列是先去先出`

## ArrayBlockingQueue 

> 四组API

|      方法      | 抛出异常  | 有返回值，不抛出异常 | 阻塞等待 |                 超时等待                  |
| :------------: | :-------: | :------------------: | :------: | :---------------------------------------: |
|      添加      |   add()   |     offer()[E e]     |  put()   | offer()[E e, long timeout, TimeUnit unit] |
|      移除      | remove()  |        poll()        |  take()  |    poll()[long timeout, TimeUnit unit]    |
| 检查队列首元素 | element() |        peek()        |          |                                           |

1. 抛出异常

```java
    public static void test1(){
        //创建array队列，给定大小3
        ArrayBlockingQueue<String> queue = new ArrayBlockingQueue<String>(3);
        //添加元素
        System.out.println(queue.add("1"));
        System.out.println(queue.add("2"));
        System.out.println(queue.add("3"));
        //添加第四个抛出异常，java.lang.IllegalStateException: Queue full 队列已满
        //System.out.println(queue.add("4"));
        
        //检查队首
        System.out.println(queue.element());

        //移除元素
        System.out.println(queue.remove());
        System.out.println(queue.remove());
        System.out.println(queue.remove());
        //移除第四个抛出异常，java.util.NoSuchElementException 队列已空
        //System.out.println(queue.remove());
    }
```

2. 有返回值，不抛出异常

```java
    public static void test2(){
        //创建array队列，给定大小3
        ArrayBlockingQueue<String> queue = new ArrayBlockingQueue<String>(3);
        //添加元素
        System.out.println(queue.offer("1"));
        System.out.println(queue.offer("2"));
        System.out.println(queue.offer("3"));
        //返回false，不会抛出异常
        System.out.println(queue.offer("4"));

        //检查队首
        System.out.println(queue.peek());
        
        //移除元素
        System.out.println(queue.poll());
        System.out.println(queue.poll());
        System.out.println(queue.poll());
        //返回null，不会抛出异常
        System.out.println(queue.poll());
    }
```

3. 阻塞等待

```java
    public static void test3() throws InterruptedException {
        //创建array队列，给定大小3
        ArrayBlockingQueue<String> queue = new ArrayBlockingQueue<String>(3);
        //添加元素
        queue.put("1");
        queue.put("2");
        queue.put("3");
        //当第四往里存的时候会一直等待(阻塞)
        queue.put("4");

        //移除元素
        System.out.println(queue.take());
        System.out.println(queue.take());
        System.out.println(queue.take());
        //当里面没有元素的时候，在去取，它会一直等待(阻塞)
        System.out.println(queue.take());
    }
```

4. 超时等待

```java
    public static void test4() throws InterruptedException {
        //创建array队列，给定大小3
        ArrayBlockingQueue<String> queue = new ArrayBlockingQueue<String>(3);
        //添加元素
        System.out.println(queue.offer("1"));
        System.out.println(queue.offer("2"));
        System.out.println(queue.offer("3"));
        //添加第四个元素等待3秒，如果3秒过后还没位子，就自动退出
        System.out.println(queue.offer("4",3, TimeUnit.SECONDS));

        System.out.println(queue.peek());

        //移除元素
        System.out.println(queue.poll());
        System.out.println(queue.poll());
        System.out.println(queue.poll());
        //移除，也是同理 先等待3秒，有就移除，3秒过后自动退出
        System.out.println(queue.poll(3,TimeUnit.SECONDS));
    }
```

## SynchronousQueue同步队列

`这个队列容量为1，进入一个元素之后，必选把这个元素取出来才能继续存`,对应的存是`put()`,取是`take()`

```java
    public static void test5() throws InterruptedException {
        SynchronousQueue<String> synchronousQueue = new SynchronousQueue<String>();
        new Thread(() -> {
            try {
                System.out.println(Thread.currentThread().getName()+"线程写入==>>"+1);
                synchronousQueue.put(1+"");
                System.out.println(Thread.currentThread().getName()+"线程写入==>>"+2);
                synchronousQueue.put(2+"");
                System.out.println(Thread.currentThread().getName()+"线程写入==>>"+3);
                synchronousQueue.put(3+"");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"A").start();
        new Thread(() -> {
            try {
                TimeUnit.SECONDS.sleep(2);
                System.out.println(Thread.currentThread().getName()+"线程取出==>>"+synchronousQueue.take());
                TimeUnit.SECONDS.sleep(2);
                System.out.println(Thread.currentThread().getName()+"线程取出==>>"+synchronousQueue.take());
                TimeUnit.SECONDS.sleep(2);
                System.out.println(Thread.currentThread().getName()+"线程取出==>>"+synchronousQueue.take());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"B").start();
    }
//结果
A线程写入==>>1
B线程取出==>>1
A线程写入==>>2
B线程取出==>>2
A线程写入==>>3
B线程取出==>>3
```



# 线程池(重要)

> 池化技术

程序的运行会占用系统的资源,优化资源的使用称之为`池化技术`，如：线程池、内存池、对象池等等

如果平凡的创建、销毁非常浪费资源。

`池化技术：就是事先准备好一些资源，有人要用，就直接在这里拿，用完之后在换回来`

> 线程池的好处

1. 降低资源的消耗
2. 提高响应的速度
3. 方便管理

`总结：线程可复用，可以控制最大并发数，可以管理线程`

> 线程池——三大方法

1. `Executors.newSingleThreadExecutor()`单个线程
2. `Executors.newFixedThreadPool(3)`自定义最大线程数
3. `Executors.newCachedThreadPool()`可伸缩的

```java
    public static void main(String[] args) {
        ExecutorService service1 = Executors.newSingleThreadExecutor();
        ExecutorService service2 = Executors.newFixedThreadPool(3);
        ExecutorService service3 = Executors.newCachedThreadPool();
        try {
            for (int i = 0; i <100 ; i++) {
                service1.execute(() -> {
                    System.out.println(Thread.currentThread().getName());
                });
            }
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            //关闭线程池
            service1.shutdown();
        }
    }
```

> 线程池——七大参数

三大方法的底层都调用的是：`ThreadPoolExecutor`

1. `int corePoolSize` ——> 核心线程池大小
2. `int maximumPoolSize` ——> 最大核心线程池大小
3. `long keepAliveTime` ——> 超时时间，如果超时了，自动释放
4. `TimeUnit unit` ——> 超时单位
5. `BlockingQueue<Runnable> workQueue` ——> 阻塞队列
6. `ThreadFactory threadFactory` ——> 线程工厂，创建线程使用，一般不用动
7. `RejectedExecutionHandler handler` ——> 拒绝策略

```java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
	
	public ThreadPoolExecutor(int corePoolSize, //核心线程池大小
                              int maximumPoolSize,//最大核心线程池大小
                              long keepAliveTime,//超时时间，如果超时了，自动释放
                              TimeUnit unit,//超时单位
                              BlockingQueue<Runnable> workQueue,//阻塞队列
                              ThreadFactory threadFactory,//线程工厂，创建线程使用，一般不用动
                              RejectedExecutionHandler handler) //拒绝策略
    {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

> 自定义线程池

i<=4,有2个线程在执行，因为队列未满

i<=5,有2个线程在执行，因为队列未满

i<=6,有3个线程在执行，因为队列已满

i<=7,有4个线程在执行，因为队列已满

i<=8,有5个线程在执行，因为队列已满

i<=9,报错 因为已经超出最大承载了，最大承载为5+3（最大核心线程数+队列容量）

```java
    public static void main(String[] args) {
        ExecutorService service1 = new ThreadPoolExecutor(
                2,//核心线程2
                5,//最大核心线程5
                3,//等待时长3秒
                TimeUnit.SECONDS,
                new LinkedBlockingDeque<>(3),//队列容量3
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.AbortPolicy()//表示如果队列满了，还有人进来，就不处理，抛出异常
                );
        try {
            //最大承载Deque+maximumPoolSize(最大核心线程)
            //如果超出就会报错java.util.concurrent.RejectedExecutionException
            for (int i = 1; i <=9 ; i++) {
                service1.execute(() -> {
                    System.out.println(Thread.currentThread().getName());
                });
            }
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            //关闭线程池
            service1.shutdown();
        }
    }
```



> 四种拒绝策略

1. `new ThreadPoolExecutor.AbortPolicy()`——> 表示如果队列满了，还有人进来，就不处理，抛出异常
2. `new ThreadPoolExecutor.CallerRunsPolicy()`——> 哪里来的就回哪里去，如果是main线程来的就回main线程
3. `new ThreadPoolExecutor.DiscardOldestPolicy()`——> 如果队列满了，它会尝试和最早的竞争，如果成功，就会执行，如果失败，就会被抛弃，也不会抛出异常
4. `new ThreadPoolExecutor.DiscardPolicy() `——> 如果队列满了就会丢掉任务，不会抛出异常

> 如何去设置线程池最大的大小

1. `CPU密集型`，电脑几核，就是几核，可以保证CPU的效率最高！
2. `IO密集型`，判断你程序中十分消耗IO的线程，只要大于就可以

```java
//获取CPU的核数 
System.out.println(Runtime.getRuntime().availableProcessors());
```



# 四大函数式接口(重要)

> 函数式接口-->只有一个方法的接口

```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
//简化编程模型，在新版本的框架底层大量使用
```

四大函数式接口：`Consumer`、`Function`、`Predicate`、`Supplier`

1. Function

```java
//Function函数接口，有一个输入参数，有一个输出
//只要是函数型接口都可以通过lambda表达是简化
public class FunctionDemo {
    public static void main(String[] args) {
        Function<String,String> function = str -> {
            return str;
        };
        System.out.println(function.apply("231"));
    }
}
输出结果：
231
```

2. Predicate（断定型接口）

```java
//Predicate函数接口，有一个输入参数，会固定返回一个boolean值
public class PredicateDemo {
    public static void main(String[] args) {
        Predicate<String> predicate = (str)->{
            return str.equals("123");
        };
        System.out.println(predicate.test("123"));
    }
}
输出结果：
true
```

3. Consumer(消费型接口)

```java
//Consumer 消费型接口，只有输入，没有返回
public class ConsumerDemo {
    public static void main(String[] args) {
        Consumer<String> consumer = str -> {
            System.out.println(str);
        };
        consumer.accept("123");
    }
}
```

4. Supplier(生产型接口)

```java
//Supplier 生产型接口，没有输入，只有返回
public class SupplierDemo {
    public static void main(String[] args) {
        Supplier<String> supplier = () ->{
          return String.valueOf(1024);
        };
        System.out.println(supplier.get());
    }
}
输出结果：
1024
```



# Stream流式计算

> 什么是Stream流式计算

存储+计算，集合、Mysql本质就是存储东西的，计算机应该交给流来操作！

```java
public class Demo {
    /**
     * 一分钟内完成此题，只能用一行代码实现
     * 现在有5个用户，筛选：
     * 1. ID 必须是偶数
     * 2. 年龄必须大于23岁
     * 3. 用户名转大写字母
     * 4. 用户名字母倒序
     * 5. 只输出一个用户！
     */
    public static void main(String[] args) {
        User u1 = new User(1,"a",21);
        User u2 = new User(2,"b",22);
        User u3 = new User(3,"c",23);
        User u4 = new User(4,"d",24);
        User u5 = new User(6,"e",25);
        List<User> list = Arrays.asList(u1, u2, u3, u4, u5);
        //链式计算
        list.stream()
                .filter(u->{return u.getId()%2==0;})
                .filter(u->{return u.getAge()>23;})
                .map(u->{return u.getName().toUpperCase();})
                .sorted((uu1,uu2)->{return uu2.compareTo(uu1);})
                .limit(1)
                .forEach(System.out::println);
    }
}
```

# ForkJoin

> 什么是ForkJoin

ForkJoin在JDK1.7，并行执行任务！提高效率，在数据量很大的情况下。

ForkJoin会把一个大数据量的东西，拆分成小的，如果还是很大，它会继续拆分，每一个小模块都会有一个结果，最后把这些结果汇总。

> ForkJoin特点：工作窃取

多线程下如果一个线程做完了该做的，它会把其他线程没做的窃取过来自己做，不然线程等待。这里面维护的都是`双端队列`



![](https://cdn.jsdelivr.net/gh/my-zhb/CDN/img/20201221165358.png)

> ForkJoin的使用

```java
public class ForkJoinDemo extends RecursiveTask<Long> {
    private Long start;
    private Long end;
    //临界值
    private Long temp=20_0000_0000L;

    public ForkJoinDemo(Long start, Long end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        if((end-start)<temp){
            Long sum = 0L;
            for (Long i = start; i <= end; i++) {
                sum+=i;
            }
            return sum;
        }else{
            //中间值
            long middle = (start+end)>>1;
            ForkJoinDemo task1 = new ForkJoinDemo(start,middle);
            task1.fork();//拆分任务，把任务压入线程队列
            ForkJoinDemo task2 = new ForkJoinDemo(middle+1,end);
            task2.fork();
            return task1.join()+task2.join();
        }
    }
}
```

测试

```java
public class Test {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        test1();
        test2();
        test3();
    }

    public static void test1(){
        Long sum = 0L;
        long start = System.currentTimeMillis();
        for (Long i = 1L; i <= 10_0000_0000 ; i++) {
            sum+=i;
        }
        long end = System.currentTimeMillis();
        System.out.println("sum="+sum+"--->>>耗时" +(end-start));
    }

    //使用forkjoin
    public static void test2() throws ExecutionException, InterruptedException {
        long start = System.currentTimeMillis();
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        ForkJoinTask<Long> task = new ForkJoinDemo(0L,10_0000_0000L);
        ForkJoinTask<Long> submit = forkJoinPool.submit(task);
        Long sum = submit.get();
        long end = System.currentTimeMillis();
        System.out.println("sum="+sum+"--->>>耗时" +(end-start));
    }

    //Stream 并行流
    public static void test3(){
        long start = System.currentTimeMillis();
        long sum = LongStream.rangeClosed(0L, 10_0000_0000L).parallel().reduce(0, Long::sum);
        long end = System.currentTimeMillis();
        System.out.println("sum="+sum+"--->>>耗时" +(end-start));
    }
}
测试结果：
sum=500000000500000000--->>>耗时7018
sum=500000000500000000--->>>耗时6157
sum=500000000500000000--->>>耗时203
```

# 异步回调

> Future设计初衷：对将来的某个事件的结果进行建模

1. 没有返回的异步

```java
public class Demo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        CompletableFuture<Void> completableFuture = CompletableFuture.runAsync(() -> {
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("异步回调");
        });
        System.out.println("我先输出....");
        //这里会阻塞等待结果返回
        completableFuture.get();
    }
}
执行结果：
我先输出....
异步回调   
```

2. 有返回的异步

```java
public class Demo {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
            return 400;
        });
        System.out.println(
                future.whenComplete((aVoid, throwable) -> {
                    System.out.println("aVoid=>"+aVoid);//这里如果是正确就会是正确的返回值，如果错误就为null
                    System.out.println("throwable=>"+throwable);//如果有错误 这里输出错误
                }).exceptionally(e -> {
                    e.getMessage(); //可以捕获错误
                    return 500;
                }).get()
        );
    }
}
正确执行结果：
aVoid=>400
throwable=>null
400
    
错误执行结果：
aVoid=>null
throwable=>java.util.concurrent.CompletionException: java.lang.ArithmeticException: / by zero
500    
```

# JMM

> 什么是JMM（JMM就是为了保证线程安全）

JMM是Java内存模型，不存在的东西，相当于概念，约定！

**关于JMM的一些同步约定**

1、线程解锁前，`必须把共享变量立刻刷回主内存`。

2、线程加锁前，必须读取主内存当中的最新值到工作内存中！

3、加锁和解锁必须是同一把锁。

**线程：工作内存、主内存**

![](https://cdn.jsdelivr.net/gh/my-zhb/CDN/img/20201222104707.png)



> 在JMM中有8种操作

- lock(锁定)：作用于主内存的变量，把一个变量标识为线程独占状态。
- unlock(解锁)：作用于主内存的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才能被其他线程锁定。
- read(读取)：作用于主内存变量，它把一个变量从主内存传输到线程的工作内存中，以便随后的load动作使用。
- load(载入)：作用于工作内存中的变量，它把read操作从主内存中变量放入工作内存中。
- use(使用)：作用于工作内存中的变量，它把工作内存中的变量传输给执行引擎，每个虚拟机遇到一个需要使用的变量的值，就会执行这个指令。
- assign(赋值/返还)：作用于工作内存中的变量，它把一个从执行引擎接收到的值放入工作内存的变量副本中。
- store(存储)：作用于主内存中的变量，它把一个从工作内存中的变量的值传送到主内存中，以便后续write使用。
- write(写入)：作用于主内存中的变量，它把store操作从工作内存中获取的变量的值，放入主内存的变量中

>JMM对这八种指令的使用，指定了对应的规则

- 不允许read和load、store和write操作之一单独出现，即使用read就必须load，使用了store就必须write。
- 不允许线程丢弃它最近的assign操作，即工作变量的数据改变之后，必须告知主内存。
- 不允许一个线程将没用assign的数据从工作内存同步会主内存。
- 一个新的变量必须在主内存中诞生，不允许工作内存直接使用一个未被初始化的变量，就是对变量实施use、store操作之前，必须经过assign和load操作。
- 一个变量同一时间只有一个线程能对其进行lock，多次lock后，必须执行相同次数的unlock才能解锁。
- 如果对一个变量进行lock操作，会清空所有工作内存中此变量的值，在执行引擎使用这个变量前，必须重新load或assign操作初始化变量的值。
- 如果对一个变量没有被lock，就不能对其进行unlock操作，也不能unlock一个被其他线程锁住的变量。
- 对一个变量进行unlock操作之前，必须把此变量同步回主内存中。



>  问题: 假设B线程已经把值写入主内存，怎么才能告诉A线程，主内存中的值已经发生变化？

![](https://cdn.jsdelivr.net/gh/my-zhb/CDN/img/20201222112724.png)



# Volatile

> 请你谈谈你对Volatile的理解

**Volatile是Java虚拟机提供的轻量级的同步机制**

1、保证可见性

2、`不保证原子性`

3、禁止指令重排

> 保证可见性

```java
public class Demo1 {
    //不加volatile 程序就会死循环
    //加上volatile 可以保证可见性
    private volatile static int num =0 ;
    public static void main(String[] args) {
        new Thread(() -> {//对主内存的变化不知道
            while (num == 0){}
        }).start();
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        num = 1;
        System.out.println(num);
    }
}
运行结果
1
```

> 不保证原子性

原子性：要么同时成功，要么同时失败

```java
public class Demo2 {
    private volatile static int num = 0;
    private  static void add(){
        num++;
    }
    public static void main(String[] args) {
        for (int i = 1; i <= 10 ; i++) {
            new Thread(() -> {
                for (int j = 0; j < 1000 ; j++) {
                    add();
                }
            }).start();
        }
        //线程数大于2说明没有执行完,多线程下一定不要用if。
        while (Thread.activeCount()>2){
            Thread.yield();
        }
        System.out.println(num);
    }
}
运行结果 不是10000  所以不能保证原子性
```

**如何在不适用synchronized和lock的情况下保证原子性呢！**

`使用java.util.concurrent.atomic下面的原子类`，这样就保证了

```java
public class Demo2 {
    private  static AtomicInteger num = new AtomicInteger();
    private  static void add(){
        num.getAndIncrement();
    }
    public static void main(String[] args) {
        for (int i = 1; i <= 10 ; i++) {
            new Thread(() -> {
                for (int j = 0; j < 1000 ; j++) {
                    add();
                }
            }).start();
        }
        //线程数大于2说明没有执行完,多线程下一定不要用if。
        while (Thread.activeCount()>2){
            Thread.yield();
        }
        System.out.println(num);
    }
}
运行结果：
10000
```

这些类的底层都直接和操作系统挂钩！在内存中修改值，Unsafe类是一个很特殊的存在。

![](https://cdn.jsdelivr.net/gh/my-zhb/CDN/img/20201222163634.png)

> 什么是指令重排

好比你写的代码，计算机并不是按照你写的代码的顺序去执行的。

源代码--->>编译器优化重排--->>指令并行也可能重排--->>内存系统也会重排--->>执行

`volatile`可以避免指令重排

内存屏障，CPU指令。作用：

1、保证特定的操作的执行顺序！

2、可以保证某些变量的内存可见性。(利用这些特性volatile就可以实现可见性)

`加了volatile关键字的操作，都会在上方和下方形成一个屏障来保证代码的顺序，防止指令重排`

> 总结

Volatile可以保证可见性，不能保证原子性，由于内存屏障，可以避免指令重排的现象产生！

Volatile在单例模式中使用的最多。

# 单例模式

> 饿汉式

`可能会出现浪费内存的现象，因为程序一启动，就加载了`

```java
public class Hungry {

    private Hungry() {
    }

    private final  static Hungry HUNGRY = new Hungry();

    public static Hungry getInstance(){
        return HUNGRY;
    }
}
```

> 懒汉式

`懒汉式在单线程下是安全的，但是在多线程下是不安全的`

```java
public class Lazy {
    private Lazy(){
        
    }
    private static Lazy lazy;
    
    public static Lazy getInstance(){
        if(lazy == null){
            lazy = new Lazy();
        }
        return lazy;
    }
}
```

> 双重校验锁(DCL懒汉)

```java
public class Lazy {
    private Lazy() {
    }
    //此处要加volatile，来保证原子性
    // lazy = new Lazy();创建的时候会出现指令重排
   	//如果不加会出现以下情况：比如a线程正在创建，还没有创建好，b现在就来了，lazy == null判断就不会是空
    private volatile static Lazy lazy;
	
    public static Lazy getInstance(){
        if(lazy == null){
            synchronized(Lazy.class){
                if(lazy == null){
                    //不是一个原子性操作
                    lazy = new Lazy();
                     /**
                     * 创建对象的过程
                     * 1、分配内存空间
                     * 2、执行构造方法，初始化对象
                     * 3、把这个对象指向这个空间
                     */
                }
            }
        }
        return lazy;
    }
}
```

> 静态内部类

```java
public class Holder {
    private Holder(){
        
    }
    public static Holder getInstance(){
        return InnerClass.HOLDER;
    }
    public static class InnerClass{
        private static final Holder HOLDER = new Holder();
    }
}
```

> 单例模式是能被破坏的--->>枚举除外

```java
class Test{
    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
        Lazy lazy1 = Lazy.getInstance();
        //获取对象类的构造器
        Constructor<Lazy> constructor = Lazy.class.getDeclaredConstructor(null);
        //破环其私有方法
        constructor.setAccessible(true);
        Lazy lazy2 = constructor.newInstance();
        System.out.println(lazy1.hashCode());
        System.out.println(lazy2.hashCode());
    }
}
```

> 枚举也是单例模式--->>不会被反射破坏

```java
public enum EnumSigle {
    INSTANCE;
    
    public EnumSigle getInstance(){
        return INSTANCE;
    }
}
```

EnumSigle的class文件

```java
package com.se.base.demo;

public enum EnumSigle {
    INSTANCE;

    private EnumSigle() {
    }

    public EnumSigle getInstance() {
        return INSTANCE;
    }
}
```

尝试破坏枚举

`java.lang.NoSuchMethodException: com.se.base.demo.EnumSigle.<init>()在这个枚举类中没有这个空参构造`

`在EnumSigle的Class文件中出现了空参构造，但是这里报错说没有空参构造`

```java
class Test1{
    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
        Constructor<EnumSigle> constructor = EnumSigle.class.getDeclaredConstructor(null);
        constructor.setAccessible(true);
        EnumSigle enumSigle = constructor.newInstance();
        System.out.println(enumSigle);
    }
}
//执行结果
Exception in thread "main" java.lang.NoSuchMethodException: com.se.base.demo.EnumSigle.<init>()
	at java.lang.Class.getConstructor0(Class.java:3082)
	at java.lang.Class.getDeclaredConstructor(Class.java:2178)
	at com.se.base.demo.Test1.main(EnumSigle.java:21)
```

通过对`Constructor.newInstance()`的源码分析可得

如果是个枚举它会报错：`Cannot reflectively create enum objects`

```java
    @CallerSensitive
    public T newInstance(Object ... initargs)
        throws InstantiationException, IllegalAccessException,
               IllegalArgumentException, InvocationTargetException
    {
        if (!override) {
            if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
                Class<?> caller = Reflection.getCallerClass();
                checkAccess(caller, clazz, null, modifiers);
            }
        }
        if ((clazz.getModifiers() & Modifier.ENUM) != 0)
            throw new IllegalArgumentException("Cannot reflectively create enum objects");
        ConstructorAccessor ca = constructorAccessor;   // read volatile
        if (ca == null) {
            ca = acquireConstructorAccessor();
        }
        @SuppressWarnings("unchecked")
        T inst = (T) ca.newInstance(initargs);
        return inst;
    }
```

通过`javap -p EnumSigle.class`反编译还是出现了空参构造

![img](https://cdn.jsdelivr.net/gh/my-zhb/CDN/img/20201228151023.png)

通过jad软件对EnumSigle.class进行反编译成java文件查看

`jad -sjava EnumSigle.class`

```
Microsoft Windows [版本 10.0.18363.1256]
(c) 2019 Microsoft Corporation。保留所有权利。

G:\学习\基础\base\target\classes\com\se\base\demo>jad -sjava EnumSigle.class
Parsing EnumSigle.class... Generating EnumSigle.java
```

编译过后的java文件，`这里我们就可以看到一个有参的构造器了`

```java
// Decompiled by Jad v1.5.8g. Copyright 2001 Pavel Kouznetsov.
// Jad home page: http://www.kpdus.com/jad.html
// Decompiler options: packimports(3) 
// Source File Name:   EnumSigle.java

package com.se.base.demo;


public final class EnumSigle extends Enum
{

    public static EnumSigle[] values()
    {
        return (EnumSigle[])$VALUES.clone();
    }

    public static EnumSigle valueOf(String name)
    {
        return (EnumSigle)Enum.valueOf(com/se/base/demo/EnumSigle, name);
    }

    private EnumSigle(String s, int i)
    {
        super(s, i);
    }

    public EnumSigle getInstance()
    {
        return INSTANCE;
    }

    public static final EnumSigle INSTANCE;
    private static final EnumSigle $VALUES[];

    static 
    {
        INSTANCE = new EnumSigle("INSTANCE", 0);
        $VALUES = (new EnumSigle[] {
            INSTANCE
        });
    }
}

```

`重新进行破坏`

这次的报错信息符合预期

```java
class Test1{
    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
        Constructor<EnumSigle> constructor = EnumSigle.class.getDeclaredConstructor(String.class,int.class);
        constructor.setAccessible(true);
        EnumSigle enumSigle = constructor.newInstance();
        System.out.println(enumSigle);
    }
}
//执行结果 
//无法以反射方式创建枚举对象
Exception in thread "main" java.lang.IllegalArgumentException: Cannot reflectively create enum objects
	at java.lang.reflect.Constructor.newInstance(Constructor.java:417)
	at com.se.base.demo.Test1.main(EnumSigle.java:23)
```



# CAS

> 什么是CAS

比较当前工作内存中的值和主内存中的值，如果满足期望就执行操作，如果不是就一直循环！

> java层面的cas-->compareAndSet

```java
public class Test1 {
    //CAS compareAndSet:比较并交换！
    public static void main(String[] args) {
        AtomicInteger atomicInteger = new AtomicInteger(2000);
        //期望、更新,如果满足2000，就跟新成2001
        //否则不更新
        atomicInteger.compareAndSet(2000, 2001);
        System.out.println(atomicInteger.get());
        System.out.println(atomicInteger.compareAndSet(2000, 2002));
        System.out.println(atomicInteger.get());
    }
}
```

`compareAndSet底层代码`

```java
public final boolean compareAndSet(int expect, int update) {
   return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

`java无法操作内存需要通过native调用C++去操作内存，Java还可以通过Unsafe这个类去操作内存`



![image-20201228165426843](https://cdn.jsdelivr.net/gh/my-zhb/CDN/img/20201228165428.png)

`unsafe.compareAndSwapInt(this, valueOffset, expect, update)`

标准的自旋锁，不满足就一直执行

![image-20201228170726744](https://cdn.jsdelivr.net/gh/my-zhb/CDN/img/20201228170728.png)

> CAS优缺点

缺点：

- 底层是自旋锁，循环会耗时
- 一次性只能保证一个共享变量的原子性
- 会出现ABA问题

> ABA问题

ABA问题就像西游记里面的真假葫芦一样，真的已经被人掉包了，而他自己还傻傻分不清。

![](https://cdn.jsdelivr.net/gh/my-zhb/CDN/img/20201228174652.png)

```java

```



# 原子引用

> 什么是原子引用

带有版本号的原子操作，称之为原子引用

`AtomicReference`不带标记的原子引用

`AtomicStampedReference`带标记的原子引用

> 原子引用可以解决ABA问题，对应的思想就是乐观锁

```java
public class CasDemo {
    public static void main(String[] args) {
        AtomicStampedReference<Integer> stampedReference = new AtomicStampedReference<Integer>(10,1);
        new Thread(() -> {
            //获取到标识
            int stamp = stampedReference.getStamp();
            System.out.println("A1==>>"+stamp);
            //休眠2秒
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            //如果满足就更换，并且修改标识值
            //stampedReference.compareAndSet(预期值，替换的值，获取当前标识值，更换的标示值)
            boolean value1 = stampedReference.compareAndSet(10, 11, stampedReference.getStamp(), stampedReference.getStamp() + 1);
            //输出操作是否执行成功
            System.out.println("A2操作==>>"+value1);
            //输出此时的标识值
            System.out.println("A2==>>"+stampedReference.getStamp());
            //和上面同理
            boolean value2 = stampedReference.compareAndSet(11,10,stampedReference.getStamp(),stampedReference.getStamp()+1);
            //输出操作是否执行成功
            System.out.println("A3操作==>>"+value2);
            System.out.println("A3==>>"+stampedReference.getStamp());
        },"A").start();


        new Thread(() -> {
            //获取到标识
            int stamp = stampedReference.getStamp();
            System.out.println("B1==>>"+stamp);
            //休眠2秒
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            //因为这里的标识值已经发生变化了，所以这里会输出false
            boolean value = stampedReference.compareAndSet(10, 111, stamp, stamp + 1);
            //输出操作是否执行成功
            System.out.println("B2操作==>>"+value);
            System.out.println("B2==>>"+stampedReference.getStamp());
        },"B").start();
    }
}
//执行结果
A1==>>1
B1==>>1
A2操作==>>true
A2==>>2
A3操作==>>true
A3==>>3
B2操作==>>false
B2==>>3
```

# 锁的理解

> 公平锁、非公平锁

公平锁：是非常公平的，它不允许插队，必须排队。

非公平锁：是不公平的，它允许查询，锁默认值都是非公平的。

```java
	//Lock默认创建的是非公平锁
    public ReentrantLock() {
        sync = new NonfairSync();
    }
	//方法重载 可以通过参数来控制创建的是公平锁还是非公平锁
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

> 可重入锁（递归锁）

个人理解：锁中锁，如果获取到外面的锁，那么里面的锁也自动获取。

synchronized版本

```java
public class Demo1 {
    public static void main(String[] args) {
        Home1 home1 = new Home1();
    	for (int i = 1; i <= 5; i++) {
            new Thread(() -> {
                home1.getIntoHome("张三"+Thread.currentThread().getName());
            },String.valueOf(i)).start();
        }
    }
}
//家
class Home1{
    //进入家
    public synchronized void  getIntoHome(String name){
        System.out.println(name+"回家");
        getIntoBedroom(name);
    }
    //进入卧室
    public synchronized void getIntoBedroom(String name){
        System.out.println(name+"进入卧室");
    }
}
//执行结果
张三1回家
张三1进入卧室
张三5回家
张三5进入卧室
张三2回家
张三2进入卧室
张三3回家
张三3进入卧室
张三4回家
张三4进入卧室
```

Lock版本

`Lock锁必须配对，不然就会出现死锁现象，一个lock.lock()就必须要有一个lock.unlock()`

```java
public class Demo1 {
    public static void main(String[] args) {
        Home1 home1 = new Home1();
        for (int i = 1; i <= 5; i++) {
            new Thread(() -> {
                home1.getIntoHome("张三"+Thread.currentThread().getName());
            },String.valueOf(i)).start();
        }
    }
}
//家
class Home1{
    Lock lock = new ReentrantLock();
    //进入家
    public  void  getIntoHome(String name){
        lock.lock();//此处会拿到一把锁
        try {
            System.out.println(name+"回家");
            //这里也会拿到对应方法的那把锁
            getIntoBedroom(name);
        }catch (Exception e){
            e.getMessage();
        }finally {
            lock.unlock();
        }
    }
    //进入卧室
    public  void getIntoBedroom(String name){
        lock.lock();
        try {
            System.out.println(name+"进入卧室");
        }catch (Exception e){
            e.getMessage();
        }finally {
            lock.unlock();
        }
    }
}
//执行结果
张三2回家
张三2进入卧室
张三4回家
张三4进入卧室
张三1回家
张三1进入卧室
张三5回家
张三5进入卧室
张三3回家
张三3进入卧室
```

> 自旋锁

`自旋锁就是不满足条件就一直执行，直到满足条件为止`

```java
public class SpinLock {

    //string默认为空
    AtomicReference<String> atomicReference = new AtomicReference<String>();

    public void lock(){
        String name = Thread.currentThread().getName();
        System.out.println(name+"加锁中.... ");
        do {

        }while (!atomicReference.compareAndSet(null,"哈哈"));
    }

    public void unlock(){
        String name = Thread.currentThread().getName();
        System.out.println(name+"解锁中....");
        atomicReference.compareAndSet("哈哈",null);
    }
}

class Test{
    public static void main(String[] args) throws InterruptedException {
        SpinLock spinLock = new SpinLock();
        new Thread(() -> {
            spinLock.lock();
            try {
                //随眠3秒
               TimeUnit.SECONDS.sleep(3);
            }catch (Exception e){
                e.getMessage();
            }finally {
                spinLock.unlock();
            }
        },"A").start();
        //休眠1秒 保证A线程先加锁
        TimeUnit.SECONDS.sleep(1);
        new Thread(() -> {
            spinLock.lock();
            try {
                //随眠1秒
                TimeUnit.SECONDS.sleep(1);
            }catch (Exception e){
                e.getMessage();
            }finally {
                spinLock.unlock();
            }
        },"B").start();
    }
}
//执行结果
A加锁中.... 
B加锁中.... 
A解锁中....
B解锁中....
```

> 死锁

死锁就是多条线程想去操作同一资源

![](https://cdn.jsdelivr.net/gh/my-zhb/CDN/img/20201229170446.png)

死锁产生的条件:

- **互斥条件：**资源是独占的且排他使用，进程互斥使用资源，即任意时刻一个资源只能给一个进程使用，其他进程若申请一个资源，而该资源被另一进程占有时，则申请者等待直到资源被占有者释放。
- **不可剥夺条件：**进程所获得的资源在未使用完毕之前，不被其他进程强行剥夺，而只能由获得该资源的进程资源释放。
- **请求和保持条件：**进程每次申请它所需要的一部分资源，在申请新的资源的同时，继续占用已分配到的资源。
- **循环等待条件：**在发生死锁时必然存在一个进程等待队列{P1,P2,…,Pn},其中P1等待P2占有的资源，P2等待P3占有的资源，…，Pn等待P1占有的资源，形成一个进程等待环路，环路中每一个进程所占有的资源同时被另一个申请，也就是前一个进程占有后一个进程所深情地资源。 
  以上给出了导致死锁的四个必要条件，只要系统发生死锁则以上四个条件至少有一个成立。事实上**循环等待**的成立蕴含了前三个条件的成立，似乎没有必要列出然而考虑这些条件对死锁的预防是有利的，因为可以通过破坏四个条件中的任何一个来预防死锁的发生

死锁示例

```java
![2](C:\Users\July\Desktop\图片\2.png)public class Deadlock {
    public static void main(String[] args) {
        String lockA = "lockA";
        String lockB = "lockB";
        new Thread(new Dead(lockA,lockB),"T1线程").start();
        new Thread(new Dead(lockB,lockA),"T2线程").start();
    }
}

class Dead implements Runnable{
    String lockA;
    String lockB;
    public Dead(String lockA, String lockB) {
        this.lockA = lockA;
        this.lockB = lockB;
    }
    @Override
    public void run() {
        synchronized (lockA){
            System.out.println(Thread.currentThread().getName()+"==>>"+lockA+"尝试获取+"+lockB);
            //休眠2秒 保证测试正确
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized (lockB){

            }
        }
    }
}
```

> 排查死锁

1.通过jdk自带的工具命令`jps -l`排查存活的进程

![img](https://cdn.jsdelivr.net/gh/my-zhb/CDN/img/20201229165357.png)

2.通过命令`jstack 17680(进程号) `查询

![](https://cdn.jsdelivr.net/gh/my-zhb/CDN/img/20201229170050.png)

这样就能拍出到死锁的信息！
---
title: CompletableFuture详解
date: 2022/10/21 21:42
swiper: true # 是否将改文章放入轮播图中
swiperImg: 'https://zangzang.oss-cn-beijing.aliyuncs.com/img/wallhaven-wy89op.jpg' # 该文章在轮播图中的图片，可以是本地目录下图片也可以是http://xxx图片
cover: https://zangzang.oss-cn-beijing.aliyuncs.com/img/wallhaven-wy89op.jpg # 该文章图片，可以是本地目录下图片也可以是http://xxx图片
categories: JUC # 分类
tags: [ 异步 ] #标签
---

# CompletableFuture详解

## 回顾Future

因为CompletableFuture实现了Future接口所以先看一下Future

Future是Java5新加的一个接口，它提供了一种异步并行计算的功能。如果主线程需要执行一个很耗时的计算任务，我们就可以通过future把这个任务放到异步线程中执行。主线程继续处理其他任务，处理完成后，再通过Future获取计算结果。

```java
/**
 * @author ZVerify
 * @since 2022/10/21 08:32
 **/
public class FutureTest {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(10);

        UserInfoService userInfoService = new UserInfoService();
        MedalService medalService = new MedalService();
        long userId = 666L;
        long startTime = System.currentTimeMillis();

        FutureTask<UserInfo> infoFutureTask = new FutureTask<UserInfo>(() -> {
            return userInfoService.getUserInfo(userId);
        });

        executorService.submit(infoFutureTask);

        FutureTask<MedalInfo> medalInfoFutureTask = new FutureTask<>(() -> {

            return medalService.getMedalInfo(userId);

        });

        Thread.sleep(300); //模拟主线程其它操作耗时
        executorService.submit(medalInfoFutureTask);

        UserInfo userInfo = infoFutureTask.get();//获取个人信息结果
        MedalInfo medalInfo = medalInfoFutureTask.get();//获取勋章信息结果

        System.out.println("总共用时" + (System.currentTimeMillis() - startTime) + "ms");


    }
}

class UserInfoService {

    public UserInfo getUserInfo(Long userId) throws InterruptedException {
        Thread.sleep(300);//模拟调用耗时
        return new UserInfo("666", "捡田螺的小男孩"); //一般是查数据库，或者远程调用返回的
    }
}

class MedalService {

    public MedalInfo getMedalInfo(long userId) throws InterruptedException {
        Thread.sleep(500); //模拟调用耗时
        return new MedalInfo("666", "守护勋章");
    }
}

class UserInfo {
    private String name;
    private String age;

    public UserInfo() {
    }

    public UserInfo(String name, String age) {
        this.name = name;
        this.age = age;
    }
}

class MedalInfo {
    private String name;
    private String age;

    public MedalInfo() {
    }

    public MedalInfo(String name, String age) {
        this.name = name;
        this.age = age;
    }
}
```

> out --> 总共用时816ms

如果不使用Future的时候而是在主线程穿行进行，耗时为3北+5北+3北 =
11北ms，可以看到Future➕自定义线程池异步的确提高了执行效率，但是Future对结果的获取不是很友好，只能通过阻塞和轮训得到结果，

- Future.get()在没有得到结果之前一直是阻塞状态
- Future的isDone方法，可以轮询的执行

> 阻塞的方法有点违背异步编程的理念了，而且轮询会频繁的进行线程的上下文切换浪费无谓的cpu资源，所以jdk1.8提出了CompletableFuture

## 简单例子

```java
public static void main(String[]args){
        ExecutorService executorService=Executors.newFixedThreadPool(10);

        long startTime=System.currentTimeMillis();
        CompletableFuture<Void> runAsync=CompletableFuture.runAsync(()->{
        try{
        TimeUnit.MILLISECONDS.sleep(700);
        }catch(InterruptedException e){
        throw new RuntimeException(e);
        }
        System.out.println("我是臧臧");
        });
        CompletableFuture<Integer> supplyAsync=CompletableFuture.supplyAsync(()->{
        try{
        TimeUnit.MILLISECONDS.sleep(700);
        }catch(InterruptedException e){
        throw new RuntimeException(e);
        }
        System.out.println("我是臧臧");
        return 1;
        });
        System.out.println(runAsync.join());
        System.out.println(supplyAsync.join());
        System.out.println(System.currentTimeMillis()-startTime);
        }
```

> out --> 720ms
>
>这里额外说一下get()方法和join方法的区别，get()方法必须捕获或者抛出异常，而join的话不需要，因为他抛出的是uncheck异常不会强制抛出

使用CompletableFuture，代码简洁了很多。CompletableFuture的supplyAsync方法，提供了异步执行的功能，线程池也不用单独创建了。实际上，它CompletableFuture使用了默认线程池是
**ForkJoinPool.commonPool**。

CompletableFuture提供了几十种方法，辅助我们的异步任务场景。这些方法包括**创建异步任务、任务异步回调、多个任务组合处理**
等方面,下面看一下。

## CompletableFuture

### 创建异步任务

CompletableFuture创建异步任务，一般有supplyAsync和runAsync两个方法

- supplyAsync执行CompletableFuture任务，支持返回值
- runAsync执行CompletableFuture任务，没有返回值。

#### supplyAsyncfangf

```java
//使用默认内置线程池ForkJoinPool.commonPool()，根据supplier构建执行任务
public static<U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)
//自定义线程，根据supplier构建执行任务
public static<U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,Executor executor)
```

#### runAsync方法

```java
//使用默认内置线程池ForkJoinPool.commonPool()，根据runnable构建执行任务
public static CompletableFuture<Void> runAsync(Runnable runnable)
//自定义线程，根据runnable构建执行任务
public static CompletableFuture<Void> runAsync(Runnable runnable,Executor executor)
```

### 任务异步回调

![image-20221021112213194](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221021112213194.png)

#### thenRun?thenRunAsync

```java
public CompletableFuture<Void> thenRun(Runnable action);
public CompletableFuture<Void> thenRunAsync(Runnable action);
```

thenRun就是昨晚第一个任务再做第二个任务。某个任务执行完成后，执行回调方法；但是前后两个任务**没有参数传递，第二个任务也没有返回值
**

```java
public static void main(String[]args)throws ExecutionException,InterruptedException{
        CompletableFuture<Void> run=CompletableFuture.runAsync(()->{

        System.out.println("执行完第一个任务了");
        });
        run.thenRun(()->{
        System.out.println("执行完第二个任务了");
        });

        }
```

> 如果主线程执行结束，异步线程没有执行完毕直接中断执行

#### thenAccept?thenAcceptAsync

见名知意，接受，就是接受一个数据，然后操作他，跟函数式接口的Consumer消费者一样

配合supplyAsync使用

```java
public static void main(String[]args)throws ExecutionException,InterruptedException{
        CompletableFuture.supplyAsync(()->{

        return"ZVerify";
        }).thenAccept((name)->{

        if("ZVerify".equals(name))System.out.println("是我喜欢的人");
        else System.out.println("不是我喜欢的人");
        });

        }
```

> out --> 是我喜欢的人

#### thenApply?thenApplySync

和函数式接口Sfunction用法一样，thenApply方法表示，第一个任务执行完成后，执行第二个回调方法任务，会将该任务的执行结果，作为入参，传递到回调方法中，并且回调方法是有返回值的。

```java
public static void main(String[]args)throws ExecutionException,InterruptedException{

        CompletableFuture<String> future=CompletableFuture.supplyAsync(()->{
        return"ZVerify";
        }).thenApply((name)->{
        if("ZVerify".equals(name))return"是我喜欢的人";
        return"不是我喜欢的人";
        });
        System.out.println(future.join());

        }
```

> out --> 是我喜欢的人

### 捕获异常

#### exceptionally

- 当出现异常时，会触发回调方法exceptionally
- exceptionally中可指定默认返回结果，如果出现异常，则返回默认的返回结果

```java
public static void main(String[]args)throws ExecutionException,InterruptedException{

        CompletableFuture<String> future=CompletableFuture.supplyAsync(()->{
        return"ZVerify";
        }).thenApply(name->{
        return name+"123";
        }).exceptionally((e)->{
        e.printStackTrace();

        return"出现异常啦";
        });
        System.out.println(future.join());

        }
```

> 没有异常的情况下 out --> ZVerify123
>
>有异常的情况下 out --> 出现异常啦
> java.util.concurrent.CompletionException: java.lang.RuntimeException

#### whenComplete

whenComplete方法表示，某个任务执行完成后，执行的回调方法，**无返回值**；并且whenComplete方法返回的CompletableFuture的*
*result是上个任务的结果**。

如果正常执行的话就是返回上一任务的结果，如果发生异常的话返回null。

```java
public static void main(String[]args)throws ExecutionException,InterruptedException{

        CompletableFuture<String> future=CompletableFuture.supplyAsync(()->{
        return"ZVerify";
        }).thenApply(name->{
        if("ZVerify".equals(name))throw new RuntimeException();
        return name+"123";
        }).whenComplete((name,throwable)->{
        if(throwable!=null)System.out.println("出现异常了");
        else System.out.println("没有出现异常");
        });
        System.out.println(future.get());

        }
```

我的评价是不如用exceptionally直接处理异常。

#### handle

```java
public static void main(String[]args)throws ExecutionException,InterruptedException{

        CompletableFuture<Integer> handle=CompletableFuture.supplyAsync(()->{
        return"ZVerify";
        }).thenApply(name->{
        if("ZVerify".equals(name))throw new RuntimeException();
        return name+"123";
        }).handle((name,throwable)->{
        if(throwable!=null)System.out.println("出现异常了");
        else System.out.println("没有出现异常");
        return 123;
        });
        System.out.println(handle.get());

        }
```

> 出现异常也会接着执行，而且他可以转换返回值类型，无论出不出现异常都会执行

#### 区别

|             | handle() | whenComplete() | exceptionly() |
|-------------|----------|----------------|---------------|
| 访问成功        | Yes      | Yes            | No            |
| 访问失败        | Yes      | Yes            | Yes           |
| 能从失败中恢复     | Yes      | No             | Yes           |
| 能转换结果从T 到 U | Yes      | No             | No            |
| 成功时触发       | Yes      | Yes            | No            |
| 失败时触发       | Yes      | Yes            | Yes           |
| 有异步版本       | Yes      | Yes            | Yes(12版本)     |

## 多任务组合处理

#### and组合

![image-20221021193109938](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221021193109938.png)

thenCombine / thenAcceptBoth / runAfterBoth都表示：**将两个CompletableFuture组合起来，只有这两个都正常执行完了，才会执行某个任务
**。

区别在于：

- thenCombine：会将两个任务的执行结果作为方法入参，传递到指定方法中，且**有返回值**
- thenAcceptBoth: 会将两个任务的执行结果作为方法入参，传递到指定方法中，且**无返回值**
- runAfterBoth 不会把执行结果当做方法入参，且没有返回值。

```java
public static void main(String[]args)throws ExecutionException,InterruptedException{

        long startTime=System.currentTimeMillis();
        System.out.println();
        CompletableFuture<String> future=CompletableFuture.supplyAsync(()->{
        try{
        TimeUnit.MILLISECONDS.sleep(10000);
        }catch(InterruptedException e){

        }
        return"abc";
        });
        CompletableFuture<Integer> async=CompletableFuture.supplyAsync(()->{
        try{
        TimeUnit.MILLISECONDS.sleep(10000);
        }catch(InterruptedException e){

        }
        return 123;
        });
        CompletableFuture<String> combine=async.thenCombine(future,(a,b)->{
        System.out.println(a+b);
        System.out.println(System.currentTimeMillis()-startTime);
        return"good";
        });

        System.out.println(combine.get());

        }
```

> out --> 123abc
> 10012ms
> good

#### or组合

![image-20221021194301721](https://zangzang.oss-cn-beijing.aliyuncs.com/img/image-20221021194301721.png)

applyToEither / acceptEither / runAfterEither 都表示：将两个CompletableFuture组合起来，只要其中一个执行完了,就会执行某个任务。

- applyToEither：会将已经执行完成的任务，作为方法入参，传递到指定方法中，且有返回值

- acceptEither: 会将已经执行完成的任务，作为方法入参，传递到指定方法中，且无返回值

- runAfterEither： 不会把执行结果当做方法入参，且没有返回值。

```java
public static void main(String[]args)throws ExecutionException,InterruptedException{

        long startTime=System.currentTimeMillis();
        System.out.println();
        CompletableFuture<Integer> future=CompletableFuture.supplyAsync(()->{
        try{
        TimeUnit.MILLISECONDS.sleep(10000);
        }catch(InterruptedException ignored){

        }
        return 123;
        });
        CompletableFuture<Void> either=CompletableFuture.supplyAsync(()->{
        try{
        TimeUnit.MILLISECONDS.sleep(5000);
        }catch(InterruptedException ignored){

        }
        return 123;
        }).acceptEither(future,(a)->{
        System.out.println(System.currentTimeMillis()-startTime);
        });

        System.out.println(either.get());

        }
```

> out --> 5021
> null
>
>有一个注意点因为他的两个任务任意一个执行结束就执行，所以两个任务的返回值应该一样

### AllOf

所有任务都执行完成后，才执行 allOf返回的CompletableFuture。如果任意一个任务异常，allOf的CompletableFuture，执行get方法，会抛出异常

```java
public static void main(String[]args)throws ExecutionException,InterruptedException{

        long startTime=System.currentTimeMillis();
        System.out.println();
        CompletableFuture<Integer> future=CompletableFuture.supplyAsync(()->{
        try{
        TimeUnit.MILLISECONDS.sleep(10000);
        }catch(InterruptedException ignored){

        }
        return 123;
        });
        CompletableFuture<Void> either=CompletableFuture.supplyAsync(()->{
        try{
        TimeUnit.MILLISECONDS.sleep(5000);
        }catch(InterruptedException ignored){

        }
        return 123;
        }).acceptEither(future,(a)->{
        // 会先输出这里
        System.out.println(System.currentTimeMillis()-startTime);
        });
        // 然后等到上边那个睡够10秒之后执行下边
        CompletableFuture<Void> aaa=CompletableFuture.allOf(future,either).whenComplete((a,b)->System.out.println(System.currentTimeMillis()-startTime));
        aaa.get();

        System.out.println(either.get());

        }
```

### AnyOf

任意一个任务执行完，就执行anyOf返回的CompletableFuture。如果执行的任务异常，anyOf的CompletableFuture，执行get方法，会抛出异常

```java
public static void main(String[]args)throws ExecutionException,InterruptedException{

        long startTime=System.currentTimeMillis();
        System.out.println();
        CompletableFuture<Integer> future=CompletableFuture.supplyAsync(()->{
        try{
        TimeUnit.MILLISECONDS.sleep(10000);
        }catch(InterruptedException ignored){

        }
        return 123;
        });
        CompletableFuture<Void> either=CompletableFuture.supplyAsync(()->{
        try{
        TimeUnit.MILLISECONDS.sleep(5000);
        }catch(InterruptedException ignored){

        }
        return 123;
        }).acceptEither(future,(a)->{

        System.out.println(System.currentTimeMillis()-startTime);
        });

        CompletableFuture<Object> whenComplete=CompletableFuture.anyOf(future,either).whenComplete((a,b)->System.out.println(System.currentTimeMillis()-startTime));

        whenComplete.get();
        System.out.println(either.get());

        }
```


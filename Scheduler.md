# Scheduler

****

> https://www.jianshu.com/p/6d0fed7088ac

根据注释的描述：

>   /**
>   * Light-weight scheduler for running asynchronous tasks after some deadline
>   * in the future. Not terribly precise but cheap.
>   */
>  `def scheduler: Scheduler`

Scheduler是一个定时器，可以控制在一定时间delay后，以某个频率不断的执行某个任务(发送某个消息或执行某个Runnable对象)。

Akka中的Scheduler设计时，不是用于长时间段的任务调度(最长调度某个事件在8个月后执行)，并且时间精度也没有特别高。


1. Scheduler使用

使用Scheduler首先需要添加依赖：

```scala
libraryDependencies += "com.typesafe.akka" %% "akka-actor" % "2.5.17"
```

Scheduler的实现是在ActorSystem启动时加载进来的(因此也可以实现一个自己的Scheduler，只需要实现几个接口方法)。因此，Scheduler对象的创建要依赖于ActorSystem对象的创建：

```Scala
implicit val actorSystem = ActorSystem("scheduler")
val scheduler = actorSystem.scheduler
```

Scheduler的几个接口基本都是schedule方法，根据参数有所不同，完成不同的功能，比如执行发送消息、执行runnable对象、控制执行的频率以及初始执行的等待时间等：

```Scala
final def schedule(
    initialDelay: FiniteDuration,
    interval:     FiniteDuration,
    receiver:     ActorRef,
    message:      Any)(implicit
    executor: ExecutionContext,
                       sender: ActorRef = Actor.noSender): Cancellabl
```
其中，`initialDelay`表示多久之后执行这个任务；`interval`表示间隔多久再次执行任务；`receiver`表示向哪个`ActorRef`发送`message`。

`scheduler`接口都有一个隐式参数`executor`，这个隐式参数也可以通过`ActorSystem`实例变量得到：

```Scala
implicit val executionContext = actorSystem.dispatcher
```
对于返回值，Scheduler的schedule方法会返回一个Cancellable对象(或抛出IllegalStateException异常)。

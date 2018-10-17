## Dispatcher

Future接收一个隐式的ExecutionContext作为参数：

```Scala
def apply[T](body: =>T)(implicit @deprecatedName('execctx) executor: ExecutionContext): Future[T]
```

这里的ExecutionContext，我的理解就是计算所需要用到的资源，如线程，或线程池的概念。就像在MR中，任务的定语与计算资源框架的调度，是分离开的。Future的定义中，将计算任务与执行计算任务的资源分离开来：body描述计算任务，但这只是一个对任务的描述，具体怎么执行、何时执行，还是要看executor的调度。


Akka中的Dispatcher，继承自`MessageDispatcher`，这个类又实现了特质`ExecutionContextExecutor`。这个特质继承自Java的`java.util.concurrent.Executor`和Scala的`scala.concurrent.ExecutionContext`。所以这个`Dispatcher`可以作为一个`ExecutionContext`被传给Scala中的Future(或作为一个`Executor`传给Java中的Future)。

Dispatcher负责在Actor的邮箱中有消息进来时，分配线程给这个Actor。Dispatcher分配的线程，依赖于Executor来提供。Akka创建了一个默认的Dispatcher，由所有的actor共享。用户可以自行创建多个Dispatcher，并分配给不同的Actor。

### Executor

Dispatcher使用到的Executor，目前有三种：`ForkJoinPool`，`ThreadPool`和`AffinityPool`。

1. `ThreadPool`Executor有一个工作队列，队列中包含了要分配给各线程的工作。线程空闲时就会从队列中认领工作。由于线程资源的创建和销毁开销很大，`ThreadPool`允许线程的重用，可以减轻线程创建和销毁的消耗，提高Executor效率。
2. `ForkJoinPool`Executor使用一种分治算法，递归的将任务分割成更小的子任务，然后把子任务分配给不同的线程执行。然后再把结果组合起来。`ForkJoinPool`Executor是在Java7中引入的。一般来说效率会比`ThreadPool`高，是默认的选择。


### Default Dispatcher

Dispatcher由配置(文件)来管理，默认的配置放在reference.conf文件下。默认情况下，Akka会根据这个配置，创建一个默认的Disappear。配置文件由ActorSystem读取，这也是唯一的配置文件的读取入口。

```Scala
default-dispatcher {
  executor = "default-executor"
  default-executor {
    fallback = "fork-join-executor"
  }

  affinity-pool-executor {
    # 这种executor底层使用的线程池实现：akka.dispatch.affinity.AffinityPool
    # 当executor设置为affinity-pool-executor时，这部分设置才会生效
    ...
  }

  fork-join-executor {
    # 底层线程池实现：akka.dispatch.forkjoin.ForkJoinPool
    parallelism-min = 8
    parallelism-factor = 3.0
    parallelism-max = 64
  }

  thread-pool-executor {
    # 底层线程池使用： java.util.concurrent.ThreadPoolExecutor
    # 只有在设置"executor = "thread-pool-executor"时，这里的配置才会生效
    ...
  }

  shutdown-timeout = 1s
  throughput = 5
}
```

默认的Executor可以看到就是`ForkJoin` Executor，因此会使用fork-join-executor的配置。fork-join-executor的底层线程池，使用的是akka.dispatch.forkjoin.ForkJoinPool。其中的几个参数:

1. `parallelism-min`表示`fork-join-executor`管理的`ForkJoinPool`线程池中所具有的最小的线程数。
2. `parallelism-factor`定义了每个处理器的最大线程数量。比如电脑是8核的，那么计算的线程数就是8*3=24个线程。加入有50个任务需要并行计算，此时只能提供24个线程来先完成前24个任务
3. `parallelism-max`表示最大的线程数

理解这个，可以修改client端代码：

```Scala
object Tasky extends App {
  val engine = TaskExecutionEngine()
  val statuses = engine.run(1 to 100 map(_ => new WaitTask(5 seconds)) toList)
  statuses.foreach(println(_))
}
```

上面代码一次性提交100个task到`TaskExecutionEngine`。运行时可以看到，每次都是一次性执行了12个任务，其他任务在等待。12这个值是由于我的电脑有4个核，`parallelism-factor`定义为3，相乘得到。但是这个计算出来的数字，最终不会小于最小值，也不会大于最大值。

### Dispatcher的创建

Dispatcher是根据配置文件来创建的，ActorSystem是唯一一个消费这个配置的地方。可以在创建`ActorSystem`对象时，传入一个`Config`对象，或者直接使用`ConfigFactory.load()`(加载根目录下的`application.conf`, `application.json`及`application.properties`文件)。这两种方法是等效的，后一种方法，`ActorSystem`会负责将读到的配置与`reference.conf`中的配置合并起来。

要在`application.conf`中定义一个`Dispatcher`，需要指定Dispatcher的类型和Executor。还可以定义一些具体配置细节，如线程的数量、Actor一次性处理的消息数量等。配置文件`src/main/resources/application.conf`文件如下：

```Scala
task-dispatcher {
  type = "Dispatcher"
  executor = "fork-join-executor"

  fork-join-executor {
    parallelism-min = 2   # Minimum threads
    parallelism-factor = 2.0 # Maximum threas per cores
    parallelism-max = 10  # Maxmum total threads
  }
  throughput = 100        #Max messages to process in an actor before moving on
}
```

Dispatcher的类型，共有四种，默认使用`Dispatcher`。还有`PinnedDispatcher`，`CallingThreadDispatcher`等。

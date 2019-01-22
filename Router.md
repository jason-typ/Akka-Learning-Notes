## Router介绍

在Akka中，Router是一个用于负载均衡和路由的抽象。作用类似于一个Load Balancer，因此Router后面跟着一个或多个Actor，并且每个Actor都完成相同的计算任务。Router负责将收到的消息路由给Router管理的Actors。Router是一种特殊的Actor，被某个Router管理的Actor都叫做Routee。


### Router创建

Router创建可以由两种方式，一种是直接创建一个Router，一种是Router自行管理其后面的routees。
(A router can also be created as a self contained actor that manages the routees itself and loads routing logic and other settings from configuration.)
我们看后面一种

#### Router Actor
这种类型的Router，在创建时，必须要传入一个Actor Group，或者由Router创建一个Actor Pool。这对应着两种创建Router背后的Actor集合(Rootee)的机制：

- 由Router Actor创建子Actor作为其背后的routees，因此当router终止时，会自动将这些routees(子Actor)移除
- 在外部、由我们创建Actor列表(Group)，并将Actor列表传给Router

注意Group和Pool的含义。在Router创建完成后，当Router收到消息，就会将消息传递给Group/Pool中的一个或多个Actor。有多种策略可以用来决定Router选择下一个消息发送的对象。

1. 由Router自动创建Actor Pool

  这种方式创建Router，Router会自动创建子Actor作为其后面的routees。可以通过配置文件来创建：

  ```Scala
  // application.conf
  akka {
    actor.deployment {
      /router1 {
        router = round-robin-pool
        nr-of-instances = 2
      }
    }
  }

  val poolRouter1 = actorSystem.actorOf(FromConfig.props(Props[Router1]), name = "poolRouter1")
  ```

  也可以通过直接在代码里创建：

  ```Scala
  val poolRouter2 = actorSystem.actorOf(RoundRobinPool(2).props(Props[Router1]), name = "poolRouter2")
  val poolRouter3 = actorSystem.actorOf(Props(classOf[Router1]).withRouter(new RoundRobinPool(2)))
  ```

  使用Actor Pool来创建Router的方式与创建一个普通Actor的方式相同。都是使用`actorOf`方法，并传入`Props`实例作为配置。只不过创建Router时，`Props`实例需要加上Router的配置，包括路由策略，以及希望Pool中包含的Actor的数量：

2. 自己创建Actor Group

  这种方式中，需要我们在外部创建routees，并传递给创建的router。此时需要传入一个包含Actor路径的列表。可以通过在配置文件中完成：

  ```Scala
  // application.conf
  akka {
    actor {
      deployment {
        /router3 {
          router = round-robin-pool
          routees.paths = ["/user/workers/w1", "/user/workers/w2", "/user/workers/w3"]
        }
      }
    }
  }

  val router3: ActorRef = actorSystem.actorOf(FromConfig.props(), "router3")
  ```
  也可以直接在代码中指定：

  ```Scala
  def createRouter(actors: List[ActorRef]) = {
    actorSystem.actorOf(RoundRobinGroup(actors.map(actor => actor.path.toString)).props(), "router4")
  }
  ```

上面两种，一种用的是`RoundRobinPool`，一种用的是`RoundRobinGroup`。无论如何，`actorOf`方法都需要接收一个`Props`类型的实例。

`RoundRobinGroup`是一个case class，接收给定的Actor列表的路径信息作为参数：

```Scala
final case class RoundRobinGroup(
  override val paths:            immutable.Iterable[String],
  override val routerDispatcher: String                     = Dispatchers.DefaultDispatcherId)
  extends Group
```

这里的组/池似乎与策略混在了一起定义。如RoundRobinGroup或RoundRobinPool，既定义了一个组/池，也定了了组/池所适用的路由策略。

### 路由策略

Akka内置了一些路由策略。如上面使用的`RoundRobin`，会一次向组/池中的各个Actor发送消息，循环往复。

### 广播

Router允许使用广播，将一条消息发送给池/组中的每一个Actor。

```Scala
actor ! akka.routing.Broadcast(msg)
```

### 监督组/池中的对象

1. Pool方式创建
使用Pool方式创建的Router，池中所有的Actor都是由这个Router创建的，因此都是这个Router的子Actor。Router负责监督池中的所有Actor。可以在定义Router时，提供一个自定义的监督策略。

```Scala
actorSystem.actorOf(Props[MyActor].withRouter(new RoundRobinPool(8).withSupervisorStrategy(strategy)))
```
注意，所有这些都是Router的属性：`Props`的配置中指明需要有一个Router，使用`withRouter`方法，指明这个Router的属性。所以，后面的都是描述这个Router的属性。包括路由策略、池的大小，以及监督策略。

这里的设计有点难以理解，`RoundRobinPool`继承自`Pool`是个池的概念，却在池上加了一系列配置。继续看下去，会发现，`Pool`还继承自`RouterConfig`这个特质。所以Akka中其实是在使用中把概念做了简化，没有区分的那么细：池自带的配置特质。

2. Group方式创建

  Group方式创建的Router，其下所有的Rootee都是事先创建好的Actor，由他们的父Actor进行监督，所以不由Router来监督。


### ConsistentHashable

如果

## 路由策略

上面使用的`RoundRobin`，会一次向组/池中的各个Actor发送消息，循环往复。Akka内置了一些路由策略，包括

- RoundRobinRoutingLogic
- RandomRoutingLogic
- SmallestMailboxRoutingLogic
- BroadcastRoutingLogic
- ScatterGatherFirstCompletedRoutingLogic
- TailChoppingRoutingLogic
- ConsistentHashingRoutingLogic

1. RoundRobinPool and RoundRobinGroup

  这种Pool的创建示例在上面，这种策略会循环将消息发送给Pool或Group中的每一个Actor
2. RandomPool and RandomGroup

  这种策略会随机选择一个routee来发送消息。

  RandomPool使用在配置文件中指定的方式来创建：

  ```Scala
  // application.conf
  akka.actor.deployment {
    /parent/router5 {
      router = random-pool
      nr-of-instances = 5
    }
  }

  val router5: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router5")
  ```

  RandomPool使用在代码中写定的方式来创建：

  ```Scala
  val router6: ActorRef =
  context.actorOf(RandomPool(5).props(Props[Worker]), "router6")
  ```

  RandomGroup使用在配置文件中指定的方式来创建：

  ```Scala
  akka.actor.deployment {
    /parent/router7 {
      router = random-group
      routees.paths = ["/user/workers/w1", "/user/workers/w2", "/user/workers/w3"]
    }
  }

  val router7: ActorRef = context.actorOf(FromConfig.props(), "router7")
  ```

  RandomGroup使用在代码中指定的方式来创建：

  ```Scala
  val paths = List("/user/workers/w1", "/user/workers/w2", "/user/workers/w3")
  val router8: ActorRef =
    context.actorOf(RandomGroup(paths).props(), "router8")
  ```
3. BalancingPool

  这种模式中，所有的routees会共享一个邮箱，会实现负载均衡，即一个routee的负载过重，Router会将它的任务重新分派给另一个Routee
4. SmallestMailboxPool

  这种策略是优先挑选负载最小的那个routee来向其发送消息
5. BroadcastPool and BroadcastGroup

  这种策略将消息发送给每一个routee
6. ScatterGatherFirstCompletedPool and ScatterGatherFirstCompletedGroup

  这种模式下，也会将消息发送给每个routee。但router会等待第一个返回的结果，并将这个结果作为最终结果发送出去。其余的routee之后返回的结果会被丢弃
7. TailChoppingPool and TailChoppingGroup

  首先随机挑选一个routee来发送消息，等待短暂的时间后从剩下的routee中再挑选一个来发送消息，直到收到某个routee返回的结果为止。其余的结果会被丢弃掉

8. ConsistentHashingPool and ConsistentHashingGroup

  ConsistentHashingPool使用一致性哈希算法，根据收到的消息选择将消息发送给哪个routee。有三种方法来定义一致性哈希的键：

  1. 在创建Router时通过传入`hashMapping`函数参数来指定。如：

    ```Scala
    final case class Evic(key: String)

    def hashMapping: ConsistentHashMapping = {
      case Evict(key) => key
    }
    val router: ActorRef = actorSystem.actorOf(ConsistentHashingPool(5, hashMapping = hashMapping).props(Props[MyActor]), name = "router")
    router ! Evict("Hello")
    ```
  2. 使用message实现ConsistentHashable特质，这个特质需要返回其中的key

    ```Scala
    final case class Get(key: String) extends ConsistentHashable {
      override def consistentHashKey: Any = key
    }

    val router: ActorRef = actorSystem.actorOf(ConsistentHashingPool(5).props(Props[MyActor]), name = "router")

    router ! Get("Hello")
    ```
  3. 消息可以使用`akka.routing.ConsistentHashingRouter.ConsistentHashableEnvelope`来包裹，并指定key：

    ```
    final case class Evict(key: String)

    val router: ActorRef = actorSystem.actorOf(ConsistentHashingPool(5).props(Props[MyActor]), name = "router")

    router ! ConsistentHashableEnvelope(message = Entry("hello", "HELLO"), hashKey = "hello")
    ```

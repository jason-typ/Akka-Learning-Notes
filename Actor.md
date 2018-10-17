# Actor

一个Actor表示一个工作节点。每个Actor都有一个用来存放其他Actor发过来的消息的地方，类似于一个消息队列，称为邮箱。Actor同步处理邮箱中的消息：从邮箱中取出一条处理一条，处理结束后将处理结果发送给目标Actor(的邮箱)。

以上可以看到，这种消息传递模型，使得各个Actor之间具有并发性。与方法调用不同，方法调用会阻塞，知道被调用的方法返回。而消息传递模型中，消息的传递是异步的(消息的处理是同步的)，Actor处理一条消息结束后，会发送另一条消息到相应的Actor。

Actor模型的另一个好处就是：可以消除共享状态。由于Actor处理消息是同步的，也就是说，Actor每次处理一条消息，所以不会存在同时修改Actor内部状态的情况，这样就可以在Actor内部安全的保存状态

Actor模型通过监督机制来提供容错性。监督机制是把处理错误的责任交给出错对象以外的实体。在Akka中，父Actor负责监督子Actor。当子Actor出错时，默认的行为是父Actor会负责重启子Actor。





## Actor定义

1. 定义一个Actor，需要继承Actor基类：

  ```Scala
  class MyActor extends Actor;
  ```
2. 重写receive方法。
  Actor是一个Trait，自定义的Actor中需要重写receive方法。Actor Trait中的receive方法返回一个`Receive`。

    ```
    def receive: Actor.Receive
    ```
    Akka的源码中，`Receive`被定义为一个PartialFunction：

    ```
    type Receive = PartialFunction[Any, Unit]
    ```

    因此可以使用模式匹配生成的PartialFunction来定义接收到消息时的响应。

    ```Scala
    class MyActor extends Actor {
      override def receive: Receive = {
        case _ => println("Hello")
      }
    }
    ```
3. 在Actor中可以使用`sender`方法获取发送者的`ActorRef`

    ```Scala
    final def sender(): ActorRef = context.sender()
    ```
4. 使用tell方法`!`发送消息

    ```Scala
    def !(message: Any)(implicit sender: ActorRef = Actor.noSender): Unit
    ```
    tell方法有个隐式参数，ActorRef类型。每个Actor内部都有一个`self`变量，表示自身的引用。

    ```Scala
    implicit final val self = context.self
    ```

    因此在Actor内部调用tell方法，会隐式的传入自身作为发送者。而如果在Actor外部调用tell方法，会使用默认值`noSender`。总之，在scala中，消息的发送者是隐式传入的，调用者不用关心。


  在Akka中，我们只会向Actor发送消息，或通过接受消息来获取Actor的状态。

  在Akka中，使用叫做ActorRef的引用来指向Actor实例。ActorRef是一个无类型的引用，将其指向的Actor封装起来，并给上层提供了一种与Actor进行通信的机制。


  Akka中的Actor具有层级关系：新建的Actor都是作为另一个Actor的子Actor。与文件系统很类似。

  位于Actor层级结构顶端的是路径为`/`的根Actor。然后是路径为`/user`的守护Actor。使用`actorSystem.actorOf()`函数创建的Actor都是守护Actor的子Actor(`/user/yourActor`)。如果在一个Actor内部创建另一个Actor，可以使用`context().actorOf()`函数，来使得新建的Actor来成为当前Actor的子Actor(`/user/yourActor/child`)。

  根Actor下面还有其他层级结构，如`/system`下面是与系统操作相关的Actor，`/temp`下面是临时Actor。

### 监督

Akka的Actor有监督策略：父Actor负责监督子Actor，并在子Actor出错时，决定该如何处理：

  * 继续，忽略错误，继续运行
  * 停止，停止发生错误的Actor
  * 重启，关闭发生错误的Actor，并重新启动一个相同的Actor
  * 向上汇报，不知道如何处理，将错误往上面报

Actor有默认的监督策略，由`defaultStrategy`：

```Scala
def supervisorStrategy: SupervisorStrategy = SupervisorStrategy.defaultStrategy
```

要修改Actor的监督策略，需要在Actor中重写`supervisorStrategy`方法。一般情况下使用默认的行为：

* Actor在运行中抛出异常，就重启Actor
* Actor在运行中发生错误，向上一层Actor反应
* Actor初始化中发生异常，停止Actor

一旦某个消息发生了异常，该消息就会被丢弃，不会重新发送。所以消息的处理，可以放在一个消息队列中，消息处理完成后，由处理消息的Actor负责从消息队列中删除这条消息。

子Actor会默认由父Actor进行监督，但同时Actor也可以对任意其他的Actor进行监督。通过

### Actor生命周期

Actor的声明周期中会自动调用几个方法：

* preStart():在构造函数之后调用
* postStop():在重启之前调用
* preRestart():默认会调用postStop
* postRestart():默认会调用preStart

preRestart和postRestart只会在重启的时候才会被调用。当调用了这两个函数后，就不会再调用preStart和postStop了。

要终止一个Actor的运行，可以有几种方法：

- 调用`ActorSystem.stop(actorRef)`，Actor会立即停止
- 调用`ActorContext.stop(actorRef)`，Actor会立即停止
- 给Actor发送一条`PositionPill`消息，会在Actor完成消息处理后，将其停止
- 给Actor发送一条`Kill`消息，会导致Actor抛出`ActorKilledException`异常

其中`kill`消息会导致Actor抛出异常，对于这个异常怎么处理，由这个Actor的父Actor决定。


### Actor状态

Actor有时会处于无法处理某种消息的状态，如数据库服务不在线，Actor无法对数据库进行任何操作。Akka提供了一种叫做stash的机制，来将目前无法处理的消息暂存到一个独立队列中。

最常用的使用状态的情况就是判断某个服务是否离线。使Actor在在线与离线状态之间切换的几种实现方法。

1. 条件语句

最直观的实现就是将状态存放在Actor中。如：

```Scala
var online = false
def receive = {
  case x: GetRequest =>
    if(online)
      processMessage(x)
    else
      stash()
  case _: Connected =>
    online = true
    unstash()
  case _: DisConnected =>
    online = false
}
```

存放一个布尔变量用于指示服务是否在线。在收到一条Request消息时，如果服务在线，就处理，否则就暂存这条消息。在收到服务上线与下线的消息时，分别修改Actor内部保存的状态。并在服务重新上线时，将队列中保存的消息重新取出。这样在服务重新上线后，Actor就会处理所有暂存的消息。

2. 热交换

使用条件语句是一种非常过程化的处理行为和状态的方法。Akka专门提供了`become()`和`unbecome()`方法，来管理不同的行为。这两个方法位于`Actor`的`context()`中。

- become(behaviour: PartialFunction)，将`receive`块中定义的行为修改为一个新的PartialFunction
- unbecome()，这个方法将Actor的行为修改为默认行为

如：

```Scala
def receive = {
  case x: GetRequest =>
    stash(x)
  case _: Connected =>
    become(online)
    unstash
}

def online: Receive = {
  case x: GetRequest =>
    processMessage(x)
  case _: DisConnected =>
    unbecome()
}
```

对于这种二值状态，使用become和unbecome的写法，可读性更高。对于不同的状态，分别有不同的处理方式，对于offline状态来说，使用receive定义的行为：当收到GetRequest消息时，将消息暂存；当收到Connected消息时，修改为online定义的行为，并将之前暂存的消息取出。对于online状态来说，当收到GetRequest消息时，就直接处理消息。当收到Disconnected消息时，将处理消息的行为重新更改为offline状态下的处理方式。

另外，由于设计使得Actor一开始处于离线状态，所以receive函数定义的状态才会用来描述offline状态。

###### stash泄露

如果不断地缓存消息，一定会有一个时间，内存耗尽或Actor的邮箱满了。因此，无论何时使用stash，都需要设置一个时间段或最大缓存的消息数目。

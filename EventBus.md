# EventBus
***

Akka中提供了事件总线EventBus，提供了一种将消息发送给一组(感兴趣的)Actor。Akka中EventBus被定义为Trait，定义了基本的订阅、取消订阅、发布等方法：

```Scala
trait EventBus {
  type Event
  type Classifier
  type Subscriber

  def subscribe(subscriber: Subscriber, to: Classifier): Boolean
  def unsubscribe(subscriber: Subscriber, from: Classifier): Boolean
  def unsubscribe(subscriber: Subscriber)
  def publish(event: Event): Unit
}
```

一个EventBus必须定义以下3个抽象类型：
* Event为发布到该总线上的事件类型
* Classifier是选择发送给订阅者的分类器类型
* Subscriber是注册到该总线上的订阅者的类型


## EventStream

每个`ActorSystem`中都有一个`EventBus`，叫做`EventStream`。`EventStream`主要用于系统消息的传递。当然也可以被上层用户复用。`EventStream`定义为：

```Scala
class EventStream(sys: ActorSystem, private val debug: Boolean) extends LoggingBus with SubchannelClassification {
  type Event = AnyRef
  type Classifier = Class[_]

  protected implicit val subclassification = new Subclassification[Class[_]] {
    def isEqual(x: Class[_], y: Class[_]) = x == y
    def isSubclass(x: Class[_], y: Class[_]) = y isAssignableFrom x
  }

  protected def classify(event: AnyRef): Class[_] = event.getClass

  protected def publish(event: AnyRef, subscriber: ActorRef) = {
    if (sys == null && subscriber.isTerminated) unsubscribe(subscriber)
    else subscriber ! event
  }

  override def subscribe(subscriber: ActorRef, channel: Class[_]): Boolean = {
    ...
  }
  ...
```

根据注释中的描述，`EventStream`是个发布-订阅事件流，其中包括系统以及用户产生的事件。订阅者的类型是`ActorRef`，事件类型是`java.lang.Object`，频道根据`Class`类来进行分类(可以看到`classify`的实现就是`event.getClass`)。例如某个`ActorRef`订阅了`String`频道，当发布`String`类型的消息时，这个`ActorRef`就会收到消息。另外，订阅某个频道时，还会自动订阅该频道的子频道(即该类型的子类型)，因为`EventStream`实现的是`SubchannelClassification`特质。


官方文档中有一个简单的例子：

```Scala
import akka.actor.{Actor, ActorSystem, Props}

abstract class AllKindsOfMusic {
  def artist: String
}
case class Jazz(artist: String) extends AllKindsOfMusic
case class Electronic(artist: String) extends AllKindsOfMusic

class Listener extends Actor {
  override def receive: Receive = {
    case m: Jazz => println(s"${self.path.name} is listening to: ${m.artist}")
    case m: Electronic ⇒ println(s"${self.path.name} is listening to: ${m.artist}")
  }
}
object Music {
  implicit val system = ActorSystem("system")

  def main(args: Array[String]): Unit = {
    val jazzListener = system.actorOf(Props[Listener], "jazz")
    val musicListener = system.actorOf(Props[Listener], "music")
    system.eventStream.subscribe(jazzListener, classOf[Jazz])
    system.eventStream.subscribe(musicListener, classOf[AllKindsOfMusic])

    system.eventStream.publish(Electronic("Parov Stelar"))
    system.eventStream.publish(Jazz("Sonny Rollins"))
  }
}
```

运行结果为：

```Scala
music is listening to: Parov Stelar
jazz is listening to: Sonny Rollins
music is listening to: Sonny Rollins
```

从源码中可以看到`EventStream`的Classifier的类型为类类型:`Class[_]`，`classify`方法，即分类(频道)标准的实现为：

```Scala
protected def classify(event: AnyRef): Class[_] = event.getClass
```

即，会根据发送的消息的类型，发布到订阅该类型消息的Actor上。另外，由于`musicListener`actor订阅的是父类`AllKindsOfMusic`，因此发送子类的消息时，它也能收到。


### Classifiers

分类器，即决定如何形成不同的频道，定义了`Event`与`Subscriber`之间的对应关系。官网文档上介绍了几种`Classifier`的定义方式，包括“查找分类”、“子频道分类”、“扫描分类”以及“Actor”分类，分别对应了几个特质：`LookupClassification`, `SubchannelClassification`, `ScanningClassification`和`ActorClassifier`。具体如何使用这些特质，来定义分类器就先不看了。另外也可以定义自己的分类器。

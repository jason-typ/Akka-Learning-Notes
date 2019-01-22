## Actor消息传递

消息应该是不可变的：只要在线程间共享数据，首先应该考虑定义数据为不可变的格式。有两种方法可以用来定义可变消息：可变引用以及可变类型。这两种都不推荐。在scala中，一般使用case class定义，来定义一个不可变的消息。

```Scala
case class ImmutableMessage(name: String, index: Int)
```

Akka中有四种核心的Actor消息模式：tell、ask、forward和pipe。

* Ask：
  此种模式不会向发送消息的Actor的邮箱返回任何消息。
* Tell：向Actor发送一条消息。接收消息的Actor中，对消息处理时，所有发送到sender的响应都会返回给发送消息的Actor(的邮箱)
* Forward：将接收到的消息转发给另一个Actor。所有发送至sender的响应都会返回给原始消息的发送者
* Pipe

### 1. Ask模式(?)
Ask模式下，会向Actor发送一条消息，返回一个Future。当Actor返回响应时，会完成Future。Actor系统外部的普通对象与Actor进行通信时，经常会使用这种模式。

Actor系统外部的普通对象调用ask向Actor发起请求时，Actor系统会在内部自动创建一个临时的Actor，并使用这个临时的Actor完成消息传递。因此，在接收消息的Actor中能够通过sender得知发送消息的Actor是哪一个。当临时Actor接收到返回的响应后，就会使用返回的响应来完成Future。

Ask模式要求提供一个超时时间，在对方没有在一定时间内返回响应时，Future就会返回失败。在Scala中这个超时参数是隐式传入的，因此使用起来很是简洁：

```Scala
val future = actorRef ? message
```

如果只是向Actor发送ask请求，而这些Actor并不包含状态，那么此时其实就是一个Future。Actor充当了异步API的作用。

###### Ask的额外性能开销

Ask看上去很简单，但是会有额外的性能开销。首先，Ask模式会导致/tmp目录下一个临时Actor的创建，这个Actor会等待从接收ask消息的Actor那里返回的响应。其次，Future也有额外的性能开销，有临时Actor负责完成。考虑到性能的话，Tell模式是更高效的解决方式。

### 2. Tell模式(!)

Tell接收消息以及一个响应地址作为参数。接收消息的Actor中的sender，就是这个响应地址。

Scala中，如果没有显式的指定这个响应地址，默认则是发送消息的Actor本身。也可以显式指定这个响应地址(别的某个Actor)，或者指定为无发送者noSender。

建议使用Tell模式

### 3. Forward模式

Forward模式就是转发的意思。在Tell模式中，我们可以指定一个响应地址，可以是发送消息的Actor本身，也可以是另一个任意的Actor，也可以指定为没有发送者。在Forward模式中，这个响应地址就是最初发送消息的那个Actor。

Forward模式完全可以使用Tell模式来完成，使用Forward只是因为在语义上更清晰一些，代码看起来更明确一点。

### 4. Pipe

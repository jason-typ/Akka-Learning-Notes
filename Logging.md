# Logging

Akka中的日志没有绑定到一个确切的日志后端(的实现)。默认情况下，日志会被打印到标准输出，但也可以使用SLF4J或其他的日志记录器。由于日志的记录通常意味着IO以及锁的处理，通常会大大降低系统的性能。因此在Akka中，对日志的处理采用异步进行，从而将日志记录对整个系统的影响降到最低。

### 使用方法

在Akka中直接使用日志记录，首先需要添加Akka Actor的依赖：

```
libraryDependencies += "com.typesafe.akka" %% "akka-actor" % "2.5.17"
```

然后，就可以创建一个`LoggingAdapter`对象，并使用它的`error`、`warning`、`info`以及`debug`方法，完成日志记录：

```Scala
class MyActor extends Actor {
    val log = Logging(context.system, this)

    override def preStart(): Unit =  {
      log.debug("starting")
    }
    override def preRestart(reason: Throwable, message: Option[Any]): Unit = {
      log.error(reason, "Restarting due to [{}] when processing [{}]", reason.getMessage, message.getOrElse(""))
    }

    def receive = {
      case "test" => log.info("Received test")
      case _ => log.warning("Received unknown message: {}", _)
    }
  }
```

如果不想在每个Actor中都创建一个`LoggingAdapter`实例，可以定义一个trait，其中包括了`LoggerAdapter`对象，其他的Actor继承自这个trait就好。Akka提供了这个trait，叫做`ActorLogging`，其中的`LoggerAdapter`对象叫做`log`:

```Scala
class MyActor extends Actor with ActorLogging {
  override def preStart(): Unit =  {
    log.debug("starting")
  }
  ...
}
```

`LoggingAdapter`实例的创建需要用到`ActorSystem`或`LoggingBus`对象。日志记录中的占位符使用`{}`。

**Dead Letters的日志记录**

默认情况下，无法送达的消息会以`info`级别记录下来。不能根据消息无法送达，直接判定系统当前状态出现问题。但无法送达的原因应该被记录。对于这种情况，可以增加配置：

```Scala
// application.conf
akka {
  log-dead-letters = 10
  log-dead-letters-during-shutdown = on
}
```
上面配置dead letters记录总数最多为10个，在系统关闭期间的dead letters也会被记录

**其他辅助Logging配置选项**

```Scala
akka {
  loglevel = "DEBUG"

  # Log the complete configuration at INFO level when the actor system is started.
  # This is useful when you are uncertain of what configuration is used.
  log-config-on-start = on

  actor {
    debug {
      # enable function of LoggingReceive, which is to log any received message at
      # DEBUG level
      receive = on

      # enable DEBUG logging of all AutoReceiveMessages (Kill, PoisonPill etc.)
      autoreceive = on

      # enable DEBUG logging of actor lifecycle changes
      lifecycle = on

      # enable DEBUG logging of unhandled messages
      unhandled = on
    }
  }
}
```

- 以Info级别，在系统启动时，打印全部的配置
- 以debug级别，记录Actor收到的任何消息
- 以debug级别，打印Actor收到的自动产生的消息(如kill, PoisonPill等)
- 以debug级别，打印Actor的任何生命周期的变化
- 以debug级别，打印Actor没有处理的任意的消息

**关闭log**

在配置文件中关闭log：

```Scala
// application.conf
akka {
  stdout-loglevel = "OFF"
  loglevel = "OFF"
}
```
其中stdout-loglevel标签，只在系统启动和关闭时有效。

### Loggers

Akka中，将message通过event bus传递给一个Actor来处理，以此来保证日志的异步记录，以及顺序处理。这个处理消息的Actor就是一个Logger。

Akka默认的配置是：

```
akka {
  # Loggers to register at boot time (akka.event.Logging$DefaultLogger logs
  # to STDOUT)
  loggers = ["akka.event.Logging$DefaultLogger"]
  # Options: OFF, ERROR, WARNING, INFO, DEBUG
  loglevel = "DEBUG"
}
```

这个默认的Logger `DefaultLogger`定义为：

```Scala
class DefaultLogger extends Actor with StdOutLogger with RequiresMessageQueue[LoggerMessageQueueSemantics] {
  override def receive: Receive = {
    case InitializeLogger(_) ⇒ sender() ! LoggerInitialized
    case event: LogEvent     ⇒ print(event)
  }
}
```

可以看到，是直接打印到控制台，因此官网建议在production环境下不要使用

**SLF4J**

Akka同样提供了SLF4J这个Logger，需要添加依赖：

```
libraryDependencies ++= Seq(
  "com.typesafe.akka" %% "akka-slf4j" % "2.5.17",
  "ch.qos.logback" % "logback-classic" % "1.2.3"
)
```

然后在配置文件中指定使用这个Logger：

```Scala
akka {
  loggers = ["akka.event.slf4j.Slf4jLogger"]
  loglevel = "DEBUG"
  logging-filter = "akka.event.slf4j.Slf4jLoggingFilter"
}
```
其余的使用方式相同。

**JavaLogger**

还可以使用JavaLogger，同样是在配置文件中指定：

```Scala
akka {
  loglevel = DEBUG
  loggers = ["akka.event.jul.JavaLogger"]
  logging-filter = "akka.event.jul.JavaLoggingFilter"
}
```

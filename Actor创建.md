## Actor创建

创建Actor都需要去调用`actorContext.actorOf`方法，或`actorSystem.actorOf`方法。其中actorSystem创建的Actor会挂载在/user目录下，是用户可以创建的层次最高的Actor节点。

`ActorContext`和`ActorSystem`都继承自`ActorRefFactory`，这里是真正定义actorOf方法的地方。

`actorOf`方法是一个工厂方法，需要接收一个`Props`类型的实例

```Scala
def actorOf(props: Props, name: String): ActorRef
```
其中`name`就是创建的Actor的名称(path)，如：

```Scala
val myActor = actorSystem.actorOf(Props[MyActor], "myactor")
```

下面首先看一下`Props`类型。

### Props实例创建

`Props`是一个配置类，用于指定创建Actor时的各种选项。创建一个`Props`实例有几种方法：

``` Scala
val props1 = Props[MyActor]
val props2 = Props(new ActorWithArgs("arg"))
val props3 = Props(classOf[ActorWithArgs], "arg")
```

对于没有参数的Actor类，可以直接使用`Props[myActor]`来创建`Props`实例，这会调用apply方法：

```Scala
def apply[T <: Actor: ClassTag](): Props = apply(defaultDeploy, implicitly[ClassTag[T]].runtimeClass, List.empty)
```

对于有参数的Actor类，可以使用第二种或第三种方法传入类的参数。此时调用的方法最终都是Props object下面的另一个apply方法：

```Scala
def apply(clazz: Class[_], args: Any*): Props = apply(defaultDeploy, clazz, args.toList)
```
其中的`args`就是我们传入的参数(列表)。这两个`apply`函数最终都是调用到的`Props`类的主构造方法：

```Scala
final case class Props(deploy: Deploy, clazz: Class[_], args: immutable.Seq[Any])
```

官网上不推荐使用第二种方法，原因以后再看

### 创建Actor最佳实践

官网上推荐，在每个Actor的伴生对象中，提供生成`Props`类型的工厂方法。如：

```Scala
class DemoActor(magicNumber: Int) extends Actor {
  def receive = {
    case x: Int ⇒ sender() ! (x + magicNumber)
  }
}

object DemoActor {
  def props(magicNumber: Int): Props = Props(new DemoActor(magicNumber))
}
```

这样在创建`DemoACtor`的实例时，可以直接使用伴生对象的`props`方法。如：

```Scala
actorContext.actorOf(DemoActor.props(4), "demoActor")
```

好处？

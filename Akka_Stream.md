## Akka Stream

Akka Stream有三个重要的组件：`Source`、`Sink`和`Flow`。与Flume类似，`Source`就是数据来源，`Sink`就是数据目标，`Flow`就是连接`Source`和`Sink`的管道。当然，在`Flow`中也可以对数据做处理、转换。

作为流，一定有一个源，在Akka Stream中这个源叫做`Source`。`Source`本身描绘了这个源可以产生的数据，而并没有包含产生的动作。产生的动作在`Source`的各种`run`方法中。

要在一个`Source`上执行任意一个`run`方法，都需要一个`Materializer`对象。`Materializer`是一个流执行引擎的工厂，让源做出产生数据的动作都是由这个`Materializer`对象完成的。一般会将这个`Materializer`对象定义为隐式，这样，在`source`上调用`run`方法时，编译器会自动找到这个`Materialier`对象。

所有的`run`方法都带有run字眼，并且最终都代理到`Materializer`对象上，真正执行动作。

流还有一个接收(消费)数据的地方，在Akka Stream中叫做`Sink`。

在Akka Stream中，不仅Source可以被重用，包括Sink和Flow都可以被重用。akka stream中的数据组件称为数据流图(graph)，可以使用graph组件更大的graph。

Akka Stream中最简单的完整(或闭合)线性数据流就是直接把一个Source和Sink相连，如：

```
source.runForeach(i => println(i))
```




class Source[+Out, +Mat]
	第一个参数Out表示这个Source会产生的数据类型
	第二个参数Mat表示，当运行这个Source时，会产生一些辅助性的值。当没有辅助性的信息产生时，使用akka.NotUsed类型。
	Out是在流中流动的数据的类型，Mat是额外的、辅助性的数据的类型。如读取文件，有可能管道中的数据是文件的内容，但是会有额外的数据如读取成功与否、单词的个数等额外的信息。

定义一个Source，如：
	val source: Source[Int, NotUsed] = Source(1 to 10)

Source的定义只是定义了一种操作，如Spark中的LDD类似，并没有实际的数据计算。要完成数据计算，需要用到Materializer类型的变量。

```
implicit val mat = ActorMaterializer()
source.runForeach(i => println(i))(materializer)
```

查看`runForeach`方法的定义：

```
def runForeach(f: Out ⇒ Unit)(implicit materializer: Materializer): Future[Done] = runWith(Sink.foreach(f))
```
其中`Out`是定义Source时指定的，该Source会产生的数据的类型。`materializer `被定义为隐式变量，使用中可以省略。返回类型是`Future`的。`Done`只是表示一种状态，当线程跑完后，返回的实际上是`Some[Int]`。

能执行`run`的一定是一个完整的graph，可以看到在`runForeach`方法中增加了一个`Sink`，将图给补充完整了。

另外，上面的run方法实际上是在一个Actor中执行的。因此，运行后会发现，产生数据后，程序没有停止。因为Actor不会自动停止，它会保持运行状态。终止运行：

```
val done: Future[Done] = source.runForeach(i => println(i))(materializer)
implicit val ec = system.dispatcher
done.onComplete(_ => system.terminate())
```


在把`Source`或`Flow`串在一起时，产生的附属信息(叫做Materialized Value)，由最左边一个图形提供。

Akka Stream实现了流控制，所有的操作都会接受后面传来的压力：如果后面处理不过来，会通知到前面产生数据慢一些。

### source API

```Scala
val source: Source[Int, NotUsed] = Source(1 to 100)
source.runForeach(println)
val factorials = source.scan(BigInt(1))((acc, next) => acc * next)
val result: Future[IOResult] =
      factorials
      .map(num => ByteString(s"$num\n"))
      .runWith(FileIO.toPath(Paths.get("factorials.txt")))
```


```scala
def apply[T](iterable: immutable.Iterable[T]): Source[T, NotUsed]
// source的apply方法接收一个可迭代对象，待产生的数据类型可以根据给定的可迭代对象推断出来

def runForeach(f: Out ⇒ Unit)(implicit materializer: Materializer): Future[Done]
// runForeach方法接收类型为`Out => Unit`的函数，所以上面用了一个`println`

def scan[T](zero: T)(f: (T, Out) ⇒ T): Repr[T]
// scan方法接收两个参数，一个是初始值，一个是接收两个参数的函数字面量。与reduce一样，接收可迭代对象的当前值与函数字面量的上一次的计算结果作为入参。zero就是第一次给定的初始值。

def runWith[Mat2](sink: Graph[SinkShape[Out], Mat2])(implicit materializer: Materializer): Mat2
// runWith方法接收一个sink(Graph)
```

`def scan[T](zero: T)(f: (T, Out) ⇒ T): Repr[T] = via(Scan(zero, f))`

# 代理属性
有一些常见类型的属性，虽然我们可以在每次需要时去手动实现，但是更好的方式是只实现一次并对所有可用，然后封装成库。具体的例子有：

- 懒属性：他们的值只在第一次访问时才被计算；
- 可被观察的属性：监听器会接收到属性变化的通知；
- 把属性存储在 map 中，无需每个属性都有一个单独的字段。

为了覆盖这些（以及其他的）case，Kotlin 支持*代理属性*：

```kotlin
class Example {
    var p: String by Delegate()
}
```

语法是：`val/var <property name>: <Type> by <expression>`。`by` 之后的表达式是*代理*，因为属性对应的 `get()`（以及 `set()`）会被代理到它们的 `getValue()` 和 `setValue()` 方法。属性代理无需实现任何接口，但是他们必须要要提供 `getValue()` 函数（以及 `var` 变量的 `setValue()`）。例如：

```kotlin
class Delegate {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return "$thisRef, thank you for delegating '${property.name}' to me!"
    }

    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: String) {
        println("$value has been assigned to '${property.name}' in $thisRef.")
    }
}
```

当我们读取 `p` （代理到一个 `Delegate` 的实例）时，会调用到 `Delegate` 的 `getValue()` 函数，因此它的第一个参数是读取 `p` 所需的对象，第二个参数携带了 `p` 自身的描述（例如，可以获取它的名字）。例如：

```kotlin
val e = Example()
println(e.p)
```

打印结果：

```
Example@33a17727, thak you for delegating 'p' to me!
```

同样的，如果给 `p` 赋值，`setValue()` 函数也会被调用。前两个参数是一样的，第三个参数携带了将要赋的值：

```kotlin
e.p = "NEW"
```

打印结果：

```
NEW has been assigned to 'p' in Example@33a17727
```

代理对象的条件规格下面会讲到。

注意，从 Kotlin 1.1 开始，你可以在函数或者代码块内部声明一个代理属性，没必要是类成员。下面可以找到例子。

## 标准代理
Kotlin 的标准库为几种有用的代理类型提供了工厂方法。

### 懒（Lazy）
`lazy()` 是一个函数，参数是一个 lambda ，返回值是一个 `Lazy<T>` 实例，这个实例可以作为实现懒属性的代理：`get()` 的第一次调用会执行传给 `lazy()` 的 lambda 并且记录执行结果，后续的 `get()` 调用仅仅返回第一次记录的结果。

```kotlin
val lazyValue: String by lazy {
    println("computed!")
    "Hello"
}

fun main(args: Array<String>) {
    println(lazyValue)
    println(lazyValue)
}
```

打印结果：

```kotlin
computed!
Hello
Hello
```

默认情况下，懒属性的计算是**同步的**：值的计算只在一个线程里，所有线程可见的是同一个值。如果初始化代理的同步操作不是必需的，那么多个线程可以同时执行初始化，只需要把 `LazyThreadSafetyMode.PUBLICATION` 作为参数传给 `lazy()` 函数。如果初始化只会发生在一个线程里，可以使用 `LazyThreadSafetyMode.NONE` 模式，这样不会引发任何线程安全保证以及相关的开销。

### 可观察的
`Delegates.observable()` 有两个参数：初始值和变化改动的处理器。处理器的调用发生在每次给属性赋值时（在赋值执行之后）。它有三个参数：被赋值的属性，旧值和新值：

```kotlin
import kotlin.properties.Delegates

class User {
    var name: String by Delegates.observable("<no name>") {
        prop, old, new ->
        println("$old -> $new")
    }
}

fun main(args: Array<String>) {
    val user = User()
    user.name = "first"
    user.name = "second"
}
```

打印结果：

```
<no name> -> first
first -> second
```

如果想能够拦截一个赋值并且“否决”它的话，可以用 `vetoable()` 来代替 `observable()`。传入 `vetoable` 的处理器的调用发生在在新属性的赋值操作执行之前。

## 在 Map 中保存属性
一个常见使用场景是把属性的值保存在一个 map 中。经常会应用在一些像解析 JSON 或者做其他“动态”事情的应用中。这种情况下，可以使用 map 实例本身作为代理属性的代理。

```kotlin
class User(val map: Map<String, Any?>) {
    val name: String by map
    val age: Int     by map
}
```

在这个例子中，构造器接收一个 map：

```kotlin
val user = User(mapOf(
    "name" to "John Doe",
    "age" to 25
))
```

代理的属性会从 map 中取值（通过字符串型的 key —— 属性的名字）：

```kotlin
println(user.name)  // Prints "John Doe"
println(user.age)   // Prints 25
```

如果把只读的 `Map` 替换成 `MutableMap`，`var` 属性也可以这么用：

```kotlin
class MutableUser(val map: MutableMap<String, Any?>) {
    var name: String    by map
    var age: Int        by map
}
```

## 本地代理属性（从 1.1 开始）
可以把代理属性声明为局部变量。例如，可以让一个局部变量懒执行：

```kotlin
fun example(computeFoo: () -> Foo) {
    val memoizedFoo by lazy(computeFoo)

    if (someCondition && meoizedFoo.isValid()) {
        memoizedFoo.doSomething()
    }
}
```
`memoizedFoo` 变量只有在第一次访问时才会被计算。如果 `someCondition` 失败了，这个变量根本就不会被计算。

## 属性代理的要求
这里我们总结了对对象进行代理的要求：

对于**只读**的属性（也就是，`val` 型），代理需要提供一个名为 `getValue` 的函数，其参数有：

- `thisRef` —— 必须和*属性拥有者**一样或者是它的超类型（对于扩展属性，指的是被扩展的类型）；
- `property` —— 必须是 `KProperty<*>` 类型或者是它的超类型。

这个函数必须返回跟属性一样的类型（或者是它的超类型）。

对于**可变**属性（`var` 型），代理必须*另外*提供一个名为 `setValue` 的函数，参数如下：

- `thisRef` —— 和 `getValue()` 一样；
- `property` —— 和 `getValue()` 一样；
- 新值 —— 必须是属性的类型或者它的超类型。

`getValue()` 以及/或者 `setValue()` 函数的提供方式可以是代理类的成员函数或者是扩展函数。在这种情况下后者会更好用：我们需要代理一个对象的属性，它原本也没有提供这些函数。这两种函数都需要用 `operator` 关键字来标记。

代理类可以实现接口 `ReadOnlyProperty` 和 `ReadWriteProperty` 的其中之一，它们都包含了所需的 `operator` 方法。这些接口定义在 Kotlin 的标准库中：

```kotlin
interface ReadOnlyProperty<in R, out T> {
    operator fun getValue(thisRef: R, property: KProperty<*>): T
}

interface ReadWriteProperty<in R, T> {
    operator fun getValue(thisRef: R, property: KProperty<*>): T
    operator fun setValue(thisRef: R, property: KProperty<*>, value: T)
}
```

### 转换规则
底层代码中，对于每一个代理的属性，Kotlin 编译器会生成一个辅助属性并代理给它。例如，对于属性 `prop` 会生成一个隐藏的属性 `prop$delegate`，访问器的代码也只是代理到这个额外的属性上：

```kotlin
class C {
    var prop: Type by MyDelegate()
}

// this code is generated by the compiler instead:
class C {
    private val prop$delegate = MyDelegate()
    var prop: Type
        get() = prop$delegate.getValue(this, this::prop)
        set(value: Type) = prop$delegate.setValue(this, this::prop, value)
}
```

Kotlin 编译器会在参数中提供所有关于 `prop` 的必要信息：第一个参数 `this` 指向外部类 `C` 的实例，`this::prop` 是一个 `KProperty` 类型的反射对象，它描述了 `prop` 自身。

注意 `this::prop` 的用法，它可以直接在代码中指向一个**受限的可调用引用**，这个语法从 Kotlin 1.1 开始支持。

## 提供代理（从 1.1 开始）
通过定义 `provideDelegate` 操作符，我们可以扩展创建对象的逻辑，这个对象的属性实现是被代理的。如果用于 `by` 右侧的对象把 `provideDelegate` 定义为成员或扩展函数，那么这个函数就会用来创建属性代理的实例。

`provideDelegate` 可能的使用场景之一是在属性创建时检查属性的一致性，而不是只是在它的 getter 和 setter 中。

例如，如果你想在绑定之前检查属性名字，可以写成如下形式：

```kotlin
class ResourceDelegate<T>: ReadOnlyProperty<MyUI, T> {
    override fun getValue(thisRef: MyUI, property: KProperty<*>): T { ... }
}

class ResourceLoader<T>(id: ResourceID<T>) {
    operator fun provideDelegate(
        thisRef: MyUI,
        prop: KProperty<*>
    ): ReadOnlyProperty<MyUI, T> {
        checkProperty(thisRef, prop.name)
        // create delegate
        return ResourceDelegate()
    }

    private fun checkProperty(thisRef: MyUI, name: String) { ... }
}

class MyUI {
    fun <T> bindResource(id: ResourceID<T>): ResourceLoader<T> { ... }

    val image by bindResource(ResourceID.image_id)
    val text by bindResource(ResourceID.text_id)
}
```

`provideDelegate` 的参数和 `getValue` 一样：

- `thisRef` —— 必须和*属性拥有者**一样或者是它的超类型（对于扩展属性，指的是被扩展的类型）；
- `property` —— 必须是 `KProperty<*>` 类型或者是它的超类型。

在 `MyUI` 实例创建期间，每个属性都会引起 `provideDelegate` 方法的调用，并且它会立即执行必要的验证。

如果没有能力来拦截属性和它的代理之间的绑定，要实现同样的功能，必须显示地传入属性名字，这样做非常不方便：

```kotlin
// Checking the property name without "provideDelegate" functionality
class MyUI {
    val iamge by bindResource(ResourceID.image_id, "image")
    val text by bindResource(ResourceID.text_id, "text")
}

fun <T> MyUI.bindResource(
    id: ResourceID<T>,
    propertyName: String
): ReadOnlyProperty(MyUI, T> {
    checkProperty(this, propertyName)
    // create delegate
}
```

生成的代码中会调用 `provideDelegate` 方法来初始化辅助的 `prop$delegate` 属性。可以把属性声明 `val prop: Type by MyDelegate()` 所生成的代码与上面生成的代码（不提供 `provideDelegate` 方法）做个比较：

```kotlin
class C {
    var prop: Type by MyDelegate()
}

// this code is generated by the compiler
// when the 'provideDelegate' function is available:
class C {
    // calling "provideDelegate" to create the additional "delegate" property
    private val prop$delegate = MyDelegate().provideDelegate(this, this::prop)
    val prop: Type
        get() = prop$delegate.getValue(this, this::prop)
}
```

注意，`provideDelegate` 方法只会影响辅助属性的创建，并不影响为 getter 和 setter 生成的代码。

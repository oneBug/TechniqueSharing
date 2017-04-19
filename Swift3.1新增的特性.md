# Swift3.1新增的特性

> Apple最近发布了Xcode8.3, 以及Swift的一个小版本3.1。 不过不要担心, Swift3.1和Swift3是兼容的,这不会给你的Swift3项目造成太多的麻烦。不幸的是, Xcode8.3无情的去掉了对Swift2.3的支持, 所以, 如果你的项目在使用3.0之前的版本，个人建议还是不要着急更新。


## 数值类型的failable initialize

这个改动, 来自于[SE-0080](https://github.com/apple/swift-evolution/blob/master/proposals/0080-failable-numeric-initializers.md)。Swift为所有的数字类型定义了`failable initializer`, 当构造失败的时候, 就会返回`nil`。

```
Add a new family of numeric conversion initializers with the following signatures to all numeric types:
//  Conversions from all integer types.
init?(exactly value: Int8)
init?(exactly value: Int16)
init?(exactly value: Int32)
init?(exactly value: Int64)
init?(exactly value: Int)
init?(exactly value: UInt8)
init?(exactly value: UInt16)
init?(exactly value: UInt32)
init?(exactly value: UInt64)
init?(exactly value: UInt)

//  Conversions from all floating-point types.
init?(exactly value: Float)
init?(exactly value: Double)
#if arch(i386) || arch(x86_64)
init?(exactly value: Float80)
#endif
```

OK, 再让我们更直观的感受一下这个方法:

```
/// 行尾注释为运行结果
let a = 1.11
let b = Int(a)          //!< 1
let c = Int(exactly: a) //!< nil

let d = 1.0
let e = Int(exactly: d) //!< 1
```
在上面这段代码中, 我们可以看到`1.11 -> Int(exactly:) -> c`的结果为`nil`, 而`1.0 -> Int() -> e`的结果却是成功的。其实不难发现, `Int(exactly:)`比`Int()`的精度检查更加严格, 不会允许精度丢失的情况。因此`Int(exactly:)`转化时,如果丢失精度, 会返回nil。

为什么要加这个特性呢？或者说这个特性的应用场景是什么呢？SE中是这么说的:
> It is extremely common to receive loosely typed data from an external source such as json. This data usually has an expected schema with more precise types. When initializing model objects with such data runtime conversion must be performed. It is extremely desirable to be able to do so in a safe and recoverable manner. The best way to accomplish that is to support failable numeric conversions in the standard library.

这段话的大体意思就是，如果你要把一个类似Any这样的松散类型转换成数字类型的时候，像服务端返回的json数据，这个特性就会提现他的价值了。


## Sequence中新添加的两个筛选元素的方法
这个特性是根据[SE-0045](https://github.com/apple/swift-evolution/blob/master/proposals/0045-scan-takewhile-dropwhile.md)改动的。初步意愿如下：
```
Modify the declaration of Sequence with two new members:

protocol Sequence {
  // ...
  /// Returns a subsequence by skipping elements while `predicate` returns
  /// `true` and returning the remainder.
  func drop(while predicate: (Self.Iterator.Element) throws -> Bool) rethrows -> Self.SubSequence
  /// Returns a subsequence containing the initial elements until `predicate`
  /// returns `false` and skipping the remainder.
  func prefix(while predicate: (Self.Iterator.Element) throws -> Bool) rethrows -> Self.SubSequence
}
Also provide default implementations on Sequence that return AnySequence, and default implementations on Collection that return a slice.

LazySequenceProtocol and LazyCollectionProtocol will also be extended with implementations of drop(while:) and prefix(while:) that return lazy sequence/collection types. Like the lazy filter(_:), drop(while:) will perform the filtering when startIndex is accessed.
```
所添加的两个新方法如下:
* `prefix(while:)`：从第一个元素开始，将符合`while`条件的元素添加进数组A，如果`while`条件不被满足，则终止判断，并返回数组A。
*  `drop(while:)`：从第一个元素开始，跳过符合`while`条件的元素，如果`while`条件不被满足，则终止判断，并将剩余的元素装进数组返回。

具体看下面代码：
```
let arr = ["ab","ac","aa","ba","bc","bb"]

let a = arr.prefix{$0.hasPrefix("a")}
let b = arr.prefix{$0.hasPrefix("b")}
let c = arr.drop {$0.hasPrefix("a")}

print(a) //!< ["ab", "ac", "aa"]
print(b) //!< []
print(c) //!< ["ba", "bc", "bb"]
```
可以看到，`a`打印出了`["ab", "ac", "aa"]`，`arr`中第0，1，2个元素都满足while条件，但是第3个元素开始不满足条件，所以，`a`收到的赋值是第0，1，2三个元素的一个数组。`b`打印的是一个`[]`，因为`arr`中的第一个元素就不满足`while`的条件，所以判断直接终止，返回一个空数组。

`c`打印出了`["ba", "bc", "bb"]`，因为`arr`中的第0，1，2个元素都满足`while`条件，前缀包含`a`，所以都被跳过，到第3个元素的时候，`ba`的前缀并不是`a`，所以第3个元素之后的所有元素`"ba", "bc", "bb"`都被装进数组返回。
## 通过available约束Swift版本

之前版本的swift语言，如果你要控制不同版本里面的API，可能需要像下面这样声明：
```
#if swift(>=3.1)
    func test() {}
#elseif swift(>=3.0)
    func test1() {}
#endif
```
`#if`是通过编译器处理的，也就是说编译器要为每一个`if`条件独立编译一遍。也就是说如果我们的方法要在`Swift3.0` 和 `3.1`可用，那么编译器就得编译两遍。
这当然不是一个好的方法。一个更好的办法，应该是只编译一次，然后在生成的程序库包含每个API可以支持的Swift版本。

为此，在[SE-0141](https://github.com/apple/swift-evolution/blob/master/proposals/0141-available-by-swift-version.md)中，Swift对@available进行了扩展，现在它不仅可以用于限定操作系统，也可以用来区分Swift版本号了。

首先，为了表示某个API从特定版本之后才可用，可以这样：
```
@available(swift 3.1)
func test() {}
```
其次，为了表示某个API可用的版本区间，可以这样：
```
@available(swift, introduced: 3.0, obsoleted: 3.1)
func test() {}
```
## 使用具体类型在extension中约束泛型参数

在`Swift3.1`之前的版本中，如果想要在`Int?`类型添加一个方法的话，可能需要这样做：
```
protocol IntValue {
    var value: Int { get }
}
 
extension Int: IntValue {
    var value: Int { return self }
}
 
extension Optional where Wrapped: IntValue {
    func lessThanThree() -> Bool {
        guard let num = self?.value else { return false }
        return num < 3
    }
}
```
声明了一个协议，给Int写了一个扩展，就是为了给`Int?`添加`lessThanThree()`方法，很明显，这不是一个好的解决方法。

在`Swift 3.1`里，我们有更优雅的实现方法：
```
extension Optional where Wrapped == Int {
    func lessThanThree() -> Bool {
        guard let num = self else { return false }
        return num < 3
    }
}
```

## 临时转换成可逃逸的closure
在`Swift3.0`函数的`closure`类型参数默认从`escaping`变成了`non-escaping`。这很好理解，因为大多数用于函数式编程的`closure`参数的确都以`non-escaping`的方式工作。

但这样也遇到了一个问题，就是有时候，我们需要把`non-escaping`属性的`closure`，传递给需要`escaping`属性`closure`的函数。来看个例子：
```
func subValue(in array: [Int], with: () -> Int) {
    let subArray = array.lazy.map { $0 - with() }
    print(subArray[0])
}
```
注意，上面代码是不能编译通过的。因为`lazy.map()`是`escaping closure`，而`with()`是默认的`non-escaping closure`。如果根据编译器的指引，我们可能会在`with:`的后面添加`@escaping`：
```
func subValue(in array: [Int], with:@escaping () -> Int) {
    let subArray = array.lazy.map { $0 + with() }
    print(subArray[0])
}
```
这很明显不是我们想要的设计。幸运的是，`Swift3.1`给出了一个临时转换的方法：`withoutActuallyEscaping()`，我们的方法可以改写成下面这样：
```
func subValue(in array: [Int], with: () -> Int) {
   withoutActuallyEscaping(with) { (escapingWith) in
       let subArray = array.lazy.map { $0 + escapingWith() }
       print(subArray[0])
   }
}
```
`withoutActuallyEscaping`有两个参数，第一个参数表示转换前的`non-escaping closure`，第二参数也是一个`closure`，用来执行需要`escaping closure`的代码，它也有一个参数，就是转换后的`closure`。因此，在我们的例子里，`escapingWith`就是转换后的`with`。

顺带说一句，这里面使用了`array.lazy.map()`而不是`array.map()`，是因为`array.lazy.map()`会延迟实现的时间，并且按需加载，上面的例子中，只有`print(subArray[0])`使用了一次`subArray`,所以闭包里的代码只会执行一次。而用`array.map()`则会遍历整个数组，具体的差异大家自己code试验吧。

## 关于内嵌类型的两种改进
在`Swift 3.1`里，内嵌类型有了两方面变化：

* 普通类型的内嵌类型可以直接使用其外围类型的泛型参数；
* 泛型类型的内嵌类型可以拥有和其外围类型完全不同的泛型参数；

在之前的版本中，我们实现一个链表中的节点可能需要这样：
```
class A<T> {
    class B<T> {
        var value: T
        init(value: T) {
            self.value = value;
        }
    }
 }
```
但这里，就有一个问题了，在`A<T>`中使用的`T`和`B<T>`中的`T`是同一个类型么？为了避开这种歧义，在`Swift 3.1`里，我们可以把代码改成这样:
```
class A<T> {
    class B {
        var value: T
        init(value: T) {
            self.value = value
        }
    }
 }
```
这就是内嵌类型的第一个特性，尽管`B`是一个普通类型，但它可以直接使用`A<T>`中的泛型参数，此时`B.value`的类型就是`A`中元素的类型

接下来，我们再来看一个内嵌类型需要自己独立泛型参数的情况。：
```
class A<T> {
    class C<U> {
        var value: U? = nil
    }
 }
```
这就是内嵌类型的第二个改进，内嵌类型可以和其外围类型有不同的泛型参数。这里我们使用了`U`来表示`C.value`的类型。其实，即便我们在`C`中使用`T`作为泛型符号，在`C`的定义内部，`T`也是一个全新的类型，并不是`A`中`T`的类型，为了避免歧义，我们最好还是用一个全新的字母，避免给自己带来不必要的麻烦。


## 最后
感谢:
[hackingwithswift](https://www.hackingwithswift.com/swift3-1)
[泊学网](https://boxueio.com/)
提供的博客

原创作品，转载请注明出处：http://www.wzh.wiki

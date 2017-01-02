
# Swift中重写重载构造(初始化)方法

> 构造方法是一个类创建对象最先也是必须调用的方法, 在OC中, 我们更习惯称这类方法为初始化方法. OC中的初始化方法与普通方法相比,并没有太严格的分界除了要以init开头. 在Swift语言体系中，构造方法与普通的方法分界十分严格，从格式写法上就有不同，普通方法函数要以func声明，构造方法统一为init命名，不需要func关键字声明，不同的构造方法采用方法重载方式创建(***在OC中并没有重载的概念,因为OC的一个方法每增加一个参数就会变成另外一个新的方法***)。

## 一、Swift方法的重写与重载

### 重写:

*在Swift中，重写父类方法必须需要重写的方法前添加`override`关键字*
* **定义**: 方法重写顾名思义,就是对一个方法的重新实现.
* **示例**
```
/// 父类中声明一个方法
class SuperClass: NSObject {
    func sayHelloWorld() {
        print("Hello World") //!< 输出 Hello World
    }
}

/// 子类中重写父类的方法
class SubClass: SuperClass {
    override func sayHelloWorld() {
        print("Hello World I am Wang") //!< 输出 Hello World I am Wang
    }
}
```
### (方法)重载:

* **定义**:简单来说就是方法名相同,但是参数列表不同. 这样同名不同参的方法之间互相称之为重载方法.
* **示例**
```
/// 子类中重载父类的方法
class SubClass: SuperClass {
	/// sayHelloWorld(name: "WangZH")
    func sayHelloWorld(name: String) {
        super.sayHelloWorld()
        print("Hello World I am \(name)") //!< 输出 Hello World I am WangZH
    }
}
```
## 二、Swift构造方法的重写和重载

>  Swift中的构造方法分为**Designated**构造方法(指定构造方法)与**Convenience**构造方法(方便构造方法)两类。**Designated**构造方法不加任何修饰关键字，**Convenience**构造方法需要使用**Convenience**关键字进行修饰。简而言之，**Convenience**构造方法就是为了方便使用**Designated**构造方法。

### NSObject 子类构造方法的重写与重载

* NSObject子类构造方法的重写、重载和正常的方法并没有什么太大的区别，唯一值得注意的是**你要保证，在构造方法完成之前，完成所有成员常量（变量）的构造和赋值，可选值（optional）除外。**
* 以下代码并没有涉及到成员变量（常量）的初始化赋值
```
class SuperClass: NSObject {
    /// 重写父类init()方法
    override init() {
        super.init()
        print("Say Hello")
    }
}

class Subclass: SuperClass {
    /// 重载父类的init()方法
    init(name: String, word: String) {
        super.init()
        print("\(name) Say \(word)")
    }
    
    /// 创建便利初始化方法，调用了self.init(name: String, word: String)方法
    convenience init(word: String) {
        self.init(name: "WangZH", word: word)
    }
}
```
* 在子类的构造方法中改变父类的变量值
```
class SuperClass: NSObject {
    /// 声明一个number变量
    var number: Int
    override init() {
        /// 初始化完成之前给number赋值
        number = 0
        super.init()
        print("Say Hello")
    }
}

class Subclass: SuperClass {
    /// 声明一个变量numberStr
    var numberStr: String
    init(name: String, word: String) {
        /// 初始化之前完成赋值
        numberStr = "1"
        super.init()
        /// 初始化完成之后修改父类的变量值
        number = 1
        print("\(name) Say \(word)")
    }
}
```
*在子类的构造方法中改变父类的变量值，需要父类初始化完成之后才可以更改，需要注意的是**如果在这个构造方法中还需要给自己的变量赋值，则需要放在父类初始化完成之前。***

### UIViewController子类构造方法的重写与重载

* UIViewController子类构造方法的重写、重载和NSObject子类的重写、重载基本上相同，但是有一些需要注意的地方。
先看以下代码:
```
class ViewController: UIViewController {
    /// 重写父类的init(nibName nibNameOrNil: String?, bundle nibBundleOrNil: Bundle?)方法
    override init(nibName nibNameOrNil: String?, bundle nibBundleOrNil: Bundle?) {
        super.init(nibName: nibNameOrNil, bundle: nibBundleOrNil)
    }
    
    /// 重载父类的init()
    init(name: String) {
        super.init(nibName: nil, bundle: nil)
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
}
```
上面代码重写和重载了父类的构造方法，大家有没有发现重载父类方法的时候`super`调用的是`init(nibName:  nibName,  bundle: bundle)`方法?这是为什么呢？ 苹果官方是这么说的：

> The designated initializer. If you subclass UIViewController, you must call the super implementation of this
method, even if you aren't using a NIB.  (As a convenience, the default init method will do this for you,
and specify nil for both of this methods arguments.) In the specified NIB, the File's Owner proxy should
have its class set to your view controller subclass, with the view outlet connected to the main view. If you
invoke this method with a nil nib name, then this class' -loadView method will attempt to load a NIB whose
name is the same as your view controller's class. If no such NIB in fact exists then you must either call
-setView: before -view is invoked, or override the -loadView method to set up your views programatically.
`public init(nibName nibNameOrNil: String?, bundle nibBundleOrNil: Bundle?)`
    
这段话的大体意思就是VC的`init()`是一个*Convenience*类型的构造方法，它会帮你把`public init(nibName nibNameOrNil: String?, bundle nibBundleOrNil: Bundle?)`方法的两个参数都设置成nil。也就是说，如果你要重载UIViewController的`init()`方法，就需要手动将上面方法的两个参数置为`nil`。修改后`init()`重载的代码如下：
```
class ViewController: UIViewController { 
    /// 便利构造方法
    convenience init(name: String) {
    /// 根据Swift构造方法的特点，如果子类没有定义任何*Designated*类型的构造方法，则默认继承父类所有的*Designated*构造方法。
        self.init(nibName: nil, bundle: nil)
    }    
}
```
当然，还可以这么写：
```
class ViewController: UIViewController {
    convenience init(name: String) {
    /// 其实和上面的原理是一样的，UIViewController的init()方法把init(nibName: nib, bundle: bundle)的两个参数默认设置成了nil, 当然，这样看起来简洁一些
        self.init()
    }
}
```

OK，说完重载再回过头来说一下构造方法的重写，看下面代码：
```
class ViewController: UIViewController {
	/// 重写方法
    override init(nibName nibNameOrNil: String?, bundle nibBundleOrNil: Bundle?) {
        super.init(nibName: nibNameOrNil, bundle: nibBundleOrNil)
    }
    /// UIViewController中标记子类必须要实现的方法
    required init?(coder aDecoder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
}
```
上面代码重写了UIViewController的`init(nibName nibNameOrNil: String?, bundle nibBundleOrNil: Bundle?)`方法，编译器没有报错，运行没有问题，那么这个就可以跳过了。等等，下面的`required init?(coder aDecoder: NSCoder)`方法是什么鬼啊？Ok，先来说一下`required`这个修饰符，`required`修饰的方法要求子类**必须实现**，如果子类不实现，编译是不会通过的。那么问题又来了，我们平常用`ViewController`的时候也没有实现这个方法啊，程序跑的也很欢快啊，这是怎么回事呢？因为Swift构造方法的特点之一就是：**"如果子类没有定义任何 **Designated** 类型的构造方法，则默认继承父类所有的 **Designated** 构造方法"**。这下就明了吧，不是子类没有实现，只不过是继承了父类的实现罢了。那么接下来就要说一下UIView的构造方法了。再等等，可是为什么**重写了方法**之后就需要子类来实现了呢？答案还是在Swift构造方法的特点里：**如果子类重写了父类的某一构造方法，则默认不在继承父类所有的构造方法。对于Designated类型的构造方法，子类重写了哪些，哪些才能够被使用。对于Convenienve类型的构造方法，需要在子类的重写方法中调用其依赖的父类Designated构造方法，代码如下:**
```
class SuperClass: NSObject {
    /// 父类convenience方法依赖的designated方法
    init(name: String, word: String) {
        super.init()
    }
    
    convenience init(word: String) {
        self.init(name: "wangzh", word: word)
    }
    
}

class SubClass: SuperClass {
   
    /// 重写父类的convenience方法
    init(word: String) {
        /// 调用父类convenience依赖的designed方法
        super.init(name: "哈哈", word: "ha")
        
        /// 这种调用是不会被编译通过的
        super.init(word: word)
        
    }
    
    func test() {
        let subClass = SubClass.init(word: "哈哈")
        print(subClass)
    }
}
```

### UIView子类构造方法的重写与重载

> UIView的构造方法其实就和UIViewController里面的差不多。可以这么说，UIKit里面构造方法的重写和重载都和上面说的UiViewController的差不多，这里就不多赘述了。


## 感谢：
[珲少](https://yq.aliyun.com/articles/39484) 
[tiborbodecs](https://theswiftdev.com/2015/08/05/swift-init-patterns/) 分享的文章

原创作品，转载请注明出处：http://www.wzh.wiki

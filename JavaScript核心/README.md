# JavaScript 核心

> 原文：[JavaScript. The Core](http://dmitrysoshnikov.com/ecmascript/javascript-the-core/)  
> 作者：[Dmitry Soshnikov](http://dmitrysoshnikov.com/)
> 第二版：[JavaScript. The Core: 2nd Edition](http://dmitrysoshnikov.com/ecmascript/javascript-the-core-2nd-edition/)

### 目录

*   1.对象

*   2.原型链

*   3.构造器

*   4.运行栈

*   5.运行上下文

*   6.变量

*   7.激活

*   8.作用域

*   9.闭包

*   10.this

*   11.结论

本文是ECMA-262-3规范系列的概述和摘要。每个章节都包含对应匹配章节的引用，以便您可以阅读以获取更深入的理解。

面向读者：有经验的开发者，专家。

我们从一个对象的概念触发，这是ECMAScript的基础。

## 对象

ECMAScript是一门高度抽象的、面向对象的语言，它处理对象。还有原始值，但是在需要的情况下也会转换成对象。

> 对象是一个属性的集合并具有单个原型对象，原型对象可能是另一个对象或者null值。

我们来看一个对象的简单例子，一个对象的原型被对象上的内部属性[[Prototype]]引用。然而，在图中的们将使用`__<internal-property>__`下划线表示法而不是双括号，特别是对原型对象：`__proto__`。

有如下代码：

``` javascript
var foo = {
  x: 10,
  y: 20
}
```

我们有一个拥有两个显式自有属性和一个隐式`__proto__`属性的对象，它是`foo`的原型的引用。

![Figure 1. A basic object with a prototype.](http://dmitrysoshnikov.com/wp-content/uploads/basic-object.png)



<p align="center">图 1. 一个带原型的基本对象</p>
这些原型需要什么？让我们考虑下原型链的概念来回答这个问题。



## 原型链

原型对象也只是简单的对象，可能有自己的原型。如果一个原型在它的`prototype`上有一个非空的引用，亦或者有多个，这就称为原型链。

> 一条原型链是有限个对象的链接关系，原型链常被用于实现继承和属性共享。

考虑一下这种情况，当我们有两个对象，他们仅仅在小部分上有不同的地方，其他全部都是相同的。很明显，对于一个设计良好的系统，我们乐意去复用那些相似的功能或者代码而不是在每个对象中重复它们。在基于类的系统中，这种代码复用学说被称为`基于类的继承`—你将相似的功能放到类A里，提供继承自类A的类B和类C，类B和类C有自己额外的小改动。

ECMAScript没有类的概念。然而，代码复用学说没有太多不同（尽管在某些种程度上比类更加灵活）并通过原型链实现。这种继承称为基于委托的继承（或者和ECMAScript相近：基于原型的继承）。

相似性如例子中的类A、B和C，用ECMAScript创建对象a、b和c。这样，对象a存储了对象b和c的公共部分。b和c仅存储它们额外的属性或方法。

```javascript
var a = {
  x: 10,
  calculate: function(z) {
    return this.x + this.y + z;
  }
};

var b = {
  y: 20,
  __proto__: a
};

var c = {
  y: 30,
  __proto__: a
};

// 调用继承方法
b.calculate(30); // 60
c.calculate(40); // 80
```

够简单吧？我们可以看到b和c访问了定义在对象a上的`calculate`方法。这是通过原型链实现的。

这个规则很简单：如果属性a或方法a在对象内找不到（即这个对象没有这个自有属性），然后就会尝试在原型链上找这个属性或方法。如果属性在原型上找不到，就会考虑在原型的原型上去找，即整个原型链去搜寻（和基于类继承完全相同，解析继承的时候会遍历类链）。首次被找到的同名属性或方法会被拿来用。这样，找到的属性称为继承属性。如果整个原型链都找不到这个属性，就会返回undefined。

注意，this值在使用继承方法时会被设置成原始对象，而不是找到该方法的原型对象。即如上面的例子中`this.y`是取自b和c而不是a。然而`this.x`是两次通过原型链机制取自a。

如果一个对象的原型没有被显示指定，`__proto__`的默认值就会设置成`Object.prototype`。对象`Object.prototype`本身也有一个`__proto__`属性，在原型链末端是`null`值。

下图展示了对象a、b和c的继承体系。

![Figure 2. A prototype chain.](http://dmitrysoshnikov.com/wp-content/uploads/prototype-chain.png)



<p align="center">图 2.原型链</p>
注意：ES5提供了一个标准化的可替代原型继承的方式：使用`Object.create`方法

```javascript
var b = Object.create(a, {y: {value: 20}});
var c = Object.create(a, {y: {value: 30}});
```

你可以在[对应章节](http://dmitrysoshnikov.com/ecmascript/es5-chapter-1-properties-and-property-descriptors/#new-api-methods)中查看更多ES5的新API。

尽管ES6标准化了`__proto__`，它仍然可以用来初始化对象。

这在让对象拥有相同或相似的状态结构（即相同的属性集）和不同的状态值的情况下通常是需要的。这种情况我们可以用指定模式的构造函数去生产对象。

## 构造器

除了通过指定模式创建对象，构造函数还做了其他一件有用的事 —— 自动给新创建的对象设置原型对象。这个原型对象存在存储在构造函数的`prototype`属性上。

例如，我们可以用构造函数重写前一个例子的b和c。这样对象a的扮演了Foo的`prototype`角色：

```javascript
// 构造函数
function Foo(y) {
  // 可以通过具体模式创建对象：创建后有自由属性"y"
  this.y = y;
}

// Foo.prototype储存了新建对象的原型
// 如此我们可以用它定义共享/继承属性或方法

// 和前一个例子一样有一个继承属性"x"
Foo.prototype.x = 10;
 
// 以及继承方法"calculate"
Foo.prototype.calculate = function (z) {
  return this.x + this.y + z;
};
 
// 再用Foo模式创建b和c
var b = new Foo(20);
var c = new Foo(30);
 
// 调用继承方法
b.calculate(30); // 60
c.calculate(40); // 80
 
// 看下如我们所期待的引用属性
 
console.log(
 
  b.__proto__ === Foo.prototype, // true
  c.__proto__ === Foo.prototype, // true
 
  // Foo.prototype自动创建了"constructor"属性，
  // 该属性是构造函数自身的引用
  // 实例b和c可以通过委托找到构造器并检查它
 
  b.constructor === Foo, // true
  c.constructor === Foo, // true
  Foo.prototype.constructor === Foo, // true
 
  b.calculate === b.__proto__.calculate, // true
  b.__proto__.calculate === Foo.prototype.calculate // true
);
```

这些代码可以展示为如下关系：

![Figure 3. A constructor and objects relationship.](http://dmitrysoshnikov.com/wp-content/uploads/constructor-proto-chain.png)

<p align="center">图 3. 构造函数和对象关系</p>

图片再次展示了每个对象都有一个原型。构造方法Foo有它自己的`__proto__`是`Function.prototype`，其又通过`__proto__`属性再次引用到`Object.prototype`，如此反复，`Foo.prototype`只是Foo的一个显式属性，是对象b和c原型的引用。

形式上，如果要考虑分类的概念（刚才我们已经分类的那个新分离的东西-Foo），构造函数和原型对象的组合可以被称为“类”。事实上，例如Python的第一类动态类具有完全相同的属性/方法解析实现。从这个角度看，Python类知识ECMAScript中使用的基于委托的继承的语法糖。

注意：ES6中类的概念已经被标准化，并且如上述在构造函数之上完全实现为语法糖。从这个角度看，原型链是类继承的实现细节：

```javascript
// ES6
class Foo {
  constructor(name) {
    this._name = name;
  }
 
  getName() {
    return this._name;
  }
}
 
class Bar extends Foo {
  getName() {
    return super.getName() + ' Doe';
  }
}
 
var bar = new Bar('John');
console.log(bar.getName()); // John Doe
```

有关该主题的完整详细说明，请参阅ES3系列的第7章。有两部分：

第7.1章 OOP。常规理论上，您将在其中找到各种OOP范例和文学描述，以及它们和ECMAScript和第7.2章的比较。

OOP.ECMAScript实现是专门用于ECMAScript中的OOP

> OOP: Object Oriented Programming，面向对象编程

现在，当我们了解基本对象方面时，让我们看看ECMAScript中运行时程序是如何实现的。也就是所谓的执行上下文堆栈，每个元素都可以抽象地表示为对象。确实，ECMAScript几乎在任何地方都以对象的概念运作。

## 执行环境堆栈

ECMAScript中有代码有三种类型：全局代码、函数代码和eval代码。每种代码都在其执行环境中进行评估。只有一个全局环境，但可能有很多函数或eval执行环境。每次调用函数，进入函数执行环境并评估执行函数代码类型。每次调用eval函数，都会进入eval执行环境评估执行其代码。

注意，一个函数可能会生成无限的环境集，因为每次调用函数（即使是递归调用自身）都会产生一个带有新上下文状态的环境：

```javascript
function foo(bar) {}

// 调用相同函数，每次调用以不同的上下文状态（如argument中的"bar"的值）生成三个不同的上下文
foo(10);
foo(20);
foo(30);
```

一个执行环境可以激活另一个环境，例如，函数调用另一个函数（或者全局函环境下调用全局函数），等等。逻辑上，这是作为堆栈实现的，称为执行环境堆栈。

激活另一个环境的环境称为调用者。正在激活的环境称为被调用者。被调用者此时也可能是一个调用者（例如，从全局环境调用的函数，该函数调用一些内部函数）。

当调用者激活（调用）一个被调用者时，调用者暂停其执行并将控制流传递鬼被调用者。被调用者被推入堆栈并成为正在运行（激活的）执行环境。在被调用者的环境结束后，控制权回交给调用者，调用者的环境代码继续评估执行（也可以激活其他环境）直到结束，以此类推。被调用者可能只是简单返回或异常退出。抛出一个未捕获的异常可能退出（从堆栈中弹出）一个或多个环境。

即所有ECMAScript程序运行时都表示为执行环境（EC）堆栈，其中栈顶是激活的环境：

![å¾4.æ§è¡ä¸ä¸æå æ ã](http://dmitrysoshnikov.com/wp-content/uploads/ec-stack.png)

<p align="center">图 4. 执行上下文堆栈。</p>

当程序开始时，它进入全局环境，是栈的底部即第一个元素。然后全局代码提供一些初始化操作，创建需要的对象和函数。在执行全局环境期间，代码可以激活一些其他（已创建的）函数，该函数进入其执行环境，将新元素推到堆栈，以此类推。初始化完成后，运行时系统正在等待一些事件（如用户的鼠标点击），这将激活某些功能并进入新的执行环境。

在下图中，将一些函数环境作为`EC1`，全局环境作为`Global EC`，我们在进入和退出`EC1`时，全局环境有以下堆栈变化。

![å¾5.æ§è¡ä¸ä¸æå æ çæ´æ¹ã](http://dmitrysoshnikov.com/wp-content/uploads/ec-stack-changes.png)

<p align="center">图 5.执行上下文堆栈的变化</p>

这就是ECMAScript的运行时系统如何管理代码的执行。

有关ECMAScript中执行环境的更多信息，请参阅相应的[第一章 执行环境](http://dmitrysoshnikov.com/ecmascript/chapter-1-execution-contexts/)。

正如我们所说，堆栈中的每个执行环境都可以表示为一个对象。让我们看看一个环境执行它的代码需要什么样的结构和状态（其属性）类型。

## 执行环境

一个执行环境可以抽象地表示为一个简单对象。每个执行环境都有一组属性（我们称环境状态）以跟踪其关联代码的执行进度。在下图中，展示了环境的结构：

![å¾6.æ§è¡ä¸ä¸æç»æ](http://dmitrysoshnikov.com/wp-content/uploads/execution-context.png)

<p align="center">图 6.执行环境结构</p>

除了这三个需要的属性（变量对象，作用域链和this值）之外，执行环境可以具有任何其他状态，具体取决于实现。

让我们详细考虑一下环境的这些重要属性。

## 变量对象

> 一个变量对象是关联执行环境数据的容器。它是一个特殊的对象，存储在环境中定义的变量和函数声明。

注意，函数表达式（与函数声明不同）不包含在变量对象中。

变量对象是一个抽象的概念。在不同的环境类型中，物理上，它使用不同的对象呈现。例如，在全局环境中，变量对象是全局对象自身（这就是为何我们能通过全局对象的属性名去引用全局变量）。

让我们在全局执行环境中考虑一下例子：

```javascript
var foo = 10;
 
function bar() {} // function declaration, FD (函数声明)
(function baz() {}); // function expression, FE (函数表达式)
 
console.log(
  this.foo == foo, // true
  window.bar == bar // true
);
 
console.log(baz); // ReferenceError, "baz" is not defined
```

然后全局环境变量（VO）将有如下属性：

![å¾7.å¨å±åéå¯¹è±¡ã](http://dmitrysoshnikov.com/wp-content/uploads/variable-object.png)

<p align="center">图 7.全局变量对象。</p>

再看一次，`baz`作为函数表达式的函数不包含在变量对象中。这就是我们尝试在函数本身之外访问它却得到`ReferenceError`的原因。

注意。这与其他语言（如C/C++）不同，ECMAScript中，只有函数创建一个新的作用域。定义在一个函数作用域内的变量和内部函数在外部不是直接可见的，也不会污染全局变量对象。

使用`eval`我们也会进入一个新的执行环境（`eval的`）。但是`eval`也使用全局变量对象或者调用者的变量对象（如从函数中调用`eval`）。

那么函数及其变量对象呢？？在函数环境中，变量对象被表示为一个激活对象。

## 激活对象

当一个函数被调用时，会创建一个称为激活对象的特殊对象。它被形参和特殊的`arguments`对象（形参的映射，具有索引属性）填充。然后激活对象被用作函数环境的变量对象。

即函数的变量对象同样是简单变量对象，但变量和函数声明之外，还存储了形参和`arguments`对象，称激活对象。

考虑如下例子：

```javascript
function foo(x, y) {
  var z = 30;
  function bar() {} // FD
  (function baz() {}); // FE
}
 
foo(10, 20);
```

我们有foo函数环境的下一个激活对象:

![å¾8.æ¿æ´»å¯¹è±¡ã](http://dmitrysoshnikov.com/wp-content/uploads/activation-object.png)

<p align="center">图 8.激活对象。</p>

并且函数表达式`baz`也不包含在变量（激活）对象中。

有关此主题的所有细微情况（如变量和函数的声明提升）的完整描述可以再同一名称中找到。[第2章 变量对象](http://dmitrysoshnikov.com/ecmascript/chapter-2-variable-object/)。

>  注意，在ES5中，变量对象和激活对象的概念被合并到词法环境模型中，详细描述可以在[对应的章节](http://dmitrysoshnikov.com/ecmascript/es5-chapter-3-2-lexical-environments-ecmascript-implementation/)中找到。

我们正在进入下一部分。众所周知，在ECMAScript中我们可以使用内部函数，在这些内部函数中，我们可以引用父函数的变量或全局环境的变量。当我们将变量对象命名为环境中的作用域对象，和上面讨论的原型链类似，这是所谓的作用域链。

## 作用域链

> 作用域链是一个对象列表，在环境的代码中搜索出现的标识符。

该规则也和原型链一样简单：如果在自己的范围（自己的变量/激活对象）中找不到变量，则在父变量对象上查找，以此类推。

对于环境，标识符是：变量名、函数声明、形参等。当函数在内部代码中引用的不是局部变量（或内部函数或形参）的标识符时，这种变量被称为自由变量，为了准确查找这些自由变量，作用域链就派上用场了。

一般情况下，作用域链是所有这些父变量对象的列表，加上（在作用域前的）函数自己的变量/激活对象。但是，作用域链还可以包含任何其他对象，例如环境执行期间动态添加到作用域链的对象，例如`with对象`或`catch分句`的特殊对象。

在解析（查找）标识符时，从激活对象开始搜索作用域链，然后（如果在自己的激活对象中找不到标识符）直到作用域链的顶端 - 重复如此，也仅是和原型链相似罢了。

```javascript
var x = 10;
 
(function foo() {
  var y = 20;
  (function bar() {
    var z = 30;
    // "x" 和 "y" 是自由变量
    // 在bar的作用域(bar的激活对象)之后被找到
    // 作用域链是： bar -> foo -> global
    console.log(x + y + z);
  })();
})();
```

我们可以假定作用域链对象的联系通过隐藏属性`__parent__`，引用到作用域链的下一个对象。这种方法可以在[真正的Rhino代码](http://dmitrysoshnikov.com/ecmascript/chapter-2-variable-object/#feature-of-implementations-property-__parent__)中进行测试，并且恰好这种技术被用于[ES5词法环境](http://dmitrysoshnikov.com/ecmascript/es5-chapter-3-2-lexical-environments-ecmascript-implementation/)（称为外部链接）。作用域链的另一种表示可以是简单的数组。使用一个`__parent__`概念，我们可以用下图表示上面的例子（因此父变量被保存在函数的`[[Scope]]`属性中）：

![å¾9.èå´é¾ã](http://dmitrysoshnikov.com/wp-content/uploads/scope-chain.png)

<p align="center">图 9.作用域链</p> 

在代码执行时，可以使用`with`语句和`catch`从句对象来扩充作用域链。由于这些对象是简单的对象，它们可能有原型（和原型链）。这一事实导致作用域链查找的二维的：（1）首先考虑作用域链，（2）然后在每个作用域链的链接上，深入原型链（如果链接有原型的情况）。

对于这个例子：

```javascript
Object.prototype.x = 10;
 
var w = 20;
var y = 30;
 
// 在 SpiderMonkey 全局对象中
// 如全局环境的变量对象继承自"Object.prototype"
// 则我们可以引用到一个未定义的变量x，它会在原来链上被查找到
console.log(x); // 10
 
(function foo() {
  // foo的本地变量
  var w = 40;
  var x = 100;
  
  // x在Object.prototype中被查找到
  // 因为{z: 50} 继承了它
  with ({z: 50}) {
    console.log(w, x, y , z); // 40, 10, 30, 50
  }
 
  // 在with对象在原型链上移除后，
  // x再次在foo环境的激活对象被查找到
  // w也是局部的
  console.log(x, w); // 100, 40
 
  // 这是我们在浏览器宿主环境中显式引用全局w变量的方法
  console.log(window.w); // 20
})();
```

我们有以下结构（也就是说，我们转到`__parent__`之前，优先考虑`__proto__`这个原型链）：

![å¾10.âwith-augmentedâèå´é¾ã](http://dmitrysoshnikov.com/wp-content/uploads/scope-chain-with.png)

<p align="center">图 10.扩展作用域链</p>

注意，并非所有实现的全局对象都继承自`Object.prototype`。图中描述的行为（引用来自全局环境的未定义的x变量）是可测试的，如在SpiderMonkey中。

在所有父变量对象存在之前，在函数内部获取父数据没啥特别的 — 我们展示遍历作用域链去解析（查找）需要的变量。但是，和上面我们提到的，在一个环境结束后，它全部的状态和自身都已被摧毁了，同时内部函数从父函数中返回。此外，这个已返回的函数之后可能会被另一个环境激活，如果一些自由变量的环境已经消失，这种激活会是什么？在一般理论中，有助于解决这个问题的是（词法上）闭包的概念，在ECMAScript中这是和作用域链直接关联的。

## 闭包

在ECMAScript中，函数是第一类对象。这个术语意味着函数可以作为参数传递给其他函数（这种情况下，它们被称为`funargs`，是`function arguments`的简称）。接收`funargs`的函数称为高阶函数，或者更贴近数学地说叫运算符。从其他函数返回的函数称为`函数值函数`（或`有函数值的函数`）。

有两个关于`funargs`和`function values`的概念性问题。并且这两个子问题被概括称为`Funarg问题`（或`函数参数问题`）的问题。为了精确地解决整个`funarg问题`，发明了闭包的概念。让我们更加详细地描述这两个子问题（我们将看到它们都是ECMAScript中在一个函数的图形属性`[[Scope]]`提到的）。

`funcarg问题`的第一个子类型是`向上的funarg问题`。当一个函数从另一个函数向上返回（回到外部）并且使用之前已经提到的自由变量。为了能够访问父环境甚至甚至父环境接收后的变量，内部函数在创建时在`[[Scope]]`属性中保存了父环境的作用域链。然后该内部函数激活时，它的环境的作用域链形成激活对象和此`[[Scope]]`属性的组合（实际上是我们刚才在图中看见的内容）：

```javascript
Scope chain = Activation object + [[Scope]] // 作用域链 = 激活对象 + [[Scope]]
```

再注意一下主要的事情 — 在创建时 — 函数保存父作用域，因为这个保存的作用域链将会用于在函数进一步调用中的变量查找。

```javascript
function foo() {
  var x = 10;
  return function bar() {
    console.log(x);
  };
}

// foo 返回一个函数，返回的函数使用了变量 x
var returnedFunction = foo();
 
// 全局变量 x
var x = 20;
 
// 返回函数的执行
returnedFunction(); // 10，而不是 20
```

这种作用域风格称为静态（词法）作用域。我们看到变量`x`在返回函数保存的`[[Socpe]]`中找到。一般理论中，当例子中的变量`x`被解析（查找）为`20`时，这也是动态作用域。只是，ECMAScript中并未使用动态作用域。

`funarg问题`的第二部分是`向下的funarg问题`，这种情况下父环境可以存在，但可能是解析标识符的歧义。问题是：从哪个作用域使用标识符的值 — 在函数创建时静态保存还是执行的时候动态形成（即调用者的作用域）？为了避免这种歧义，形成闭包，就决定使用静态作用域：

```javascript
// 全局 x
var x = 10;
// 全局函数
function foo() {
  console.log(x);
}
 
(function (funarg) {
  // 局部 x
  var x = 20;
  // 这是没有歧义的，因为我们用了在foo中静态保存到[[Scope]]的全局x
  // 而不是激活了函数参数的调用者（这个匿名立即执行函数）作用域中的 x
  funarg(); // 10, 而不是 20
})(foo); // foo 通过`向下`作为函数参数
```

我们可以得出结论，在语言中使用闭包，静态作用域是强制性要求。但是某些语言可能会提供动态和静态作用域的组合，允许程序员选择关闭什么打开什么。因为在ECMAScript中只是用静态作用域（即我们对`funarg问题`的两个子类型都有解决方案），结论是：ECMAScript完全支持闭包，技术上是使用函数`[[Scope]]`属性去实现的。现在我们可以给出一个对闭包的正确定义：

>闭包是代码块（在ECMAScript中是一个函数）和静态/词法保存父作用域的组合。因此，通过这些保存的作用域，函数可以轻松引用到自由变量。

注意。每个（普通）函数创建时`[[Scope]]`都会保存，理论上，在ECMAScript中全部函数都是闭包。

另一个需要注意的重要事项，一个函数可能拥有相同的父作用域（例如，我们有两个内部/全局函数，这是非常正常的情况）。这种情况下，保存在`[[Scope]]`属性中的变量是被拥有相同父作用域链的全部函数共享的。一个闭包对变量所做的改变反应在另一个闭包的读取：

```javascript
function baz() {
  var x = 1;
  return {
    foo: function foo() { return ++x; },
    bar: function bar() { return --x; }
  };
}
 
var closures = baz();
 
console.log(
  closures.foo(), // 2
  closures.bar()  // 1
);
```

这段代码可以用以下插图表示：

![图片描述](http://dmitrysoshnikov.com/wp-content/uploads/shared-scope.png)

<p align="center">图 11.共享的[[Scope]]</p>

正是这个特性与在循环中创建多个函数的混淆是相关的。在创建的函数内部使用循环计数器，当所有函数内拥有相同的计数器值，一些程序员经常会得到以外的结果。现在应该清楚为什么会这样 — 因为这些函数拥有相同的`[[Scope]]`，循环计数器拥有最后赋值的值。

```javascript
var data = [];
 
for (var k = 0; k < 3; k++) {
  data[k] = function () {
    console.log(k);
  };
}
 
data[0](); // 3, 而不是 0
data[1](); // 3, 而不是 1
data[2](); // 3, 而不是 2
```

有几种技术可以解决这个问题。其中一种是在作用域链中提供一个额外的对象 — 例如使用额外方法：

```javascript
var data = [];
 
for (var k = 0; k < 3; k++) {
  data[k] = (function (x) {
    return function () {
      console.log(x);
    };
  })(k); // 传递 k 值
}
 
// 现在是正确的了
data[0](); // 0
data[1](); // 1
data[2](); // 2
```

注意：ES6引入了块级作用域绑定。通过`let`或`const`关键字来完成。如以上例子可以简单便捷地重写：

```javascript
let data = [];
 
for (let k = 0; k < 3; k++) {
  data[k] = function () {
    console.log(k);
  };
}
 
data[0](); // 0
data[1](); // 1
data[2](); // 2
```

那些有兴趣对闭包理论和实际应用有更深入的人可以再[第六章 闭包](http://dmitrysoshnikov.com/ecmascript/chapter-6-closures/)找到更多信息。想获得关于作用域链的更多信息，请查看[第四章 作用域链](http://dmitrysoshnikov.com/ecmascript/chapter-4-scope-chain/)。

考虑到执行环境的最后一个属性，我们即将进入下一部分，关于`this`值的概念。

## This 值

> this值是一个关联执行环境的特殊对象，因此，它可以被命名为环境对象（即在所激活执行环境的环境对象）。

任何对象都能被用作环境的`this`值。一个重要的点是`this`值是执行环境的属性，而不是变量对象的属性。

这个特性非常重要，因为和变量相比，`this`值从不参与标识符的解析过程。即在代码中访问`this`是，它的值直接来自执行环境，不经过作用域链查找。`this`值在进入执行环境时只确定一次。

注意：在ES6中`this`实际上称为词法环境的一个属性，即ES3属于中变量对象的属性。这样做是为了支持从父环境中继承的、有词法上的`this`的箭头函数。

顺便一提，和ECMAScript相反，例如Python有它的函数`self`参数作为简单变量解析，可以在执行期间改变成另外的值。在ECMAScript中是不可能对`this`赋新的值的，重复一遍，`this`不是变量，也不存放在变量对象中。

在全局环境中，`this`值是全局对象自身（这意味着，`this`值和变量对象相等）：

```javascript
var x = 10;
 
console.log(
  x, // 10
  this.x, // 10
  window.x // 10
);
```

在函数环境的情况，`this`值在每个单独调用的函数中可能不同。`this`值由调用者通过调用表达式（例如，函数激活的方式）的形式来提供。举例子，如下的`foo`是一个被调用者，从全局环境（它是一个调用者）中被调用。看下面例子，如何对相同代码的函数，`this`值是如何被不同调用者在不同的调用（不同的函数激活方式）中提供的：

```javascript
// 函数foo的代码从未改变，但 this 值在每次激活都不同
function foo() {
  alert(this);
}
 
// 调用者激活 foo(被调用者) ，为被调用者提供 this 值
foo(); // 全局对象
foo.prototype.constructor(); // foo的原型
 
var bar = {
  baz: foo
};
 
bar.baz(); // bar
(bar.baz)(); // 也是 bar
(bar.baz = bar.baz)(); // 但这是全局对象
(bar.baz, bar.baz)(); // 也是全局对象
(false || bar.baz)(); // 也是全局对象
 
var otherFoo = bar.baz;
otherFoo(); // 又是全局对象
```

为了深入考虑`this`在每次函数调用时可能改变（可能更重要），你可以阅读[第三章 This](http://dmitrysoshnikov.com/ecmascript/chapter-3-this/)，这将详细讨论上述案例。

## 结论

到这一步，我们完成了这个简单的概述。虽然事实并非如此简短，对某些这题的正题解释需要一本完整的书。我们没有触及的两个主题：函数（以及函数类型的差异，例如函数声明和函数表达式）和ECMAScript中使用的评估策略。这两个主题可以在ES3系列的对应章节中找到：[第五章 函数](http://dmitrysoshnikov.com/ecmascript/chapter-5-functions/)和[第八章 评估策略](http://dmitrysoshnikov.com/ecmascript/chapter-8-evaluation-strategy/)。

如果您有意见、问题或补充，我很乐意在评论中看到你们。

祝你在学习ECMAScript学习中好运！

**撰稿：** Dmitry A. Soshnikov 
**发布于：** 2010-09-02


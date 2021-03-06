# 04 this、call、apply、bind

[验证](https://juejin.im/post/59bfe84351882531b730bac2)

### 1. this

先看例子。

```JS
var name = "windowsName"
function f1() {
  name: 'f1'
  console.log(this.name)
}
f1(); // windowsName

var obj2 = {
  f2: function () {
    console.log(this.name)
  }
}
obj2.f2(); // undefined


var obj3 = {
  name: 'obj3',
  f3: function() {
    console.log(this.name)
  }
}
obj3.f3(); // obj3
window.obj3.f3(); // obj3

var fn4 = obj3.f3;
fn4(); // windowsName

function f5() {
  var name = 'f5'
  innerFun();
  function innerFun() {
    console.log(this.name)
  }
}
f5(); // windowsName
```

输出结果有疑惑吗？ 逐条分析之前，回忆下作用域篇提到的 `this的取值是在执行的时候确定的，就是运行时所在的环境`， 简要说明就是『**this 取值永远指向最终调用它的对象**』。

1. f1() 在全局作用域下执行，最终调用它的对象是 window ，那么 this 指向 window 对象，this.name -> window.name 既 "windowsName"。
2. obj2.f2() ，由 obj2 调用 f2() 函数，那么 this 指向 obj2 对象，this.name -> obj2.name ，obj2 没有 name 属性 既 "undefined"。
3. 同上，obj3.f3() 执行中，this 指向 obj3，obj3 的 name 属性值为 obj3 既输出 "obj3"；window.obj3.f3() ，最终也是由 obj3 来调用 f3 函数，同样 this.name 得到 "obj3"。
4. fn4() ，不同的是 obj3 将方法 f3 赋给了 f4，然后再由 window 来调用 fn4 ，那么 this 指向 window 对象，既 "windowsName"。
5. f5() 依然由 window 调用，答案也是显而易见的。

似乎能找到一些规律，函数要执行就需要被调用，例子中函数最终能找到一个调用它的对象。

#### 剪头函数的 this

剪头函数没有 this，剪头函数的 this 继承于，定义时最近一层非剪头函数的 this。this 指向运行时的上下文环境。

看一些例子：

1.  绑定事件

```JS
const fn1 = function(){console.log(this, 'fn1')};
const fn2 = ()=>{console.log(this, 'fn2')};
const fn3 = function () {
  return function () {
    console.log(this, 'fn3')
  }
};
const fn4 = function () {
  return ()=> {
    console.log(this, 'fn4')
  }
};
const fn5 = () => {
  return function () {
    console.log(this, 'fn5')
  }
};

document.addEventListener('click', fn1) // document
document.addEventListener('click', fn2) // window
document.addEventListener('click', fn3()) // document
document.addEventListener('click', fn4()) // window
document.addEventListener('click', fn5()) // document
```

2.

```JS
var name = "windowsName";
var obj = {
    name : "Cherry",
    fn0: function () {
      console.log(this.name, 'fn0')
    },
    fn1: ()=> {
        console.log(this.name, 'fn1')
    },
    fn2: function () {
      setTimeout( () => {
        console.log(this.name, 'fn2')
      });
    },
    fn3: ()=> {
      setTimeout( () => {
        console.log(this.name, 'fn3')
      });
    },
    fn4: function () {
      const context = this;
      setTimeout( () => {
        console.log(context.name, 'fn4')
      });
    }
};

obj.fn0(); // 'Cherry'
obj.fn1(); // 'windowsName'
obj.fn2(); // 'Cherry'
obj.fn3(); // 'windowsName'
obj.fn4(); // 'Cherry'
```

#### 函数调用的几种方法

1. 作为普通函数
2. 作为对象方法
3. 显式调用（call、apply、bind 强行绑定）
4. 作为构造函数（new 操作符）

在下一步 this 之前，先搞清楚 new 和 call 在解析过程中发生了什么，才好理解为什么会改变 this。

### 2. new Object() 与 Object.create()

1. Object.create() 方法的定义：创建一个新对象，使用现有的对象来提供新创建的对象的**proto**。

#### 手写一个 Object.create()：

```JS
// 模拟 Object.create() 方法
function _create(Foo){
  let obj = {}
  obj.__proto__ = Foo
  return obj
}
```

**通常用 Object.create(null)来创建一个原型链干净的对象。**

2. new 运算符的定义：创建一个用户定义的对象类型的实例或具有构造函数的内置对象的实例。new 关键字会进行如下的操作：
   1. 创建一个空的简单 JavaScript 对象（即{}）；
   2. 链接该对象（即设置该对象的构造函数）到另一个对象 ；
   3. 将步骤 1 新创建的对象作为 this 的上下文 ；
   4. 如果该函数没有返回对象，则返回 this。

#### 手写一个 new 操作符：

```JS
// 模拟 new 运算符
function _new(fn, ...rests){
  // 1
  const obj = {}
  // 2
  obj.__proto__ = fn.prototype

  // 1、2两步可以用Object.create实现
  // const o = Object.create(fn.prototype)

  // 3
  const result = fn.apply(obj, rests)

  // 4
  return typeof result === 'object' ? result : obj
}
```

这里用了一个 apply 来修改了 this，那么来看看 apply 过程中发生了什么

### 3. call、apply、bind

先看定义，是学习认识新东西的第一步。Function.prototype.call()，call() 方法使用一个指定的 this 值和单独给出的一个或多个参数来调用一个函数。

#### 手写一个 call 函数：

```JS
// 模拟 call函数
var a ={
    fn : function (a,b) {
        console.log( a + b)
    }
}

var b = a.fn;
b(); // NaN
b.call(a,1,2) // 3

Function.prototype._call = function(context,...args) {
  // 1. context如果为null或者undefined或者没有传入，则指向window
  context = context || window
  // 2. 将调用call的函数 赋给 context[fn]，fn作为context方法用来访问context的所有属性
  // Symbol定义一个独一无二的值，避免和context中现有属性冲突
  const fn = Symbol()
  context[fn] = this
  // 3. 其余参数作为参数传递给context[fn]函数
  let foo = context[fn](...args)
  // 4. context[fn] 没有价值了手动删除
  delete context[fn]
  // 5. 返回
  return foo;
}

b._call(a,1,2) // 3
```

#### 手写一个 apply 函数：

```JS
// 与call一致，只是传参不同，apply使用数组
b.apply(a,[1,2]) // 3


// 模拟 apply函数，只是在call基础上改在下传参方式即可
Function.prototype._apply = function(context,args) {
  context = context || window
  const fn = Symbol()
  context[fn] = this
  let foo = context[fn](...args)
  delete context[fn]
  return foo;
}
```

#### 手写一个 bind 函数：

```JS
// bind 和call类似，不同的是函数返回一个函数
b.bind(a,1,2)() // 3

// 模拟 bind 函数
Function.prototype._bind = function(context, ...args){
  const _this = this
  return function(){
    // return _this.apply(context, args)
    return _this.call(context, ...args)
  }
}
```

### 4. 总结之前，补充剪头函数的 this

箭头函数和普通函数有些区别，箭头函数没有 this。使用时注意以下事项：

1. 没有 this，所以不能用 call()、apply()、bind()这些方法去改变 this 的指向。函数体内的 this 对象，继承的是外层代码块的 this。
1. 不可 new，也就是说不可以作为构造函数，否则会抛出一个错误。
1. 没有 arguments 对象，该对象在函数体内不存在。如果要用，可以用 rest 参数代替。
1. 没有 yield 命令，因此箭头函数不能用作 Generator 函数。

```JS
var obj = {
    hi: function(){
        console.log(this);
        return ()=>{
            console.log(this);
        }
    },
    sayHi: function(){
        return function() {
            console.log(this);
            return ()=>{
                console.log(this);
            }
        }
    },
    say: ()=>{
        console.log(this);
    }
}
let hi = obj.hi();  //输出obj对象
hi();               //输出obj对象
let sayHi = obj.sayHi();
let fun1 = sayHi(); //输出window
fun1();             //输出window
obj.say();          //输出window
```

### 5. 如何准确判断 this 的指向

接着 1 开始继续讲，如何判断 this 的指向。

#### 不同的调用形式对应的 this 指向

1. 作为普通函数，指向 window
1. 作为对象方法，指向对象
1. 显式调用（call、apply、bind 强行绑定），指向传入的对象
1. 作为构造函数（new 操作符），指向实例

**this 指定有优先级：new > 显式调用 > 方法调用 > 普通调用**

总结：

1. 函数是否在 new 中调用，如果是，this 指向新创建的对象；
2. 函数是否通过 call、apply、bind 调用，如果是，this 指向指定的对象；
3. 函数是否作为对象方法调用，形式如 `obj.foo()` （注意紧跟执行圆括号），this 指向对象 obj；
4. 如果都不是，this 指向全局对象；
5. 把 null 或者 undefined 作为 this 的绑定对象传入 call、apply 或者 bind，this 指向全局对象；
6. 匿名函数 this 指向全局

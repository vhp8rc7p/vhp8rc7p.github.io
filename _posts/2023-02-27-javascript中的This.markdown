---
layout: post
title:  "javascript中的This"
date:   2023-02-27 09:38:58 +0800
tags: JavaScript 学习笔记

---

1468


先来看一下ECMAScript标准对于this的定义

>The [AO] ResolveThisBinding […] determines the binding of the keyword this using the LexicalEnvironment of the running execution context. [Steps]:
>1. Let envRec be GetThisEnvironment().
>2. Return ? envRec.GetThisBinding().

我们暂称AO(Abstarct Operation)为抽象操作

Global Environment Records（全局环境记录）, module Environment Records, and function Environment Records（函数环境记录）都有他们自己的GetThisBinding()方法。

GetThisEnvironment抽象操作找到当前[running execution context](https://tc39.es/ecma262/#running-execution-context)的LexicalEnvironment（词法环境），并找到最近的具有 this 绑定（即HasThisBinding返回值为真）的ascendant Environment Record（通过迭代访问它们的 [[OuterEnv]] 属性）。 直到环境记录为上面的三种为止。

The value of this often depends on whether code is in strict mode.

GetThisBinding的返回值反映了current execution context的this的值, 每当一个新的execution context建立的时候, this resolves to a distinct value. This can also happen when the current execution context is modified. The following subsections list the five cases where this can happen.

You can put the code samples in the AST explorer to follow along with specification details.

### 1. Global execution context in scripts
   
This is script code evaluated at the top level, e.g. directly inside a <script>:

```
<script>
// Global context
console.log(this); // Logs global object.

setTimeout(function(){
  console.log("Not global context");
});
</script>

```
When in the initial global execution context of a script, GetThisBinding通过以下的步骤计算出this的值:

>The GetThisBinding concrete method of a global Environment Record envRec […] [does this]:

>Return envRec.[[GlobalThisValue]].

global Environment Record（全局环境记录）的[[GlobalThisValue]] 属性总是被设置为 host-defined global object, which is reachable via globalThis (window on Web, global on Node.js; Docs on MDN). Follow the steps of InitializeHostDefinedRealm to learn how the [[GlobalThisValue]] property comes to be.

### 2. Global execution context in [modules](https://developer.mozilla.org/en/docs/Web/JavaScript/Guide/Modules)
Modules（模块）在ECMAScript 2015被引入.

This applies to modules, e.g. when directly inside a <script type="module">, as opposed to a simple <script>.

When in the initial global execution context of a module, GetThisBinding通过以下的步骤计算出this的值:

>The GetThisBinding concrete method of a module Environment Record […] [does this]:

>Return undefined.
In modules, the value of this is always undefined in the global context. Modules are implicitly in strict mode.

### 3. Entering [eval](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/eval) code
一共有两种不同的eval调用: direct（直接）和 indirect（间接）. This distinction exists since the ECMAScript 5th edition.

- 直接的eval调用看起来像 eval(…); or (eval)(…); (or ((eval))(…);, etc.).1 It’s only direct if the call expression fits a narrow pattern.2
- 间接的调用 involves calling the function reference eval in any other way. It could be eval?.(…), (…, eval)(…), window.eval(…), eval.call(…,…), etc. Given const aliasEval1 = eval; window.aliasEval2 = eval;, it would also be aliasEval1(…), aliasEval2(…). Separately, given const originalEval = eval; window.eval = (x) => originalEval(x);, calling eval(…) would also be indirect.
  
间接调用具体可参见[chuckj’s answer to “(1, eval)('this') vs eval('this') in JavaScript?”](https://stackoverflow.com/a/9107491/196844) and [Dmitry Soshnikov’s ECMA-262-5 in detail – Chapter 2: Strict Mode (archived)](https://dmitrysoshnikov.com/ecmascript/es5-chapter-2-strict-mode#indirect-eval-call)

PerformEval executes the eval code. It creates a new declarative Environment Record as its LexicalEnvironment, which is where GetThisEnvironment gets the this value from.

Then, if this appears in eval code, the GetThisBinding method of the Environment Record found by GetThisEnvironment is called and its value returned.

And the created declarative Environment Record depends on whether the eval call was direct or indirect:

In a direct eval, it will be based on the current running execution context’s LexicalEnvironment.
In an indirect eval, it will be based on the [[GlobalEnv]] property (a global Environment Record) of the Realm Record which executed the indirect eval.
Which means:

In a direct eval, the this value doesn’t change; it’s taken from the lexical scope that called eval.
In an indirect eval, the this value is the global object (globalThis).
What about new Function? — new Function is similar to eval, but it doesn’t call the code immediately; it creates a function. A this binding doesn’t apply anywhere here, except when the function is called, which works normally, as explained in the next subsection.

### 4. Entering function code
Entering function code occurs when calling a function.

共有四种调用函数的语法。

- The EvaluateCall AO is performed for these three:3
    - Normal function calls
    - Optional chaining calls
    - Tagged templates
- And EvaluateNew is performed for this one:3
    - Constructor invocations
  
The actual function call happens at the Call AO, which is called with a thisValue determined from context; this argument is passed along in a long chain of call-related calls. Call calls the [[Call]] internal slot of the function. This calls PrepareForOrdinaryCall where a new function Environment Record is created:

>A function Environment Record is a declarative Environment Record that is used to represent the top-level scope of a function and, if the function is not an ArrowFunction, provides a this binding. If a function is not an ArrowFunction function and references super, its function Environment Record also contains the >state that is used to perform super method invocations from within the function.

In addition, there is the [[ThisValue]] field in a function Environment Record:

>This is the this value used for this invocation of the function.

The NewFunctionEnvironment call also sets the function environment’s [[ThisBindingStatus]] property.

[[Call]] also calls OrdinaryCallBindThis, where the appropriate thisArgument is determined based on:

the original reference,
the kind of the function, and
whether or not the code is in strict mode.
Once determined, a final call to the BindThisValue method of the newly created function Environment Record actually sets the [[ThisValue]] field to the thisArgument.

Finally, this very field is where a function Environment Record’s GetThisBinding AO gets the value for this from:

>The GetThisBinding concrete method of a function Environment Record envRec […] [does this]:[…]
>1. Return envRec.[[ThisValue]].

Again, how exactly the this value is determined depends on many factors; this was just a general overview. With this technical background, let’s examine all the concrete examples.

**Arrow functions**
When an arrow function is evaluated, the [[ThisMode]] internal slot of the function object is set to “lexical” in OrdinaryFunctionCreate.

At OrdinaryCallBindThis, which takes a function F:

>Let thisMode be F.[[ThisMode]].

>If thisMode is lexical, return NormalCompletion(undefined). […]
which just means that the rest of the algorithm which binds this is skipped. An arrow function does not bind its own this value.

So, what is this inside an arrow function, then? Looking back at ResolveThisBinding and GetThisEnvironment, the HasThisBinding method explicitly returns false.

The HasThisBinding concrete method of a function Environment Record envRec […] [does this]:

If envRec.[[ThisBindingStatus]] is lexical, return false; otherwise, return true.
So the outer environment is looked up instead, iteratively. The process will end in one of the three environments that have a this binding.

This just means that, in arrow function bodies, this comes from the lexical scope of the arrow function, or in other words (from Arrow function vs function declaration / expressions: Are they equivalent / exchangeable?):

>Arrow functions don’t have their own this […] binding. Instead, [this identifier is] resolved in the lexical scope like any other variable. That means that inside an arrow function, this [refers] to the [value of this] in the environment the arrow function is defined in (i.e. “outside” the arrow function).

**Function properties**
In normal functions (function, methods), this is determined by how the function is called.

This is where these “syntax variants” come in handy.

Consider this object containing a function:

const refObj = {
    func: function(){
      console.log(this);
    }
  };
Alternatively:

const refObj = {
    func(){
      console.log(this);
    }
  };
In any of the following function calls, the this value inside func will be refObj.1

refObj.func()
refObj["func"]()
refObj?.func()
refObj.func?.()
refObj.func``
If the called function is syntactically a property of a base object, then this base will be the “reference” of the call, which, in usual cases, will be the value of this. This is explained by the evaluation steps linked above; for example, in refObj.func() (or refObj["func"]()), the CallMemberExpression is the entire expression refObj.func(), which consists of the MemberExpression refObj.func and the Arguments ().

But also, refObj.func and refObj play three roles, each:

they’re both expressions,
they’re both references, and
they’re both values.
refObj.func as a value is the callable function object; the corresponding reference is used to determine the this binding.

The optional chaining and tagged template examples work very similarly: basically, the reference is everything before the ?.(), before the ``, or before the ().

EvaluateCall uses IsPropertyReference of that reference to determine if it is a property of an object, syntactically. It’s trying to get the [[Base]] property of the reference (which is e.g. refObj, when applied to refObj.func; or foo.bar when applied to foo.bar.baz). If it is written as a property, then GetThisValue will get this [[Base]] property and use it as the this value.

Note: Getters / Setters work the same way as methods, regarding this. Simple properties don’t affect the execution context, e.g. here, this is in global scope:

const o = {
    a: 1,
    b: this.a, // Is `globalThis.a`.
    [this.a]: 2 // Refers to `globalThis.a`.
  };
Calls without base reference, strict mode, and with
A call without a base reference is usually a function that isn’t called as a property. For example:

func(); // As opposed to `refObj.func();`.
This also happens when passing or assigning methods, or using the comma operator. This is where the difference between Reference Record and Value is relevant.

Note function j: following the specification, you will notice that j can only return the function object (Value) itself, but not a Reference Record. Therefore the base reference refObj is lost.

const g = (f) => f(); // No base ref.
const h = refObj.func;
const j = () => refObj.func;

g(refObj.func);
h(); // No base ref.
j()(); // No base ref.
(0, refObj.func)(); // Another common pattern to remove the base ref.
EvaluateCall calls Call with a thisValue of undefined here. This makes a difference in OrdinaryCallBindThis (F: the function object; thisArgument: the thisValue passed to Call):

Let thisMode be F.[[ThisMode]].
[…]

If thisMode is strict, let thisValue be thisArgument.
Else,
If thisArgument is undefined or null, then
Let globalEnv be calleeRealm.[[GlobalEnv]].
[…]
Let thisValue be globalEnv.[[GlobalThisValue]].
Else,
Let thisValue be ! ToObject(thisArgument).
NOTE: ToObject produces wrapper objects […].
[…]

Note: step 5 sets the actual value of this to the supplied thisArgument in strict mode — undefined in this case. In “sloppy mode”, an undefined or null thisArgument results in this being the global this value.

If IsPropertyReference returns false, then EvaluateCall takes these steps:

Let refEnv be ref.[[Base]].
Assert: refEnv is an Environment Record.
Let thisValue be refEnv.WithBaseObject().
This is where an undefined thisValue may come from: refEnv.WithBaseObject() is always undefined, except in with statements. In this case, thisValue will be the binding object.

There’s also Symbol.unscopables (Docs on MDN) to control the with binding behavior.

To summarize, so far:

function f1(){
  console.log(this);
}

function f2(){
  console.log(this);
}

function f3(){
  console.log(this);
}

const o = {
    f1,
    f2,
    [Symbol.unscopables]: {
      f2: true
    }
  };

f1(); // Logs `globalThis`.

with(o){
  f1(); // Logs `o`.
  f2(); // `f2` is unscopable, so this logs `globalThis`.
  f3(); // `f3` is not on `o`, so this logs `globalThis`.
}
and:

"use strict";

function f(){
  console.log(this);
}

f(); // Logs `undefined`.

// `with` statements are not allowed in strict-mode code.
Note that when evaluating this, it doesn’t matter where a normal function is defined.

.call, .apply, .bind, thisArg, and primitives
Another consequence of step 5 of OrdinaryCallBindThis, in conjunction with step 6.2 (6.b in the spec), is that a primitive this value is coerced to an object only in “sloppy” mode.

To examine this, let’s introduce another source for the this value: the three methods that override the this binding:4

Function.prototype.apply(thisArg, argArray)
Function.prototype. {call, bind} (thisArg, ...args)
.bind creates a bound function, whose this binding is set to thisArg and cannot change again. .call and .apply call the function immediately, with the this binding set to thisArg.

.call and .apply map directly to Call, using the specified thisArg. .bind creates a bound function with BoundFunctionCreate. These have their own [[Call]] method which looks up the function object’s [[BoundThis]] internal slot.

Examples of setting a custom this value:

function f(){
  console.log(this);
}

const myObj = {},
  g = f.bind(myObj),
  h = (m) => m();

// All of these log `myObj`.
g();
f.bind(myObj)();
f.call(myObj);
h(g);
For objects, this is the same in strict and non-strict mode.

Now, try to supply a primitive value:

function f(){
  console.log(this);
}

const myString = "s",
  g = f.bind(myString);

g();              // Logs `String { "s" }`.
f.call(myString); // Logs `String { "s" }`.
In non-strict mode, primitives are coerced to their object-wrapped form. It’s the same kind of object you get when calling Object("s") or new String("s"). In strict mode, you can use primitives:

"use strict";

function f(){
  console.log(this);
}

const myString = "s",
  g = f.bind(myString);

g();              // Logs `"s"`.
f.call(myString); // Logs `"s"`.
Libraries make use of these methods, e.g. jQuery sets the this to the DOM element selected here:

$("button").click(function(){
  console.log(this); // Logs the clicked button.
});
Constructors, classes, and new
When calling a function as a constructor using the new operator, EvaluateNew calls Construct, which calls the [[Construct]] method. If the function is a base constructor (i.e. not a class extends…{…}), it sets thisArgument to a new object created from the constructor’s prototype. Properties set on this in the constructor will end up on the resulting instance object. this is implicitly returned, unless you explicitly return your own non-primitive value.

A class is a new way of creating constructor functions, introduced in ECMAScript 2015.

function Old(a){
  this.p = a;
}

const o = new Old(1);

console.log(o);  // Logs `Old { p: 1 }`.

class New{
  constructor(a){
    this.p = a;
  }
}

const n = new New(1);

console.log(n); // Logs `New { p: 1 }`.
Class definitions are implicitly in strict mode:

class A{
  m1(){
    return this;
  }
  m2(){
    const m1 = this.m1;
    
    console.log(m1());
  }
}

new A().m2(); // Logs `undefined`.
super
The exception to the behavior with new is class extends…{…}, as mentioned above. Derived classes do not immediately set their this value upon invocation; they only do so once the base class is reached through a series of super calls (happens implicitly without an own constructor). Using this before calling super is not allowed.

Calling super calls the super constructor with the this value of the lexical scope (the function Environment Record) of the call. GetThisValue has a special rule for super calls. It uses BindThisValue to set this to that Environment Record.

class DerivedNew extends New{
  constructor(a, a2){
    // Using `this` before `super` results in a ReferenceError.
    super(a);
    this.p2 = a2;
  }
}

const n2 = new DerivedNew(1, 2);

console.log(n2); // Logs `DerivedNew { p: 1, p2: 2 }`.
5. Evaluating class fields
Instance fields and static fields were introduced in ECMAScript 2022.

When a class is evaluated, ClassDefinitionEvaluation is performed, modifying the running execution context. For each ClassElement:

if a field is static, then this refers to the class itself,
if a field is not static, then this refers to the instance.
Private fields (e.g. #x) and methods are added to a PrivateEnvironment.

Static blocks are currently a TC39 stage 3 proposal. Static blocks work the same as static fields and methods: this inside them refers to the class itself.

Note that in methods and getters / setters, this works just like in normal function properties.

class Demo{
  a = this;
  b(){
    return this;
  }
  static c = this;
  static d(){
    return this;
  }
  // Getters, setters, private modifiers are also possible.
}

const demo = new Demo;

console.log(demo.a, demo.b()); // Both log `demo`.
console.log(Demo.c, Demo.d()); // Both log `Demo`.
1: (o.f)() is equivalent to o.f(); (f)() is equivalent to f(). This is explained in this 2ality article (archived). Particularly see how a ParenthesizedExpression is evaluated.

2: It must be a MemberExpression, must not be a property, must have a [[ReferencedName]] of exactly "eval", and must be the %eval% intrinsic object.

3: Whenever the specification says “Let ref be the result of evaluating X.”, then X is some expression that you need to find the evaluation steps for. For example, evaluating a MemberExpression or CallExpression is the result of one of these algorithms. Some of them result in a Reference Record.

4: There are also several other native and host methods that allow providing a this value, notably Array.prototype.map, Array.prototype.forEach, etc. that accept a thisArg as their second argument. Anyone can make their own methods to alter this like (func, thisArg) => func.bind(thisArg), (func, thisArg) => func.call(thisArg), etc. As always, MDN offers great documentation.


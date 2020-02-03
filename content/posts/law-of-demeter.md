---
title: "迪米特法则(Law of Demeter)"
date: 2020-02-02T16:44:02+08:00
draft: true
---

迪米特法则又称最少知识原则，
主要是软件开发特别是面向对象开发中的一条设计准则。
他是低耦合的一种特例。
这条规则可以归纳为以下三点：

1. 每个某块应当尽减少和其他某块的关联程度(Each unit should have only limited knowledge about other units: only **units "closely" related** to the current unit.)
2. 每个模块应当只和他的“朋友”发生关联(Each unit should only talk to its **friends**; don't talk to strangers.)
3. 每个模块只和他的直接朋友关联(Only talk to your **immediate friends**.)

> The Law of Demeter (LoD) or principle of least knowledge is a design guideline for developing software, particularly object-oriented programs. In its general form, the LoD is a specific case of loose coupling. The guideline was proposed by Ian Holland at Northeastern University towards the end of 1987, and can be succinctly summarized in each of the following ways:[1]

```A``` 对象可以调用 ```B``` 对象的服务，
但是 ```A``` 对象不能通过 ```B``` 对象 访问 ```C``` 对象来调用 ```C``` 的服务。
这样做意味着 ```A``` 对象需要了解更多 ```B``` 对象的内部结构。

> An object ```A``` can request a service (call a method) of an object instance ```B```, but object ```A``` should not "reach through" object B to access yet another object, ```C```, to request its services. Doing so would mean that object ```A``` implicitly requires greater knowledge of object ```B's``` internal structure.

正确的做法是，修改 ```B``` 对象来满足 ```A``` 对象的请求：通过 ```B``` 对象提供的服务调用 ```C``` 对象。
或者 A 对象直接调用 C 对象的方法。

> Instead, ```B's``` interface should be modified if necessary so it can directly serve object ```A's``` request, propagating it to any relevant subcomponents. Alternatively, ```A``` might have a direct reference to object ```C``` and make the request directly to that. If the law is followed, only object B knows its own internal structure.

更准确的讲，迪米特法则要求对象 ```O``` 的方法 ```m``` 只调用以下对象的方法：

1. ```O``` 对象自己的方法
2. ```m``` 方法的参数
3. ```m``` 方法创建、初始化的对象
4. ```O``` 对象的直接组件的对象
5. ```O``` 对象访问的全局变量，范围局限在方法 ```m```

> More formally, the Law of Demeter for functions requires that a method m of an object O may only invoke the methods of the following kinds of objects:[2]

1. O itself
2. m's parameters
3. Any objects created/instantiated within m
4. O's **direct component** objects
5. A global variable, accessible by O, in the scope of m

特别的，一个对象应该避免调用**其他方法返回的成员对象**的方法。
简单的讲就是应该避免类似 ```a.b.Method()``` 这样的调用。
一个比较贴切的类比是：你命令一条狗走路，而不是命令狗的腿去走路。

> In particular, an object should avoid invoking methods of a member object returned by another method. For many modern object oriented languages that use a dot as field identifier, the law can be stated simply as "use only one dot". That is, the code a.b.Method() breaks the law where a.Method() does not. As an analogy, when one wants a dog to walk, one does not command the dog's legs to walk directly; instead one commands the dog which then commands its own legs.



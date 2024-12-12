---
title: Lua实现面向对象
date: 2022-05-02 11:22:42
tags: ['Lua', '编程语言', '面向对象', '语言基础']
categories: 语言
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA1f5eddee1c38fb6231591d40df4d29f2.png
---

最近在实习，接到的第一个任务是用公司的自研引擎制作一款抖音特效《宅家舞蹈秀》并于五一期间上线抖音。目前特效已经上线了，不过数据不是很好看，希望之后能被推起来吧orz。

说到自研引擎，给我的感觉和 Unity 一类的游戏引擎很像，只不过有些功能开发的不是很成熟（或者说健全），会有一些偶发的诡异的bug（例如粒子播放失败）。虽然存在些许问题，但这款引擎的功能足够满足抖音小游戏/抖音特效这样轻量级的开发了。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA129cef078d6f75eefe49a7c1e0152179.png" width=200 />

自研引擎所使用的编程语言是**Lua**。Lua是一个灵活且轻量的脚本语言，在实际的游戏开发工作流中，可以用它结合**xLua框架**来实现游戏的**热更新**，也可以直接用Lua开发游戏（特别是一些早期的小游戏）。Lua本身并没有提供面向对象的特性，当游戏系统变得复杂时我们该怎么办呢？

在现代游戏编程中，我们会采用具有面向对象特性的语言（例如Java、C#、C++）来开发游戏。面向对象有三个特性，分别是：

1. **封装**：就是把对象的属性和行为（数据）结合为一个独立的整体，并尽可能隐藏对象的内部实现细节
2. **继承**：从已有的类中派生出新的类称为子类，子类继承父类的数据属性和行为，并能根据自己的需求扩展出新的行为，可以提高代码的复用性
3. **多态**：指允许不同的对象对同一消息做出响应。即同一消息可以根据发送对象的不同而采用多种不同的行为方式（发送消息就是函数调用）

答案是：我们可以借助Lua的数据结构模拟面向对象编程所具有的特性，这也是本文所讨论的主题。

# 利用table+元表实现OOP

## 封装
Lua中有一个名为 `table` 的强大的数据结构，通过它我们可以创建不同的数据类型，如：数组、字典等。Lua 中的类可以通过 `table` + `function` 模拟出来，我们可以使用表来创建对象。

```Lua
-- 声明一个table
t = {}
​-- 声明属性，其中.之后的是键， = 之后的是值。
t.name = "GT"
-- function是一个函数
t.sayHello = function ()
    print("hello")
end
​
print(t.name)   -- GT
t.sayHello()    -- hello
```

## 继承
至于继承，则可以通过Lua的**元表**（`metetable`）模拟出来。当我们访问 `table` 中一个不存在的字段时，则会查找它的元表中的 `__index` **元方法**，元方法中如果存在这个字段就返回这个字段的值，没有则返回**nil**。

假设我们声明了一个动物的类：

```Lua
animal = {}
animal.name = "animal"
animal.age = 1
animal.sayHello = function ()
    print("I'am animal")
end
```

添加一个new方法，用来修改传入的table的元表。

```Lua
animal.new = function (self, o)
    o = o or {}             -- 如果o是nil，创建一个table
    self.__index = self     -- 将元表的 __index 指向自身(self)
    setmetatable(o,self)    -- 将自身设置为o的元表
    return o
end
```

这时我们就可以通过这个新定义的 `new` 方法来创建“**子类**”了。

我们分别调用 `new` 方法来创建一个 `dog` 和 `cat` 的实例，该方法会将 `dog` 和 `cat` 的元表设置为 `animal` 类，当查找它们中不存在的字段时，将访问它们的元表(也就是 `animal` )中进行查找。

```Lua
dog = animal:new()
cat = animal:new()
print(dog.name)     -- animal
print(cat.age)      -- 1
```

那么我们是否有办法实现多重继承呢？答案是有的。在Lua中，如果它的元表的 `__index` 字段被设置成一个函数时，当 Lua 在原来的表中找不到某个键时就会以**表**和没找到的**字段**为参数调用这个函数。如果 `__index` 字段是一个表，则会直接访问这个表。所以实现多重继承的思路很简单，我们只需要在创建类的时候传入它的多个父类，并且将 `__index` 字段属性设置为一个函数，这个函数会遍历所有父类，查找父类中的属性。

```Lua
-- 遍历plist列表，查找其中的k
local function search (k, plist)
    for i = 1, #plist do
        local v = plist[i][k] -- 尝试在第i个父类中查找k
        if v then return v end
    end
end
​
function createClass (...)
    local c = {} -- 新创建的类
    local parents = {...} -- 所有的父类
​
    -- 定义元表，从父类中查找不存在的方法
    setmetatable(c, {__index = function (t, k)
                return search(k, parents)
            end})
​
    -- 将c作为其实例的元表
    c.__index = c
​
    -- 为新类定义一个构造函数
    function c:new (o)
        o = o or {}
        setmetatable(o, c)
        return o
    end
​
    return c -- return new class
end
```

## 多态
多态指子类能够对父类的函数进行重写（例如 `animal` 类中的 `sayHello` 方法），这在Lua中也是可以实现的。

```Lua
dog = animal:new()
dog.name = "dog"
dog.sayHello = function ()  -- 重新定义sayHello
    print("wangwang")
end
dog.sayHello()  -- wangwang
​
cat = animal:new()
cat.name = "cat"
cat.sayHello()  -- 仍是animal中的sayhello，即输出I'am animal
```

# 参考资料
https://www.runoob.com/lua/lua-object-oriented.html
https://zhuanlan.zhihu.com/p/473823334?utm_psn=1850512079083147264
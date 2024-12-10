---
tags: ['游戏编程','经验分享','简单工厂模式','抽象工厂模式','工厂方法模式','工厂模式','设计模式','Unity']
categories: 设计模式
title: 游戏中的工厂模式
date: '2022-01-03T15:37:17.071Z'
updated: '2022-01-03T17:52:21.258Z'
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAbd0d24caacbe13c505f82573f273cebd.png
---

# 什么是工厂模式？
> 工厂模式（Factory pattern）是一种实现了“**工厂**”概念的面向对象设计模式，它处理在不指定对象具体类别的情况下创建对象的问题。工厂方法模式的实质是“**定义一个创建对象的接口，但让实现这个接口的类别来决定实例化哪个类别。工厂方法让类别的实例化推迟到子类中进行。**”

由此可知，工厂模式定义一个创建对象的接口，但**让子类决定实例化哪个类**。工厂方法允许类将实例化延迟到子类，这节省了实例化的开销。在游戏开发中，工厂模式可以被用于创建某一类对象的诸多不同类型的“变体”，例如敌人这个大类分为远程敌人、近战敌人、辅助敌人等小类。

实例化一个具体的敌人是一个复杂的过程（需要指定类别、设置不同的不同初始数值），但又具有相似性，那么为什么不把这一过程抽象出来、交给一个专门的管理器呢？这就是工厂类的思想，创建过程都由工厂统一管理，当发生创建业务逻辑变化时，只需要修改工厂即可。

根据产品是 <u>具体产品</u> 还是 <u>具体工厂</u>，工厂模式可分为**简单工厂模式**和**工厂方法模式**；根据工厂的 <u>抽象程度</u> 可分为**工厂方法模式**和**抽象工厂模式**。但不论是哪种工厂模式，其中都有**工厂**和**产品**两个主要角色。

# 简单工厂模式(Simple Factory Pattern)
简单工厂模式又称为**静态工厂方法**(Static Factory Method)模式，包含如下角色：
- **工厂**
    工厂负责实现创建所有实例的内部逻辑，根据条件（例如字符串）判断返回哪个具体产品
- **抽象产品（接口）**
    抽象产品是所创建的所有具体产品对象的父类，负责描述所有具体产品实例所共有的公共接口
- **具体产品**
    具体产品是**创建目标**，所有创建的对象都充当这个角色的某个具体类的实例。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA70ada692dc4dc6f83e9d4ff1c6c91fd3.png" />

简单工厂模式的要点在于：只需要传入一个正确的参数就可以获取所需要的对象，无须知道其创建细节。由于工厂方法是静态方法，可通过类名直接调用，只需要传入一个简单的参数即可。在实际开发中，还可以在调用时将所传入的参数保存在XML等格式的**配置文件**中，修改参数时无须修改任何源代码。

> 开闭原则：对于扩展是**开放**的，但是对于修改是**封闭**的。

简单工厂模式最大的问题在于**工厂类的职责相对过重**，增加新的产品需要修改工厂类的判断逻辑，这一点与**开闭原则**是相违背的（不利于系统的扩展和维护）。而且一旦工厂类不能正常工作，整个系统都要受到影响。

适用于创建的对象较少的场景（避免业务逻辑过于复杂），如JDK中的 `java.text.DateFormat`。

## 代码实现
```C#
// 工厂
public class Factory{
    public Factory();
    public static Product CreateProduct(string name){
        // 判断逻辑（必须）
        if(name.Equals("A")){
            return new ProductA();
        }else if(name.Equals("B")){
            return new ProductB();
        }
    }
}
// 抽象产品
public abstract class Product{
    // 某些产品通用方法
    public abstract void Describe();
}
// 具体产品
public class ProductA : Product{
    public override void Describe(){
        console.WriteLine("A");
    }
}
public class ProductB : Product{
    public override void Describe(){
        console.WriteLine("B");
    }
}
```

# 工厂方法模式(Factory Method Pattern)
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAc1a82a2c9034362cdcd18fdd95eaf057.png" />

工厂方法模式又称为工厂模式（<u>狭义的</u> 工厂模式），也叫**多态工厂**(Polymorphic Factory)模式（因为使用到了多态性）。在工厂方法模式中，**工厂父类**负责定义创建产品对象的公共接口，而**工厂子类**则负责生成具体的产品对象，这样做的目的是**将产品类的实例化操作延迟到工厂子类中完成**，即通过工厂子类来确定究竟应该实例化哪一个具体产品类，减轻了简单工厂模式中工厂类的负担，将具体创建工作交给子类去做。工厂方法模式包含如下角色：

- **抽象产品**
    抽象产品是所创建的所有具体产品对象的父类，负责描述所有具体产品实例所共有的公共接口
- **具体产品**
    具体产品是**创建目标**，所有创建的对象都充当这个角色的某个具体类的实例。
- **抽象工厂**
    返回产品对象的工厂方法。 该方法的返回对象类型必须与产品接口相匹配
- **具体工厂**
    重写基础工厂方法， 使其返回不同类型的产品。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAd1392cf620f649ed685999c34a1f9a10.png" />

工厂方法将创建产品的代码与实际使用产品的代码分离，从而能在不影响其他代码的情况下扩展产品创建部分代码。例如需要向应用中添加一种新产品，只需要开发新的具体工厂类，然后重写其工厂方法即可。工厂方法模式可以在不改变现有架构的情况下随意添加新的工厂和产品，便于扩展，符合开闭原则。

不过工厂方法模式需要引入许多新的子类（比如说一百种产品和一百种工厂），代码可能会因此变得更复杂。采用**抽象工厂模式**、**原型模式**或**生成器模式**或许会是一个更好的选择。

工厂方法模式适用于一个类不知道它所需要的对象的类；一个类通过其子类来指定创建哪个对象；将创建对象的任务委托给多个工厂子类中的某一个，客户端在使用时可以无须关心是哪一个工厂子类创建产品子类，需要时再动态指定。

## 代码实现

```C#
// 抽象工厂
public abstract class Factory{
    public abstract Product CreateProduct();
}
// 具体工厂
public class FactoryA : Factory{
    public override Product CreateProduct(){
        return new ProductA();
    }
}
public class FactoryB : Factory{
    public override Product CreateProduct(){
        return new ProductB();
    }
}

// 抽象产品
public abstract class Product{
    // 某些产品通用方法
    public abstract void Describe();
}
// 具体产品
public class ProductA : Product{
    public override void Describe(){
        console.WriteLine("A");
    }
}
public class ProductB : Product{
    public override void Describe(){
        console.WriteLine("B");
    }
}
```

# 抽象工厂模式(Abstract Factory Method)
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA4299c9fd6949f460b3da817b55f87022.png" />

前面的工厂方法模式的使用场景可能是这样的：通过工厂创建好几种不同类型的敌人。那如果我们想创建好几种不同类型的玩家呢？再重新写一套玩家工厂方法吗？虽然这么做符合开闭原则，但却让系统变得更复杂了，能不能让创建这个操作更加精简呢？

当产品愈发复杂时，我们可以考虑用**抽象工厂模式**（又称为**Kit模式**）替代工厂方法模式。抽象工厂模式相当于在工厂模式的基础上再加了一层抽象，属于“**工厂的工厂**”。为了更清晰地理解抽象工厂模式，需要先引入两个概念：

- **产品等级结构**
产品等级结构即产品的**继承结构**，如一个抽象类是敌人，其子类有远程敌人、近战敌人、辅助敌人等等，则抽象敌人与具体敌人之间构成了一个产品等级结构，抽象敌人是父类，而具体敌人是其子类。
- **产品族**
在抽象工厂模式中，产品族是指由同一个工厂生产的，位于不同产品等级结构中的一组产品，如远程工厂生产的远程敌人、远程玩家，远程敌人位于敌人这一产品等级结构中，远程玩家位于玩家这一产品等级结构中。

当系统所提供的工厂所需生产的具体产品并不是一个简单的对象，而是多个位于不同产品等级结构中属于不同类型的具体产品时，需要使用抽象工厂模式。抽象工厂模式与工厂方法模式最大的区别在于：

> **工厂方法模式针对的是一个产品等级结构，而抽象工厂模式则需要面对多个产品等级结构。一个工厂等级结构可以负责多个不同产品等级结构中的产品对象的创建。**

当一个工厂等级结构可以创建出分属于不同产品等级结构的一个产品族中的所有对象时，抽象工厂模式比工厂方法模式更为简单、有效率。抽象工厂模式同样包含如下角色：

- 抽象工厂
- 具体工厂
- 抽象产品
- 具体产品

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA89cc347d979455f85857bccc3b3299a4.png" />

抽象工厂模式同样隔离了具体类的生成，使得客户并不需要知道什么被创建。所有的具体工厂都实现了抽象工厂中定义的那些公共接口，因此只需改变具体工厂的实例，就可以在某种程度上改变整个软件系统的行为。另外，应用抽象工厂模式可以实现**高内聚低耦合**的设计目的，因此抽象工厂模式得到了广泛的应用。

当一个产品族中的多个对象被设计成一起工作时，它能够保证客户端始终只使用**同一个产品族**中的对象（例如：友好实体，其中包括了玩家也包括一些生物）。这对一些需要根据当前环境来决定其行为的软件系统来说，是一种非常实用的设计模式。

增加新的具体工厂和产品族很方便，无须修改已有系统，同样符合**开闭原则**。

不过，当只有一个产品族时，**抽象工厂模式会退化成工厂方法模式**。

# 简单工厂模式、工厂模式和抽象工厂模式的区别

1. 简单工厂根据动物工厂，可以直接得到对应的动物

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA2119f4f68caa6fbeca73cfd85016db32.png" />

2. 工厂模式在工厂实例化之后，需要调用具体的动物工厂的制造方法得到具体的动物

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAa57e8a9221375b195b978a0edd976908.png" />

3. 抽象工厂模式更多一层抽象，一个工厂对应多种动物

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA6ada05c942f0d6ecf3479293ee92ae4a.png" />

并不能盲目选择哪种工厂模式，例如抽象工厂模式更多的是使用在**一系列相互关联的基础产品，而这些产品共同组成一个成品**（产品族）上的。如果各个产品之间更类似于并列关系而不是关联关系的话，工厂方法模式更合适一些。设计模式本身就有其适用的场景，并不是滥用的，

# 参考资料
https://unity.com/cn/how-to/how-use-factory-pattern-object-creation-runtime
https://design-patterns.readthedocs.io/zh-cn/latest/creational_patterns/simple_factory.html
https://design-patterns.readthedocs.io/zh-cn/latest/creational_patterns/factory_method.html
https://design-patterns.readthedocs.io/zh-cn/latest/creational_patterns/abstract_factory.html
https://refactoringguru.cn/design-patterns/factory-method
https://refactoringguru.cn/design-patterns/abstract-factory
https://juejin.cn/post/7032133433031835678
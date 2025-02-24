---
title: Unity中的序列化与反序列化
date: 2021-03-14 13:45:46
tags: ['探索发现','游戏开发','序列化','反序列化','Unity']
categories: 探索发现
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAdb9e1d27869fa49926aa9627295fad49.png
---

# 什么是序列化与反序列化

**序列化**与**反序列化**是两个至关重要却又常常被开发者们忽视的概念。想象一下，当你精心设计了一个复杂的游戏角色，它拥有独特的属性、技能和装备，而你希望能够将这个角色的数据保存到本地，以便下次游戏时可以继续使用（**存档**与**读取存档**）；或者你需要将角色的数据通过网络发送给其他玩家，让他们也能看到你的角色，该怎么办呢？这时，序列化与反序列化就派上用场了。序列化能够将复杂的数据结构转换为一种可以存储或传输的格式，反序列化再将序列化文件还原成原来的数据结构。在本文中，我们将深入探讨 Unity 的序列化与反序列化，揭开它们神秘的面纱。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAdb9e1d27869fa49926aa9627295fad49.png" />

## 序列化
序列化是指将对象的状态信息转换为**可以存储或传输的形式**的过程。在 Unity 中，这个过程通常是将对象的属性和字段转换为**二进制字节流**、**JSON 字符串**、**XML 文档**等格式。以 JSON 为例，一个包含角色生命值、攻击力和防御力的角色对象，经过序列化后可以变成如下所示的文本文件：

```JSON
{
    "health": 100,
    "attack": 20,
    "defense": 15
}
```

这样的字符串就可以轻松地保存到本地文件，或者通过网络发送到其他设备了。除了常见的int、float、string数据之外，JSON 还支持基本数据类型的数组、列表、字典，或声明了 `[System.Serializable]` 的内部类（对于需要序列化的类，需要一个包装类）。

> **JSON**（JavaScript Object Notation）是一种轻量级的数据交换格式，易于人类阅读和编写，同时也易于机器解析和生成。在 Unity 中，`JsonUtility` 类提供了简单的 JSON 序列化和反序列化功能。当进行序列化时，Unity 会遍历对象的可序列化字段，将其转换为 JSON 格式的键值对。反序列化时，则根据 JSON 字符串中的键值对，将数据赋值给对象的相应字段。

## 反序列化
反序列化则是序列化的**逆向操作**，它把序列化后的数据重新**还原**为对象。当我们从本地文件读取之前保存的 JSON 字符串，或者从网络接收其他设备传来的 JSON 字符串后，就能够借助反序列化将其转化为原来的角色对象，以便在游戏里继续使用。以下是使用 Unity 中的 `JsonUtility` 类将上述 JSON 字符串反序列化为角色对象的示例代码：

```C#
using UnityEngine;

// 记得要加上System.Serializable，这样类能够被Unity序列化，否则无法读取json内容
[System.Serializable]
public class Character
{
    public int health;
    public int attack;
    public int defense;
}

public class DeserializeExample : MonoBehaviour
{
    void Start()
    {
        string json = "{\"health\": 100, \"attack\": 20, \"defense\": 15}";
        Character character = JsonUtility.FromJson<Character>(json);
        Debug.Log("Health: " + character.health + ", Attack: " + character.attack + ", Defense: " + character.defense);
    }
}
```

当然，序列化的方式不止包括 JSON，还有前面提到的二进制和 XML。让我们来看看它们都怎么用吧！

# XML
我相信平常接触前端开发比较多的同学对 XML 绝对不会陌生，这是一种类似于 HTML 的、使用标签来表示数据的数据结构（也因为这个特性，XML 文件容易变得冗长，可读性不如 JSON）：

```XML
<Person>
    <Name>Alice</Name>
    <Age>20</Age>
</Person>
```

类似于 JSON，**XML**（eXtensible Markup Language）也常用于数据的存储和传输，是一种标记语言。在 Unity 中，可以使用 `System.Xml.Serialization` 命名空间下的类进行 XML 序列化和反序列化。序列化时，会将对象的字段和属性转换为 XML 标签和属性；反序列化时，根据 XML 文档的结构将数据还原到对象中。


在语法上，XML 相对会更加复杂一些，涉及 `StringWriter` 的读写操作。

```C#
using System;
using System.IO;
using System.Xml.Serialization;
using UnityEngine;

[Serializable]
public class Enemy
{
    public string type;
    public int health;
}

public class XmlSerializationExample : MonoBehaviour
{
    void Start()
    {
        Enemy enemy = new Enemy { type = "Goblin", health = 50 };

        // 序列化
        XmlSerializer serializer = new XmlSerializer(typeof(Enemy));
        using (StringWriter writer = new StringWriter())
        {
            serializer.Serialize(writer, enemy);
            string xml = writer.ToString();

            // 反序列化
            using (StringReader reader = new StringReader(xml))
            {
                Enemy newEnemy = (Enemy)serializer.Deserialize(reader);
                // 测试读取结果
                Debug.Log("Deserialized Enemy: Type - " + newEnemy.type + ", Health - " + newEnemy.health);
            }
        }
    }
}
```

# 二进制文件
在几种序列化方式中，二进制是**可读性最差**的一种格式，但也是**安全性最高**、**体积最小**、**读写速度最快**的一种格式。在 Unity 中，可以使用 `System.Runtime.Serialization.Formatters.Binary` 命名空间下的 `BinaryFormatter` 类进行二进制序列化和反序列化。序列化时，对象的所有字段（包括私有字段）都会被转换为字节流；反序列化时，字节流会被还原为对象。

```C#
using System;
using System.IO;
using System.Runtime.Serialization.Formatters.Binary;
using UnityEngine;

[Serializable]
public class Item
{
    public string name;
    public int value;
}

public class BinarySerializationExample : MonoBehaviour
{
    void Start()
    {
        Item item = new Item { name = "Sword", value = 100 };

        // 序列化
        BinaryFormatter formatter = new BinaryFormatter();
        using (MemoryStream stream = new MemoryStream())
        {
            formatter.Serialize(stream, item);
            byte[] bytes = stream.ToArray();

            // 反序列化
            using (MemoryStream newStream = new MemoryStream(bytes))
            {
                Item newItem = (Item)formatter.Deserialize(newStream);
                // 测试读取结果
                Debug.Log("Deserialized Item: Name - " + newItem.name + ", Value - " + newItem.value);
            }
        }
    }
}
```

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA254b97ad74bdfb8e51031243c3ccfa8c.png" alt="序列化后的文件，完全看不出来原本是什么" width=400 />

# 实际应用
1. **数据持久化**
“数据持久化”，或者说“存档”，是现代大多数游戏必不可少的功能之一。无论是 JSON、XML 还是二进制序列化，都可以用于将游戏的进度、玩家的角色数据、游戏配置等信息保存到本地文件，以便下次游戏时继续使用。
2. **网络通信**
在多人在线游戏中，网络通信是必不可少的。通过序列化和反序列化，可以将游戏中的数据（如玩家的位置、动作、状态等）转换为可以在网络中传输的格式，然后发送给其他玩家；其他玩家接收到数据后，再通过反序列化将其还原为对象，以便在自己的游戏中显示。
3. **场景存储**
在 Unity 中，可以使用序列化和反序列化来保存和加载场景。通过将场景中的对象和它们的状态信息进行序列化，可以将整个场景保存到文件中；在需要加载场景时，再通过反序列化将其还原。
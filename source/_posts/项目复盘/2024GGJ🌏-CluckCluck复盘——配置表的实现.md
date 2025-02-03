---
title: "2024GGJ\U0001F30F - CluckCluck复盘——配置表的实现"
date: '2024-02-11'
updated: '2024-02-15'
tags:
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAcf0538525ce9dc3d4b37bf124f74df47.png
---

CluckCluck 的开发时间只有 48 小时，在这样短的开发时间内，作为程序的我们需要频繁对场景以及各类预制件做出修改，从避免提交冲突的角度考虑，我们没有办法让策划同学直接在Unity场景中配置道具的数据。

不过，我想到之前在 hll 学长的代码仓库里看到过**配置表工具**，结合实际开发中策划人员也是通过打表实现数据传递的，我决定在 CluckCluck 中使用这个功能，完美解决了策划的配置与调试可能影响到提交冲突的问题（好耶！

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA448722cab57cf85bc3d842b4e8b88996.png" alt="游戏内道具的配置表" />
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAfab0ba968a753dff48996e7092f12ddd.png" alt="游戏内武器的配置表" />

所以在本文中，我想剖析一下这个妙妙小工具，看一下它是怎么运作的。首先来看看工具的文件结构：

```ASCII
配置表
    |- Excels
    |- Scripts
        |- Plugins
        |- Foods
            |- FoodProps.cs
            |- FoodsInfo.cs
            |- Editor
                |- FoodsInfo_Editor.cs
        |- Weapons
        |- BaswExcel.cs
```

`Excels` 顾名思义，是存放 `.xlsx` 文件（也就是上面两张截图）的目录。让我们看看 `Scripts` 文件夹里头的东西吧！

# Plugins
`Plugins` 文件夹里有三个文件，分别是：

- **EPPlus.dll**：EPPlus 库的核心动态链接库，包含了 EPPlus 的所有功能代码。
- **EPPlus.pbd**：调试符号文件，包含了调试信息，可以用来帮助定位 EPPlus 库中的错误或异常（并非必须）。
- **EPPlus.xml**：EPPlus 库的 XML 文档，包含了库中所有类、方法和属性的注释说明。

这个 **EPPlus** 就是实现 excel 表格操作的核心工具，它究竟是何方神圣？EPPlus 是一个用于操作 Excel 文件的<a href="https://github.com/EPPlusSoftware/EPPlus">开源 .NET 库</a>，允许开发者在 C# 中创建、读取、编辑和格式化 Excel 文件（支持 .xlsx 格式），能够处理**单元格数据**、**公式**、图表、样式、数据验证、条件格式等功能。对于配置表工具来说，加粗的这两项工具就足够我们使用啦。

关于 EPPlus 的各个 API 如何使用，可以参阅<a href="https://epplussoftware.com/docs/7.2/articles/readme.html">官方文档</a>，在本文中不过多描述。

# BaseExcel.cs
这个脚本很简单，只包含了 `IndividualData` 一个类。该类用于存储 Excel 表中一行中每一列的数据。它的结构如下：

```C#
namespace 配置表 {
    public class IndividualData {
        public string[] Values;
        public IndividualData(int Columns) {
            Values = new string[Columns];
        }
    }
}
```

# 道具
Foods 和 Weapons 这两个文件夹一个代表**食物道具**，一个代表**武器道具**。它们本质上都是在通过配置表工具解析表格中数据并使用，在功能上是一样的，因此在下文中我们只介绍 `Foods` 文件夹的结构：

```ASCII
|- Foods
    |- FoodProps.cs
    |- FoodsInfo.cs
    |- Editor
        |- FoodsInfo_Editor.cs
```

## 道具属性（Properties）
道具属性对应 Foods 文件目录下的 `FoodProps.cs` 脚本，该脚本通过 **EPPlus API** 从记录了食品道具配置信息的 Excel 文件（`foods.xlsx`）中读取并解析数据，并将其转换为一个**字典**（Dictionary<int, IndividualData>），其中**键**是每行数据的**唯一标识**（ID），**值**是一个包含该行数据的 `IndividualData` 对象。

展开来说，`FoodProps.cs` 主要做了以下四件事：

1. **读取 Excel 文件**：
    - 从指定路径（Application.dataPath + "/配置表/Excels/foods.xlsx"）加载 Excel 文件。
    - 使用 EPPlus 库解析 Excel 文件内容。

2. **解析 Excel 数据**：
    - 读取第一个工作表（workSheets[0]）。
    - 遍历每一行（从第二行开始，**跳过表头**）。
    - 将每行的数据存储到 `IndividualData` 对象中。

3. **存储数据**：
    - 对于每一行，创建一个 `IndividualData` 对象，初始化其 `Values` 数组大小为 `CountOfAttributes`。
    - 遍历每一列（从第一列到最后一列），将单元格的值存储到 `Values` 数组中。
    - 将第一列的值（item.Values[0]）转换为整数，作为道具 **ID**。
    - 将 **ID** 和对应的 `IndividualData` 对象存储到字典中。

4. **返回结果**：
    - 返回构建好的字典，供其他代码使用。

```C#
using System;
using System.Collections.Generic;
using System.IO;
using OfficeOpenXml;
using UnityEngine;

namespace 配置表.Foods {
    public class FoodProps : MonoBehaviour {
        // 表中属性字段的数量
        public static int CountOfAttributes = 6;

        public static Dictionary<int, IndividualData> LoadExcelAsDictionary() {
            // 道具字典，用于存储以行为单位的各个道具
            Dictionary<int, IndividualData> ItemDictionary = new Dictionary<int, IndividualData>();
            // 表格的文件路径。编辑器模式下Application.dataPath就是Assets文件夹
            string path = Application.dataPath + "/配置表/Excels/foods.xlsx";
            // 建立文件流fs，用于读取 Excel 文件
            FileStream fs = new FileStream(path, FileMode.Open, FileAccess.Read, FileShare.ReadWrite);

            ExcelPackage excel = new ExcelPackage(fs);
            // 获取全部工作表
            ExcelWorksheets workSheets = excel.Workbook.Worksheets;
            // 只看第一个工作表
            ExcelWorksheet workSheet = workSheets[0];

            int colCount = workSheet.Dimension.End.Column;// 工作表的列数
            int rowCount = workSheet.Dimension.End.Row;// 工作表的行数

            // 从当前工作表的第二行遍历到最后一行(第一行是表头，所以不读取)
            for (int row = 2; row <= rowCount; row++) {
                // 用于接收本行数据
                IndividualData item = new IndividualData(CountOfAttributes);

                // 从第一列遍历到最后一列
                for (int col = 1; col < colCount; col++) {
                    // 读取每个单元格中的数据，写入IndividualData
                    item.Values[col - 1] = workSheet.Cells[row, col].Text;
                }
                // 获取道具ID
                int itemID = Convert.ToInt32(item.Values[0].ToString());
                // 将ID和IndividualData写入字典
                ItemDictionary.Add(itemID, item);
            }

            Debug.Log("complete");
            return ItemDictionary;
        }
    }
}
```

## 道具信息（Information）
`FoodsInfo.cs` 这个脚本定义了两个类：`FoodSettings` 和 `FoodsInfo`，主要用于在 Unity 中管理和存储食物道具的具体配置项信息。

- `FoodSettings`：用于存储食物的配置属性（如 ID、名称、是否为buff、体积、速度、buff时间等）。
- `FoodsInfo`：可以挂载到 Unity 的游戏对象上，是策划配置好表格后在 Unity 中测试的工具（结合下一小节的编辑器工具使用）。
    - 包含一个 `FoodSettings` 类型的字段，用于在编辑器中显示存储具体的食物配置。
    - 包含一个 `InitFromID` 字段，用于指定从配置表中加载的食物的 ID。

```C#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using System;

[Serializable]
public class FoodSettings
{
    public int id;
    public string name;
    public int is_buff;
    public float volume;
    public int speed;
    public int buff_time;
}

public class FoodsInfo : MonoBehaviour
{
    public FoodSettings Settings;

    [Header("配表内ID")]
    public int InitFromID;
}
```

## 编辑器工具（Editor）
**Editor** 文件夹里包含了一个自定义编辑器脚本，通过增加一个按钮扩展了 `FoodsInfo` 组件在 Unity Inspector 中的功能，允许策划同学从配置表中**加载指定 ID** 的食物道具数据，并将其赋值给 FoodsInfo 组件的 Settings 字段。

这样，策划同学就可以直观地看到配置表中的数据了：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAc144cc496b1780b28109a81b16229524.png" width=300 />

```C#
using UnityEngine;
using UnityEditor;
using System;
using 配置表;
using 配置表.Foods;

// 指定为FoodsInfo组件的编辑器自定义模块
[CustomEditor(typeof(FoodsInfo),true)]
public class FoodsInfo_Editor : Editor {
    public override void OnInspectorGUI() {
        DrawDefaultInspector();// 绘制常规内容

        // 添加刷新配置按钮
        if (GUILayout.Button("从配表ID刷新食物数据")) {
            FoodsInfo foodsInfo = (FoodsInfo)target;
            Init(foodsInfo);
        }
    }

    // 刷新配置
    public void Init(FoodsInfo instance) {
        Action initfood;

        var dictionary = FoodProps.LoadExcelAsDictionary();

        if (!dictionary.ContainsKey(instance.InitFromID)) {
            Debug.LogErrorFormat("未能在 foods 配表中找到指定的ID:{0}", instance.InitFromID);
            return;
        }
        IndividualData item = dictionary[instance.InitFromID];

        initfood = (() => {
            instance.Settings.id = Convert.ToInt32(item.Values[0]);
            instance.Settings.name = Convert.ToString(item.Values[1]);
            instance.Settings.is_buff = Convert.ToInt32(item.Values[2]);
            instance.Settings.volume = Convert.ToSingle(item.Values[3]);
            instance.Settings.speed = Convert.ToInt32(item.Values[4]);
            instance.Settings.buff_time = Convert.ToInt32(item.Values[5]);
        });

        initfood();
    }
}
```

# 参考资料
https://github.com/EPPlusSoftware/EPPlus
https://epplussoftware.com/docs/7.2/articles/readme.html
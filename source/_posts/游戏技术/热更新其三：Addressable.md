---
title: 热更新其三：Addressables
date: '2024-09-14'
tags: ['AssetBundle','Addressable', '打包','资源管理', '热更新', 'Unity']
categories: 引擎功能
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA897df47010021e029beb2af054f892a9.png
---

Addressables（可寻址资源系统）是 Unity 新推出的用于智能管理 AssetBundle 资源和加载的工具包，它为 AssetBundle 提供了更高层次的抽象，旨在取代 AssetBundle Manager。

{% note success %}
Addressables 允许开发人员通过一个独一无二的**地址**字符串请求资产，无需关心底层复杂的资源加载逻辑。
一旦资产（例如预制件）被标记为“Addressable”，它就会生成一个可以从任何地方调用的地址，**无论资产位于何处**（本地或远程），系统都会找到它及其依赖项，然后将其返回，非常简单易用。
{% endnote %}

# 与AssetBundle的关系
Addressables 的底层基于 AssetBundle，相比于通过自己写代码造轮子或第三方插件去管理 AB 包，Addressables 提供了一个直观易用的图形管理界面，同时可以帮助开发者自动处理 AssetBundle 的**生成**和**依赖关系**，减轻项目管理的压力和打包遇到问题的几率。

{% note orange 'fas fa-lightbulb' simple %}
**AssetsBundle的依赖关系**
有一些 AB 包之间存在依赖关系，这是什么意思呢？比方说某个 AB 包在的 Prefab A 使用了一个材质 M，而材质 M 存在于另一个 AB 包中，那么在加载 Prefab A 时必须确保材质 M 所在的 AB 包也被加载，否则资源就会缺失。在这种情况下，我们就可以说预制件所在的 AB 包和材质所在的 AB 包存在依赖关系。
{% endnote %}

**AssetBundle 的不足**：
- 手动打包资源，并处理依赖关系。
- 手动加载和卸载，容易导致内存管理问题。

**Addressables 的优势**：
- 自动处理 AssetBundle 的生成和依赖。
- 提供更简单的 API 和资源管理机制。
- 支持异步加载和高效的内存回收。

# 在项目中使用Addressables
首先，在 Package Manager 中引入 Addressable 包：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA720ce99f71cf4e1ea671e5398dd661de.png" width=500 />

随后，在 Addressables Groups 窗口（Window > Asset Management > Addressables > Groups）中，我们可以新建一个 Addressables 设置：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA8ae85d4bbc58b838b2e18ce859e3e1f6.png" width=450 />

对于我们希望用 Addressables 去管理的素材来说（比如一个 Canvas 的预制件），我们可以直接勾选检查器上的 Addressables 选框，或者将希望由 Addressables 管理的文件拖入 Group 视图里，资源就会出现在组里了。下图中组的名字是 `Default Local Group`，我们也可以给组改个名字，方便后续的资源分类和管理。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA16469cac32adf8665a93ef2ed17011d8.png" />

## 加载资源
在脚本中，我们可以通过直接挂载的方式去加载 Addressables 里的资源：

``` C#
using UnityEngine;
using UnityEngine.AddressableAssets;

public class Loader : MonoBehaviour {
    [SerializeField] private AssetReference _canvas;
    // Start is called before the first frame update
    void Start() {
        _canvas.InstantiateAsync().Completed += (handle) => {
            print("Canvas Loaded Successfully!");
        };
    }
}
```

当然，当项目资源体系变得很庞大的时候，这样手动挂载的方式是不太现实的（也很容易出现管理混乱的问题）。我们可以通过**地址路径**去加载资源：

```C#
using UnityEngine;
using UnityEngine.AddressableAssets;
using UnityEngine.ResourceManagement.AsyncOperations;

public class Loader : MonoBehaviour {
    [SerializeField] private string address;
    // Start is called before the first frame update
    void Start() {
        Addressables.LoadAssetAsync<GameObject>(address).Completed += OnLoadComplete;
    }
    void OnLoadComplete(AsyncOperationHandle<GameObject> obj) {
        if (obj.Status == AsyncOperationStatus.Succeeded)
            Instantiate(obj.Result);
        else
            Debug.LogError("加载资源失败：" + address);
    }
}
```

那这个地址怎么获取呢？在 Addressables Groups 的管理界面，找到需要的资源拷贝路径即可：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA315fbed974d3e827560fcee0595508b4.png" width=400 />

## 本地资源
在 Addressables Group 的检查器窗口中，可以找到两个路径：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA36efc7687c8513a0bb004ff20bb12a15.png" width=400 />

其中 Build Path 指的是**构建路径**，通过 Addressables Group 选项卡的 Build 构建完成之后，打好的资源包所存储的位置（比如我们存的是一个 canvas 的预制件）：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAcdeb641ba07c47abb9da778067c21900.png" width=350 />
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA69d129346df271d94f6d639bd05beb7b.png" width=300 />

Load Path 指的是**加载路径**，决定游戏运行时从哪里加载资源，可能是本地存储，也可能是远程服务器。

### 实现本地热更新
万事俱备，只欠东风，我们该如何通过 Addressables 实现本地资源热更新呢？其实我们只需要替换打包好的项目文件的构建路径里的资源就可以了。首先，把项目先打包出来：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA489952dfb18de75dde03774efa15137d.png" width=400 />

在 _Data 文件夹中，可以找到对应的构建路径：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAd285c569621576c647db84f443be7520.png" width=300 />

假设现在策划提出需求，需要更改 canvas 中的素材，例如将图片底色从白色改成蓝色。按照传统的思路来说，我们可能会修改之后重新打包，然后把新的包发给玩家。

但借助 Addressables，我们就可以实现本地热更新了。对于修改后的 canvas 预制件，我们可以对其重新构建 Addressables Group，然后在构建路径中找到 bundle 文件，用新构建的 bundle 文件和目录（如下图，如果没有一块复制过去会导致找不到 bundle）替换 _Data 文件夹中对应的文件：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA5a514e6d9563f40a7e2095494424d6dc.png">

再次运行游戏，就会发现替换完成了。

{% note blue 'fas fa-question' %}
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAe5d15204c4714852b3bb60906bb6f7a0.png" width=300 />

诶，安卓包怎么没有构建路径呢？这是因为安卓平台应用在打包时资源嵌入到 APK 的 assets 目录里了，在运行时不可修改，所以没有暴露出来。那么对于安卓平台来说怎么实现热更新呢？这就需要我们使用远端来进行更新了。
{% endnote %}

## 远端资源
Addressables 同样支持将资源托管在远程服务器上，这样可以减少初始安装包的体积，更新游戏内容时拉去需要更新的包就可以了。在 Addressables 的 Profile 界面中，我们可以自行设置远端加载路径的位置，例如用本机测试的话可以是：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA0a787c2021ab7afeaa39b7e269c9504b.png">

接着，在 Addressables 的设置（Settings）中，勾选**构建远端目录**：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA7b3f5cf0eb208fbba11bb76ddb306123.png" width=400 />

然后我们就可以正常的对资源进行打包了。我们可以在项目文件结构（或者其他的任何地方）下新建一个文件夹 Server，用 Node.JS 结合 express 来模拟服务器行为。Node.JS 需要在官网下载，安装完成后用 VSC 打开文件夹，在终端执行安装命令即可。

{% note info %}
**什么是 Express？**

Express 是 Node.js 的一个轻量级 Web 应用框架，用于快速构建服务器端应用程序和 API。其核心功能是提供 HTTP 路由和中间件支持，让开发者可以定义 API 端点、处理请求/响应以及管理服务器逻辑。
{% endnote %} 

安装完成后，我们可以在服务器目录下新建一个 `Server.js`，用于启动静态服务并监听 3000 端口：

```JavaScript
const express = require('express')
const app = express()
const port = 3000
// 使用 public 文件夹存储
app.use(express.static('public'))
// 监听 3000 端口
app.listen(port, ()=>{
    console.log(`Server running - port:${port}$`)
})
```

### 实现远端热更新
当资源发生变更时（例如改变了 canvas 预制件中材质的颜色），我们可以直接通过 Addressables Groups 界面的 Update a Previous Build 来更新发现变更的资源包。点击更新时，选择 AddressableAssetsData 路径下的 .bin 文件（表示 AB 包的状态）即可完成 AB 包的更新。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA2a9f383796f4ba777cffabf812ae4963.png" />
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA9d3f4f590a1ba31146c8cc3bf8c201bd.png" width=350 />

随后，我们的远端构建目录（在前面我们设置成了 ServerData）里就有了新的包的数据。把这些数据拷贝到服务器文件目录的 `public` 里，再通过 `node server.js` 命令开启服务器监听：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAe7d98d3ab02000cc77104ddcb592e44b.png" />

现在，更新后的 AB 包就在服务器上了，可以通过**加载路径**（也就是localhost:3000）加载到。现在直接运行游戏文件，就会发现资源已经变成更新过的 AB 包资源了。此时即使我们关闭服务器，再次运行游戏也是更新后的资源，因为本地 AB 包已经更新成新的啦。

# 参考资料
https://docs.unity3d.com/Manual/com.unity.addressables.html
https://docs.unity3d.com/Packages/com.unity.addressables@2.3/manual/index.html
https://www.youtube.com/watch?v=bCObS3teFGM
https://www.youtube.com/watch?v=dfSws1QQmKo
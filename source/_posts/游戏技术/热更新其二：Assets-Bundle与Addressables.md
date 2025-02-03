---
title: 热更新其二：AssetBundle初窥
date: '2024-09-12'
tags: ['AssetBundle','打包','资源管理', '热更新', 'Unity']
categories: 引擎功能
cover: https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAbb638bd203617ec12ceff2586949cccd.png
---

在项目开发的过程中，我们经常会遇到“版本更新”或者“资源补丁”，例如游戏素材随着活动而发生改变等等：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAb3f0a996b5c542dc8599987a56573e35.png" alt="旧版本的洛克王国地图" width=400>

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAb671a966f5aad54c07bea0ef4d61c8df.png" alt="新版本的洛克王国地图" width=400>

在上一篇文章中我们也提到过了，如果每次更新都让用户重新下载客户端对产品留存来说是一大挑战，因为大家都很懒，而且卸载重装的流量也是要钱的。在实际项目中，项目组都会使用热更新将基础功能模块和业务逻辑分开，方便做热更新。

在 Unity 中实现资源热更新的方式有 **AssetBundle** 和 **Addressable**，一般大家都用前者（因为轮子多），后者是 Unity 推出的比较新的资源管理系统（旨在替换 AB 包）。

# AssetBundle
<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAb83cfa80dd6af78763d5daa7bdfcba8f.png">

我们常说的“打包”其实指的就是打 AB 包。“bundle”一词就是“a bunch of ...”的意思，因此顾名思义，asset bundle是一捆资源的意思。

{% note info %}
AssetBundle 是一个存档文件，包含可在运行时由 Unity 加载的特定于平台的非代码资源（比如**模型**、**纹理**、**预制件**、**音频**甚至整个**场景**）。

AssetBundle 可以表示彼此之间的依赖关系，例如一个 AssetBundle 中的材质可以引用另一个 AssetBundle 中的纹理。为了提高通过网络传输的效率，可以根据用例要求（**LZMA** 和 **LZ4**）选用内置算法选择来压缩 AssetBundle。
{% endnote %}

AssetBundle 工作流通常包含以下几步：
1. 设置资源的 BundleName 以及 BundleVariant
2. 打包 AssetBundle
3. （将AB包上传到服务端，若放在本地则可跳过）
4. 加载 AssetBundle
5. 实例化 AssetBundle 中的资源
6. AssetBundle的卸载

Unity 的资源加载有两种方式：**Resource.Load**\<T\> 和上文中提到的 **AssetBundle.LoadFromFile** + **LoadAsset**\<T\>。通过 Resource 加载资源时，Resource 里面的资源在构建项目时会自动打包到一个文件中，然后就可以通过路径找到对应的资源了，这样用起来比较简单方便，但 Resources 文件夹中的资源会全部加载到内存中（即使部分资源未使用），这会导致内存浪费。AssetBundle 的好处在于可以**按需加载**对应的包，减少内存占用，还能通过更新包的方式实现热更新，缺点就是需要手动管理资源的加载和卸载，使用前需提前打包资源。

所以对于资源来说，可以分成一些始终需要加载的**静态资源**和需要变更的**动态更新资源**；对静态资源使用 Resource、对动态资源使用 AB 包的方式加载。对 Unity 初学者来说，所接触到的都是偏小的项目/demo，因此使用 Resource 就可以满足绝大多数需求了。但在大型项目中，通常都有数量庞大的资源，外加对于热更新的需求，采用 AssetBundle 的情况更多。

## 在项目中引入AssetBundle

要使用 AssetBundle，需要在 Assets 文件夹下新建一个 Editor 文件夹，然后将 Unity 官方给出的 <a href="https://docs.unity3d.com/cn/2021.1/Manual/AssetBundles-Workflow.html">CreateAssetBundles</a> 脚本包含进 Editor 文件夹中。Editor 文件夹是 Unity 的编辑器工具，我们可以通过这个文件夹制作一个引擎妙妙工具来提高开发效率，在其中的内容不会被包含在正式的游戏包中。

```C#
public class CreateAssetBundles {
    [MenuItem("Assets/Build AssetBundles")]
    static void BuildAllAssetBundles() {
        string assetBundleDirectory = "Assets/AssetBundles";
        if(!Directory.Exists(assetBundleDirectory)) {
            Directory.CreateDirectory(assetBundleDirectory);
        }
        BuildPipeline.BuildAssetBundles(assetBundleDirectory, 
            BuildAssetBundleOptions.None, 
            BuildTarget.StandaloneWindows);
    }
}
```

这个脚本就是一个编辑器工具，完成之后点击 Unity 菜单栏的 Assets 标签，我们就可以找到加入的“Build AssetBundles”选项了：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA94f62946be7372afd788b2f5271099eb.png" width=300/>

代码声明了一个资源路径 `assetBundleDirectory`，用于让 `BuildPipeline` 注册所有的AB包。`BuildTarget` 指的是项目构建的目标平台，在这里是 Windows，也可以直接设置成 `EditorUserBuildSettings.activeBuildTarget`，根据设置的构建平台自动配置打包平台。

接着，对于我们要热更新的资源对象（例如一些贴图素材），在检查器中我们会发现底部多了一行与 AB 包有关的设置：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VAe010090f191679a65f194799558959df.png">

点击第一个下拉选框的 new，对打的资源 AB 包进行命名（例如“UI_asset”），我们就可以创建出第一个 AssetBundle 了：

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA41aec8ffe08a2a1556438f8ea296d421.png" />

{% note info %}
**关于AssetBnudle Variants**

你是否好奇AssetBundle界面的第二个下拉选框是什么？这就是 Asset Variants，也就是**变体**。
变体是 Unity 中的一个常见的概念。我们可以在创建预制件时从一个预制件延伸出多个变体（不同的**版本**），方便我们对资源进行更精细的控制。那么 AssetBnudle 的 Variants 是做什么的呢？类似于预制件的变体，AssetBnudle Variants 可以将资源打包成不同的版本，在游戏中根据不同的需求载入不同的 AB 包，可以实现**多语言版本**游戏的功能，可参考<a href="https://dev.twsiyuan.com/2017/05/unity-assetbundles-variants.html">这篇文章</a>。

{% endnote %}

完成后再点击“Build AssetBundles”选项，Unity 就会开始打 AB 包了。打包的时间可能比较漫长，最长可以到 30+ 分钟，而且出了问题还得重新打，所以每次到打包的时候都会比较痛苦。打包结束后，之前代码中设置的资源路径 `assetBundleDirectory` 中就会出现打好的 AB 包，其中包含了一个打好包的 AssetBundle 文件和一个**文本形式**的 manifest 描述文件。文件里头就是资源，manifest 则类似一个目录，描述了 AB 包的名称、所包含的资源概述（Assets Included）、依赖关系（Dependencies）、版本、CRC码和哈希值等信息，方便加载和管理资源。

>- manifest 还有一个**二进制形式**，被包含在打好包的 AssetBundle 里了。
>- 哈希值用于校验该 AB 包的完整性，特别适合热更新的**版本控制**。
>- CRC即我们所熟知的**循环冗余校验码**，用于快速验证数据完整性。

```
ManifestFileVersion: 0
CRC: 1148559550
Hashes:
  AssetFileHash:
    serializedVersion: 2
    Hash: 4d8aca9981db63069a87bc01f8386329
  TypeTreeHash:
    serializedVersion: 2
    Hash: c21d61fe394c58399df3c6fb5c7c99b3
HashAppended: 0
ClassTypes:
- Class: 28
  Script: {instanceID: 0}
- Class: 213
  Script: {instanceID: 0}
SerializeReferenceClassIdentifiers: []
Assets:
- Assets/ArtMat/pg3/my.png
- Assets/ArtMat/pg2/serv.png
Dependencies: []
```

## 在项目中加载AssetBundle
### 本地加载
如果将 AB 包打在本地而不选择上传到云端，可以直接通过路径加载包里的资源，例如使用 `AssetBundle.LoadFromFile` 方法。从本地存储中加载未压缩的捆绑包时 `AssetBundle.LoadFromFile` 非常高效，因为只需占用解析后的 AssetBundle 内存，而无需额外的原始数据内存。

如果捆绑包**未压缩**或采用了 **LZ4**，`AssetBundle.LoadFromFile` 将直接从磁盘加载捆绑包。使用此方法加载 **LZMA** 捆绑包将首先解压缩捆绑包，然后再将其加载到内存中。

```C#
var myLoadedAssetBundle = AssetBundle.LoadFromFile(Path.Combine(Application.streamingAssetsPath, "myassetBundle"));
// 找不到资源
if (myLoadedAssetBundle == null) {
    Debug.Log("Failed to load AssetBundle!");
    return;
}
var prefab = myLoadedAssetBundle.LoadAsset<GameObject>("MyObject");
Instantiate(prefab);
```

这里提到了一个 `Application.streamingAssetsPath` 的概念，指的是 Unity 的一个特殊目录 StreamingAssets（特殊目录即类似 Resources，Editor，Plugin 这样的目录，在Rider 中会有角标显示）。

<img src="https://b.bdstatic.com/comment/HPpFm-ziUYsgpwpjCcQ1VA7f179d570838c4c11a11420387303f0c.png">

这是一个**只读**、**不可写**的目录，该文件夹下的资源会保持原始格式（比如图片不会被引擎进行纹理压缩），dll文件或者脚本放在该文件夹下也不会参与编译。Unity 官方推荐使用 `Application.streamingAssetsPath` 来获得该文件夹的实际位置，其可以规避平台差异。

{% note warning %}
Unity 会将该目录下的所有文件都打包，因此需要特别注意的是：在打包不同平台的应用程序时需要避免将其他目标平台的 AB 包一起打包，否则将大大增加包体大小。如：Android 的 apk 包就应该确保 StreamingAssets 目录下只有 Android 平台的资源，其他平台的资源都是没用的，只会增加包体大小。
{% endnote %}

我们也可以通过 `UnityEngine.Networking.UnityWebRequest` 和 `AssetBundle.LoadFromMemoryAsync` 实现本地资源的获取，例如：

```C#
IEnumerator InstantiateObject()
{
    string uri = "file:///" + Application.dataPath + "/AssetBundles/" + assetBundleName; 
    UnityEngine.Networking.UnityWebRequest request 
        = UnityEngine.Networking.UnityWebRequest.GetAssetBundle(uri, 0);
    yield return request.Send();
    // 获取到包
    AssetBundle bundle = DownloadHandlerAssetBundle.GetContent(request);
    // 获取包内的具体资源
    GameObject cube = bundle.LoadAsset<GameObject>("Cube");
    GameObject sprite = bundle.LoadAsset<GameObject>("Sprite");
    // 实例化
    Instantiate(cube);
    Instantiate(sprite);
}
```

和第一种方法不同的是，`UnityEngine.Networking.UnityWebRequest` 是异步的（比如这里使用了一个协程），整个过程不会阻塞主线程，比较适合加载大型资源的场合。

`AssetBundle.LoadFromMemoryAsync` 则将内存中的字节流数据解析为可用的 AssetBundle 对象。这个过程也是异步的，相比于 `AssetBundle.LoadFromFile` 多了需要同时存储文件字节流和解析后的 AB 数据这一步，内存占用更高，加载效率也比较慢（额外的字节流读取和传递增加开销）。

```C#
IEnumerator LoadFromMemoryAsync(string path) {
    AssetBundleCreateRequest createRequest = AssetBundle.LoadFromMemoryAsync(File.ReadAllBytes(path));
    yield return createRequest;
    AssetBundle bundle = createRequest.assetBundle;
    var prefab = bundle.LoadAsset<GameObject>("MyObject");
    Instantiate(prefab);
}
```

### 远程加载
更多时候，资源是托管在服务器上的，本地客户端通过对比版本号决定是否要拉取某些 AB 包的资源进行更新。我们可以通过前面提到过的 `UnityEngine.Networking.UnityWebRequest`，将 url 从本地路径替换成网络资源就可以实现远端下载了。下面是一个通过协程实现的 AB 包资源请求逻辑：

```C#
IEnumerator InstantiateObject()
{
    string uri = "http://example.com/mybundle/" + assetBundleName; 
    // GetAssetBundle 获取 AssetBundle 的位置 URL 以及要下载的捆绑包的版本。
    UnityEngine.Networking.UnityWebRequest request 
        = UnityEngine.Networking.UnityWebRequest.GetAssetBundle(uri, 0);
    yield return request.Send();
    // 下载到包
    AssetBundle bundle = DownloadHandlerAssetBundle.GetContent(request);
    // 获取包内的具体资源
    GameObject cube = bundle.LoadAsset<GameObject>("Cube");
    GameObject sprite = bundle.LoadAsset<GameObject>("Sprite");
    // 实例化
    Instantiate(cube);
    Instantiate(sprite);
}
```

## 压缩格式
AB 包的压缩格式可以选择设置为**无压缩**、**LZMA** 和 **LZ4**，那么它们之间有什么区别呢？

{% tabs 不同压缩格式的区别 %}

<!-- tab 无压缩 -->

无压缩的 AB 包又名 Uncompressed AssetBundle，即不对 AB 包做任何压缩，数据以原始格式存储。未压缩的 AssetBundles 是 16 字节对齐的。在代码中，通过 `BuildAssetBundleOptions.UncompressedAssetBundle` 设置。

**特点**：
1. 加载速度快：直接从磁盘读取，不需要解压缩。
2. 文件体积大：由于没有压缩，AB 包占用的存储空间较多。

**适用场景**：
- 磁盘空间充裕且需要最快加载速度的情况。
- 用于开发阶段调试，因为无压缩的包加载速度更快，调试更高效。
<!-- endtab -->

<!-- tab LZMA -->

LZMA 全称 “Lempel-Ziv-Markov chain Algorithm”，用于 7-Zip 归档工具中的 7z 格式和 Unix-like 下的 xz 格式。它是 AssetBundle 的默认压缩格式，压缩率很高，可以将 AB 包压缩到**最小体积**。在代码中，通过 `BuildAssetBundleOptions.None` 设置。

**特点**：
1. 压缩率高：生成的 AB 包文件体积最小。
2. 加载时解压慢：LZMA 是基于整个文件的压缩算法，加载时需要一次性**解压整个包**到内存。
3. 需要解压后存储到临时文件：解压完成后数据会存放到临时缓存路径。

**适用场景**：
- 网络下载时使用，适合初次下载资源（因为传输体积小）。
- 不适合频繁加载的资源。
<!-- endtab -->

<!-- tab LZ4 -->

LZ4 全称“Lempel-Ziv 4”，是一种注重解压速度的轻量级压缩算法，支持**流**式解压（这是因为 LZ4 使用基于**块**的算法，该算法允许 AssetBundle 分段或“块”加载）。在代码中，通过 `BuildAssetBundleOptions.ChunkBasedCompression` 设置。

**特点**：
1. 压缩率较低：生成的 AB 包体积比 LZMA 大，但比无压缩小。
2. 解压速度快：支持流式解压，加载时只解压需要部分，无需解压整个文件（速度和无压缩相当）。
3. 占用内存较低：加载时不需要额外的内存空间存储解压后的数据。

**适用场景**：
- 实时加载的资源（如场景切换时加载的资源）。
- 热更新时常用，因为平衡了体积和加载速度。
<!-- endtab -->

{% endtabs %}

默认情况下，Unity 通过 LZMA 压缩来创建 AssetBundle，然后通过 LZ4 压缩将其**缓存**。这是从内容分发网络 (CDN) 下载的 AssetBundle 的首选格式，因为文件大小小于使用 LZ4 压缩的文件。但这也意味着，如果我们需要读取某个资源，在开始时就必须将整个流解压缩。不过实际上消耗的时间比预期的会更长一些，因为最初加载缓存的 LZMA AssetBundle 所花费的时间更长，因为 Unity 必须将存档重新压缩为目标格式再存入缓存中，随后的加载将使用缓存版本。

缓存分为内存缓存和磁盘缓存两种：

- **内存缓存**：以 UncompressedRuntime 格式将 AssetBundle 存储在 **RAM** 中。虽然加载速度很快，但这会占用大量的内存资源，性价比不高。
- **磁盘缓存**：将提取的 AssetBundle 以 LZ4 或不压缩的格式存储在**磁盘**中，具体采用哪种压缩格式与 `Caching.compressionEnabled` 参数的设置有关。磁盘缓存减少了内存占用，但可能增加磁盘空间的使用。

{% note warning %}
如果在使用 UnityWebRequest API 下载 AssetBundle 时提供了版本参数（版本号或哈希值），Unity 会将下载的数据存储在磁盘缓存中，否则将存储在内存缓存中。
{% endnote %}

对于 LZMA 压缩的 AssetBundle，建议使用 UnityWebRequest 来加载，以利用磁盘缓存，减少内存占用。直接使用 `AssetBundle.LoadFromFile` 或 `AssetBundle.LoadFromFileAsync` 会导致使用内存缓存，增加内存消耗。如果无法使用 UnityWebRequest，可以使用 `AssetBundle.RecompressAssetBundleAsync` 方法将 LZMA 压缩的 AssetBundle 重新压缩并保存到磁盘，以优化后续加载性能。

在选择缓存策略时，需要在内存使用、磁盘空间占用和资源加载时间之间进行权衡。使用磁盘缓存可以显著减少 RAM 的使用，但可能增加磁盘空间的需求。因此，应根据具体应用场景和性能要求，选择最适合的缓存和压缩策略。

# 参考资料
https://docs.unity3d.com/cn/2021.1/Manual/AssetBundles-Workflow.html
https://docs.unity3d.com/cn/2021.1/Manual/AssetBundles-Cache.html
https://zhuanlan.zhihu.com/p/342694796
https://zhuanlan.zhihu.com/p/484137488
https://dev.twsiyuan.com/2017/05/unity-assetbundles-variants.html#google_vignette
---
tags: ['游戏编程','经验分享','代理模式','设计模式','Unity']
categories: 设计模式
title: 游戏中的代理模式
date: '2022-01-06T15:37:17.071Z'
updated: '2022-01-06T17:52:21.258Z'
cover: https://i0.wp.com/wx4.sinaimg.cn/large/006UcwnJly1i6wr878wmbj30qo0goaj6.jpg
---

# 什么是代理模式？
代理模式是 一种结构型设计模式，它为其他对象提供一种替身或代理，以控制对这个对象的访问。通俗地说，客户端并不直接与**真实对象**（RealSubject）交互，而是通过**代理**（Proxy）来间接访问真实对象。代理可以在请求到达真实对象之前或之后添加额外行为（如懒加载、缓存、权限检查、日志等）。

## 解决问题
- 控制对资源或对象的访问（权限／保护）。  
- 延迟、优化或避免昂贵对象的创建（虚拟代理）。  
- 在分布式场景下通过本地对象代表远端对象（远程代理）。  
- 增强对象（如加入缓存、日志、事务、重试机制）而不修改真实对象代码。 

## 代理类型
1. **虚拟代理（Virtual Proxy）**：推迟创建昂贵对象，只有在真正需要时才实例化。  
2. **保护代理（Protection Proxy）**：根据调用者权限决定是否转发请求。  
3. **远程代理（Remote Proxy）**：本地代理对象代表远程对象，封装网络通信细节。  
4. **缓存 / 智能代理（Caching / Smart Proxy）**：在代理中缓存结果或添加智能处理逻辑。  

## UML说明
<img src="https://raw.githubusercontent.com/Guiny-Time/PictureBed/main/20220103031718.png"/>

典型参与者包括：

- **Subject**：接口或抽象类，定义公共方法。  
- **RealSubject**：真实对象的具体实现。  
- **Proxy**：实现 `Subject`，内部持有 `RealSubject` 引用。  

客户端仅依赖 `Subject` 接口，与 `Proxy` 或 `RealSubject` 互换。

### 优点
- 分离目标对象：代理模式可以将代理对象与实际的被称为目标对象分离。
- 降低耦合：在一定程度上降低了系统耦合，具有良好的可扩展性。
- 保护目标对象：代理类表示目标对象的业务逻辑。客户端直接与代理类交互，并且客户端与实际目标对象之间没有关联。
- 增强目标对象：代理类可以在目标对象的基础上添加新的函数。

### 缺点
- 过度使用代理会引入复杂性与性能开销。
- 线程安全问题：懒加载代理需考虑并发场景，防止多次实例化。
- 安全风险：远程代理或智能代理设计不当会导致访问控制失效，应在服务端强制校验。

# 应用场景
遵循前文中的UML结构，该示例展示了虚拟代理的延迟加载机制：仅在真正调用 display() 时才创建真实对象。
1. 真正的图像类运行加载、展示图像
2. 代理图像类，用户传入图像文件名，在需要时通过代理类的display加载并显示图像
```java
interface Image {
    void display();
}

class RealImage implements Image {
    private String filename;
    public RealImage(String filename) {
        this.filename = filename;
        loadFromDisk();
    }
    private void loadFromDisk() {
        System.out.println("Loading " + filename);
    }
    public void display() {
        System.out.println("Displaying " + filename);
    }
}

class ImageProxy implements Image {
    private String filename;
    private RealImage realImage;
    public ImageProxy(String filename) {
        this.filename = filename;
    }
    public void display() {
        if (realImage == null) {
            realImage = new RealImage(filename);
        }
        realImage.display();
    }
}
```

为什么要多这一层代理呢？感觉好像很麻烦，不直接用RealImage看上去更简单。但问题在于：RealImage的创建成本很高，比如：
- 图片文件很大，加载耗时。
- 图片来自网络，下载延迟明显。
- 系统内存有限，不能一次性加载所有资源。

而 ImageProxy 可以做到按需加载，只有当真的需要显示图片时，才去创建真实对象。换句话说，**Proxy 是为了延迟昂贵操作、控制访问、或添加额外逻辑**。

## 分块加载
在游戏开发中，虚拟代理的思想随处可见。比如开放世界游戏的场景贴图、模型、音效，不会一开始就全加载，当玩家走近某个区域，才通过“代理”去触发资源加载。

```C#
// Unity伪代码
class CharacterProxy : ICharacter {
    private string prefabPath;
    private Character realCharacter;

    public CharacterProxy(string path) {
        prefabPath = path;
    }

    public void Render() {
        if (realCharacter == null) {
            realCharacter = LoadFromDisk(prefabPath);
        }
        realCharacter.Render();
    }
}
```

这样游戏就不会因为一口气加载上千个NPC而卡顿。

# 参考资料
1. Refactoring.Guru — Proxy Pattern
2. Wikipedia — Proxy Pattern
3. Java Design Patterns 示例项目
4. CertiK Blog — Proxy Design and Security Risks
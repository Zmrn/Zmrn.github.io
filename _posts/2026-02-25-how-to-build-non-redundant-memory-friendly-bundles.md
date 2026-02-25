---
layout: post
title: "如何构建零冗余又内存友好的资源打包方案？"
categories: unity
---

> 本文会简单介绍下使用Unity的AssetBundle进行打包时可能遇到的一些问题，之后列出几个常见的分包算法并分析其利弊，最后给出一个可以解决前述算法中提到的问题的分包算法。
> 只是一些笔者处理资源打包相关问题时的研究结果，用于优化资源的打包方案。自认为可以用于大多数的情况，但未必适用于所有情况。有疑问或更好的解决方案的话欢迎讨论交流。

# 一个简单的冗余
如果游戏中有两个角色的预制件A和B，他们可以出现在地图上进行战斗。同时，他们持有着同一张图片的引用。

![whiteboard_exported_image](/assets/whiteboard_exported_image.png)

图中蓝色块代表预制件（或其他会被直接加载的资源），白色块代表被预制件引用的资源。

这种情况，两个资源同时引用另一个资源，有哪些打Bundle的方案呢？我们使用绿色的块将同一个bundle中的文件包裹起来，以展示几种不同的分包方案：

打进同一个bundle


<div style="display: flex; justify-content: space-between; align-items: flex-end; gap: 16px;">
    <div style="text-align: center;">
        <img src="/assets/whiteboard_exported_image_1.png" alt="打进同一个bundle" style="max-width: 220px; height: auto;">
        <div style="margin-top: 8px; font-size: 1em; height: 28px; display: flex; align-items: center; justify-content: center;">打进同一个bundle</div>
    </div>
    <div style="text-align: center;">
        <img src="/assets/whiteboard_exported_image_3.png" alt="打进两个不同bundle" style="max-width: 220px; height: auto;">
        <div style="margin-top: 8px; font-size: 1em; height: 28px; display: flex; align-items: center; justify-content: center;">打进两个不同bundle</div>
    </div>
    <div style="text-align: center;">
        <img src="/assets/whiteboard_exported_image_2.png" alt="打进三个不同bundle" style="max-width: 220px; height: auto;">
        <div style="margin-top: 8px; font-size: 1em; height: 28px; display: flex; align-items: center; justify-content: center;">打进三个不同bundle</div>
    </div>
</div>
当然还有更多不太可取的打包方案，这里不再枚举。

在不同场景下，不同的打包方案可能各有优劣。

-   方案1：在A，B总是同时被加载和卸载时，只看资源分配的话，这是最优的方案（但未必是合理的方案）。内存占用和Bundle数量、大小都是该需求下的最优解。但是这种情况只是少数，而且无法保证A和B是不是总是同时加载和卸载。由于A和B耦合在一起，如果游戏中A先于B卸载了，那么必须在B也卸载后，这个Bundle才能被彻底卸载；如果只想加载A，这种情况也会无法避免地将一部分多余的B的表单信息加载进来。

-   方案2：在A和B不保证同时出现时，是一种可以接受的方案。对内存是最优解，但是出现了冗余资源。role.png同时被包含在两个Bundle中，资源占用的硬盘空间变大了。

-   方案3：不论A和B是否同时出现，该方案在内存和资源占用上都是最优解，但Bundle数量是最多的。Bundle的信息本身也会占用一定的容量，也会反映在内存和硬盘的占用上。同时，Bundle数量越多，加载的IO消耗也越高。所以我们在考虑内存、硬盘大小的同时，还要尽量让Bundle数量更少。但总的来说，考虑到更普遍、更常见的状况，方案3是综合最佳的。方案3兼顾内存和硬盘的容量，在满足一些条件的情况下还可以通过程序自动生成这样的分包方案。
不过，本节的例子还是太简单了一些。

# 一个稍复杂的冗余

如果我们的角色持有的引用并非图片，而是Spine动画呢？

![whiteboard_exported_image (4)](/assets/whiteboard_exported_image_4.png)

Spine动画通常至少包含六个文件，这种情况下，比较合理的打包是什么样的呢？

![whiteboard_exported_image (5)](/assets/whiteboard_exported_image_5.png)

如果只有A.prefab引用了这个Spine动画，那么把Spine动画的文件和预制件A打包在同一个Bundle里就是最优解。这种情况下，加载一次Bundle就可以将预制件A所需要的依赖全部加载进来。

那么如果同时有两个预制件引用了同一个Spine动画呢？

![whiteboard_exported_image (6)](/assets/whiteboard_exported_image_6.png)

预制件A和B同时引用这个Spine动画。可以看出，Spine的六个文件中，有五个都是由Spine的主配置文件XXX_SkeletonData.asset间接持有引用，不被预制件直接引用。多数（但不是全部）情况下，我们不会单独加载或引用其中的一个文件，或绕过XXX_SkeletonData.asset访问后面的文件。

![whiteboard_exported_image (7)](/assets/whiteboard_exported_image_7.png)

参考之前的案例，我们可以得出上面的分包方案。将Spine动画的六个文件单独分包，A和B也单独分包。

如果再加入一个不同的角色，那么更好的分包方案应该是这样的：

![whiteboard_exported_image (8)](/assets/whiteboard_exported_image_8.png)

由于role2只被C.prefab引用，它应该和C.prefab打入同一个Bundle。

一个优秀的打包方案应该满足以下条件：

-   零冗余，对硬盘友好
    
    -   不要同一个资源复制几份。
        
    -   Unity内置资源需要单独处理。
        
    -   使用SBP。Built-in Pipeline有Bug，打包对Texture2D的引用时必然出现冗余。
        
-   低耦合，对内存友好
    
    -   加载A的时候，不要把不需要的B相关信息也加载进来。
        
    -   释放Bundle的时候资源不会相互掣肘。
        
-   Bundle数量要少，对IO性能友好
    
    -   加载一个需要的资源时，能尽量少地加载新的Bundle。
        
    -   Bundle数量越多，内存占用越高，也更容易导致加载一个资源需要一口气加载一大堆Bundle的情况。

# 常见的零冗余打包方案
## 反向引用计数法
我们将一个资源**被哪些资源引用**的列表称之为这个资源的反向引用列表。

![whiteboard_exported_image](/assets/whiteboard_exported_image.png)

如上图，role.png的反向引用为

```Plain
[A.prefab, B.prefab]
```

几乎所有打包方案都会获取资源的反向引用：

```C#
/// <summary>
/// 获取全量反向引用
/// </summary>
/// <param name="assetSet">全量资源的信息集合</param>
/// <returns></returns>
public static void GenerateInvertReference(Dictionary<string, CompactFileBundleInfo> assetSet)
{
    foreach (var asset in assetSet.Keys)
    {
        string[] dependencies = AssetDatabase.GetDependencies(asset， true/false);
        foreach (var dependencyPath in dependencies)
        {
            // 不需要的文件不处理，例如.cs文件通常不算进去
            if (!IsNeededAsset(dependencyPath)) continue;
            // GetDependencies返回的列表里包含资源本身，需要过滤
            if (dependencyPath == asset) continue;
 
            // 记录反向引用，只记录这个资源被哪些资源引用了
            assetSet[dependencyPath].InvertReferences.Add(asset);
        }
    }
}
```

反向引用计数法的基本原理是，计算出资源的反向引用数（不需要记录具体被哪些资源引用，只需要记录反向引用的数量），将反向引用数大于1的资源各自单独打包。

![whiteboard_exported_image (10)](/assets/whiteboard_exported_image_10.png)

如上图，由于role.png的反向引用数为2，大于1，所以role.png将会被单独打包。

这种打包方案可以保证零冗余，但是也有一些别的问题：

![whiteboard_exported_image (11)](/assets/whiteboard_exported_image_11.png)

如上图，当role1.png，role2.png，role3.png都被A和B引用时，按照上述方案，这三张贴图会被各自打进一个bundle里。最终的bundle数为5。这并不是很好的方案，只为了加载一个预制件A，需要加载4个bundle。实际上很容易看出，这三张贴图打进同一个bundle里会更好。

## 自定义规则法

在面对Bundle数过多问题时，追加特殊规则来约束Bundle的数量是一个很常见的方法，

-   例如可以设定一个这样的规则，当打包面对上面的问题时，如果role1.png，role2.png，role3.png在同一个文件夹，那么将它们打入同一个bundle。也就是根据资源所在的文件夹将次级引用的资源整合打包。
    
-   者根据项目情况，约定某些文件夹内的资源必定按照文件夹整合打包，将特定目录及子目录中的资源根据文件夹进行分包。
    
-   或约定某些类型的文件会被打入同一个Bundle。例如将lua文件打入同一个bundle，将shader打入同一个bundle，都是非常常见的做法。
    
-   还有一些根据项目情况处理的更复杂的自定义规则，例如将Spine相关的文件全部打入同一个Bundle，通过规则规避上面提到的spine引用问题。
    

自定义规则法可以**配合其他的零冗余打包算法一起使用**，该方法往往需要对打包出来的资源进行人工分析后，由于该方法根据具体项目具体分析，具体规则往往五花八门。

值得注意的是，如果不谨慎考虑规则的合理性和实现方法的话，自定义规则可能会引入一些额外的问题。例如笔者就见过一些自定义规则导致了打出的bundle之间有环引用，进一步导致引用关系混乱的问题。

<mark> Bundle之间总是不应该出现环引用，因为这说明这几个bundle总是需要一起加载，那么它们其实应该打成同一个bundle才对。如果出现了环引用，那么需要考虑一下分包规则是不是有什么问题。<mark>

## 反向查找法

首先计算并储存每个资源的**直接反向引用**。
### 直接反向引用是什么？

-   上面计算反向引用的算法中使用AssetDatabase.GetDependencies(asset)递归获取了资源的所有引用，然后让这些引用记录自己为**反向引用**。使用AssetDatabase.GetDependencies(asset, false)非递归地进行计算的话，得到的就是**直接反向引用**。
    
-   和反向引用计数法不同，反向查找法需要使用**直接反向引用**，具体原因将在下文讲解。

算法通过以下规则进行递归查找：

每个资源查找自己的**直接****反向引用****的Bundle名**

1.  如果没有反向引用，使用自己的路径作为Bundle名。
    
2.  当该资源所有的**直接反向引用**Bundle名相同时，自己也使用该Bundle名；否则使用自己的路径作为Bundle名。
    

最后可得**每个资源应当被分配到哪个Bundle**的信息。Bundle名可以在最后替换成哈希值。

**理想情况下**，这些文件的反向引用看起来是这样的：

![whiteboard_exported_image (12)](/assets/whiteboard_exported_image_12.png)

图中虚线代表反向引用。

<mark>注意，这里讲解的是理想情况下，实际情况下并非如此。我们将在之后讨论实际的情况。<mark>

下面提到的包名不是最终的包名，只是一个例子。实际上可以将前面的点号替换为\_之类的符号，最终换成哈希值。

-   A和B两个预制件没有反向引用，根据规则1，直接使用自己的路径作为Bundle名【A.prefab.bundle, B.prefab.bundle】；
    
-   role\_SkeletonData.asset有两个直接反向引用，也就是A和B。由于A和B的Bundle名不同，根据规则2，使用自己的路径作为Bundle名【role\_SkeletonData.asset.bundle】
    
-   role.skel.bytes和role\_Atlas.asset的直接反向引用是role\_SkeletonData.asset，所以它们所有的直接反向引用bundle名都是【role\_SkeletonData.asset.bundle】，于是根据规则2它们也使用这个bundle名。
    
-   role.atlas.txt和role\_Material.mat的直接反向引用是role\_Atlas.asset，由于role\_Atlas.asset的bundle名是【role\_SkeletonData.asset.bundle】，所以这两个文件的所有的直接反向引用bundle名都是【role\_SkeletonData.asset.bundle】，于是根据规则2它们也使用这个bundle名。
    
-   role.png的直接反向引用是role\_Material.mat，所以它的所有直接反向引用bundle名都是【role\_SkeletonData.asset.bundle】，于是根据规则2它也使用这个bundle名。
    

最终分包会得到三个Bundle：

![whiteboard_exported_image (13)](/assets/whiteboard_exported_image_13.png)

算法如下：

```C#
/// <summary>
/// 计算资源所在Bundle的名字
/// </summary>
/// <param name="asset">计算这个资源所在Bundle的名字</param>
/// <param name="allAssetsDict">所有资源的记录，其中包含资源的直接反向引用</param>
public static string GetAssetBundleName(CompactFileBundleInfo asset, Dictionary<string, CompactFileBundleInfo> allAssetsDict)
{
    if (asset.BundleName != null)
    {
        return asset.BundleName;
    }
    // 先根据文件名生成一次Bundle名防止无限循环
    asset.BundleName = asset.FullPath;
    string tempBundleName = null;
    foreach (var iref in asset.InvertReferences)
    {
        var refAsset = allAssetsDict[iref];
        var refBoundleName = GetAssetBundleName(refAsset, allAssetsDict);
        if (tempBundleName == null)
        {
            tempBundleName = refBoundleName;
        }
        if (tempBundleName != refBoundleName)
        {
            asset.BundleName = asset.FullPath;
            return asset.BundleName;
        }
    }
    if (tempBundleName != null)
    {
        asset.BundleName = tempBundleName;
    }
    return asset.BundleName;
}
```

### 现实总是不够理想的情况……

刚才提到的理想情况下的反向引用是这样的：

![whiteboard_exported_image (12)](/assets/whiteboard_exported_image_12.png)

但实际情况是这样的：

![whiteboard_exported_image (14)](/assets/whiteboard_exported_image_14.png)

预制件实际上是直接持有材质的引用的。所以该打包方案并不会如预期一般打出三个bundle，而会将材质和贴图单独打包，打出4个bundle。

### 还有一种情况

![whiteboard_exported_image (11)](/assets/whiteboard_exported_image_11.png)
对于传统的反向查找法，面对刚才提到过的这种情况，三张贴图也会被分别打进3个Bundle里。

## 反向查找法的增强
对于上面的问题，可以通过完善反向查找法的逻辑来解决。这种方法在继承直接反向引用的包名时，保留了更多的信息。

```Plain
/// <summary>
/// 计算资源的递归反向引用集
/// </summary>
/// <param name="asset">计算这个资源的递归反向引用集</param>
/// <param name="allAssetsDict">所有资源的记录，其中包含资源的直接反向引用</param>
public static SortedSet<string> GetAssetBundleName(CompactFileBundleInfo asset, Dictionary<string, CompactFileBundleInfo> allAssetsDict)
{
    if (asset.BundleNameDict.Count > 0)
    {
        return asset.BundleNameDict;
    }
    // 先生成一次防止无限循环
    asset.BundleNameDict.Add(asset.RefreshDirectAssetsBundleName());
    foreach (var iref in asset.InvertReferences)
    {
        var refAsset = allAssetsDict[iref];
        var refBoundleName = GetAssetBundleName(refAsset, allAssetsDict);
 
        if(asset.BundleNameDict.Count == 1 && asset.BundleNameDict.Contains(asset.BundleName)) 
        {
            asset.BundleNameDict.Clear();
        }
        foreach (var i in refBoundleName)
        {
            asset.BundleNameDict.Add(i);
        }
    }
    return asset.BundleNameDict;
}
```

通过继承所有的直接反向引用，来保留所有的引用信息。

```Plain
foreach (var asset in allAssetsDict.Values)
{
    var hashSet = GetAssetBundleName(asset, allAssetsDict);
    // SortedSet<string>遍历时，字符串是排序后的
    asset.BundleName = string.Join("&", hashSet);
}
```

最后，将直接反向引用名集合**排序后连接起来**作为Bundle名。

![whiteboard_exported_image (15)](/assets/whiteboard_exported_image_15.png)

由于保留了更多反向引用信息，算法可以得知：虽然两个直接反向引用被打入了不同的Bundle，但role1.png、role2.png、role3.png的直接反向引用是相同的，所以也会被打入同一个Bundle。遂解决了上面提到的情况。

但即使是加强的反向查找法，也无法解决Spine打包中提到的问题。


![whiteboard_exported_image (16)](/assets/whiteboard_exported_image_16.png)

材质文件的直接反向引用列表为\[A.prefab, B.prefab, role\_Atlas.asset\]，依旧无法与其他Spine文件打入同一个Bundle。

面对这种情况，当然可以辅以自定义规则法特殊处理Spine动画文件。但是，就没有一种更加完善的打包方案，能在不进行特殊处理的情况下解决这个问题吗？

## 资源的分级：一级资源和依赖资源

想要使用更完善的资源打包方案，必须有更完善的资源管理方式。这里，我们定义一级资源和依赖资源的概念：

凡是通过代码显式加载的资源都属于**一级资源**。这些资源往往是有**明确的使用途径**的。

其他**被一级资源引用**的资源也会被打入Bundle里，这些资源被称为**依赖资源**。
### 一级资源通常有哪些？

-   各种预制件
    
    -   UI界面预制件
        
    -   UI组件
        
    -   角色/敌人/物体
        
    -   地图元素
        
-   动态加载的文本/序列化文件
    
    -   代码文件
        
    -   自定义序列化文件
        
        -   json/xml/bytes/...
            
    -   自定义配置文件
        
-   动态加载的贴图
    
    -   头像/头像框/装饰
        
    -   技能图标/Item图标
        
    -   角色头像/立绘/半身绘
        
    -   动态背景图/动态UI元素
        
    -   图集
        
    -   ……
        
-   动态加载的材质
    
-   动态加载的网格/动画数据
    
-   **任何可能需要直接加载的资源**
    

管理一级资源时，应当对资源进行**人工分类**，放入相应的、**有明确管理标准**的文件夹。任何一个一级资源的文件夹都应该能明确其中的资源是什么用途的，**命名时也应该能结合内容一眼看出这些资源的用途**。
如：

-   存放头像的文件夹
    
-   存放立绘的文件夹
    
-   存放技能图标的文件夹
    
-   存放UI页面的文件夹
    
-   存放浮窗的文件夹
    
-   存放角色预制件的文件夹
    
-   ……

![](https://rcnly4ddjvpc.feishu.cn/space/api/box/stream/download/asynccode/?code=ODIxYTI2NzkzMDVkZTFiZTk4MDRlMDkxNWI5NmEzYTdfdTRDQVNzZjBrZzFmSVFyMGxpM0RHR0o3RWh3d0Q0Yk5fVG9rZW46RHptOWJkYmM1b3hSSTd4cW1FdmNtQmhobjRlXzE3NzA4MDQ5OTM6MTc3MDgwODU5M19WNA)

> 图中展示了一种一级资源管理方式，按照业务逻辑进行分类，按照$符号文件夹名管理。每个业务逻辑可以拥有自己的资源文件夹，其中存放了该业务逻辑使用的UI界面、浮窗预制件和lua代码文件。如果需要的话，还可以有其他的文件。UI预制件所引用的**贴图、动画、材质或其它资源**不会放在这里，而是放在美术/策划用于堆放素材的另一个不被$符号标记的文件夹。这些资源将在打包时被自动囊括进来，并按照规则分入适当的Bundle。

不好分类的页面/浮窗可以统一放在一个文件夹里。

也可以选择将同类的资源也可以统一放在同一个文件夹中，如将所有UI预制件放在同一个文件夹里，将代码文件放在另一个文件夹。虽然对于分包算法来说不会有太大区别，但这会让开发人员在开发和维护业务逻辑时，在寻找想要的文件这件事上浪费一些时间。

部分特殊的一级资源可以用**自定义规则法**单独处理，如lua或shader文件可以在一个统一的流程中进行搜集，在搜集其他的资源时忽略。或将角色与其技能特效的大量预制件放入特定命名的文件夹内，使其打入同一个bundle（因为这些预制件往往都是同时加载和卸载的）。

通过一级资源向下递归检索来获取游戏中需要使用的所有资源。凡不是一级资源的，都是依赖资源，应该放在一级资源文件夹之外的地方，避免扰乱检索。可以新建相应的文件夹进行管理。

依赖资源的文件管理不需要那么严格，只要不会干扰一级资源，打包系统不在乎它在哪里。但为了开发中的方便，依赖资源也应该按照一定规则进行存放。依赖资源的存放规则以美术/策划人员方便为准。

# 引用交集法

有了合理的资源管理方案，就可以有更完善的资源打包方案。

接下来介绍引用交集法。依旧以该分包图为例：

![whiteboard_exported_image (8)](/assets/whiteboard_exported_image_8.png)

```Plain
[MenuItem("Tools/BuildBundleCompact")] 
public static void Build()
{
    CompactBundleBuilderTools.ClearBuildPath();
    AssetDatabase.Refresh();
 
    // 生成所有要打入bundle的资源，以及这个资源需要打入哪个Bundle的信息
    var allAssetsDict = GetCompactFileBundleInfoDict();
    // 生成每个bundle的信息
    var bundleInfoDict = GetAllAssetBundlesInfo(allAssetsDict);
    // 保存log
    CompactBundleBuilderTools.SaveBundleInfoToJson(bundleInfoDict);
    
    Debug.Log($"Start Build AssetBundles. AssetBundles Count : {bundleInfoDict.Count}, Build Target : {BundleBuildTarget}");
    BuildAssetbundles(bundleInfoDict);
    Debug.Log($"Build Success! AssetBundles build to path : {BundlePath}");
}
```

和其他分包算法类似，该算法也需要获取所有需要打入Bundle的资源，并计算出它们应当被分配到哪个Bundle里。

```Plain
public static Dictionary<string, CompactFileBundleInfo> GetCompactFileBundleInfoDict()
{
    // 获取一级资源
    var directAssetDict = CompactBundleBuilderTools.GetAllDirectAssetsInfoDict();
    foreach (var asset in directAssetDict.Values)
    {
        // 每个一级资源生成自己的BundleName
        asset.RefreshDirectAssetsBundleName();
    }
 
    // 获取依赖资源
    var dependenciesAssets = GetDependenciesInfo(directAssetDict);
    foreach (var asset in dependenciesAssets.Values)
    {
        // 每个依赖资源根据自己被引用的情况，决定自己应该在哪个bundle里
        asset.RefreshBundleNameByInvertDirectAssetReferences();
    }
 
    // 将一级资源和资源资源合并处理
    // 因为依赖资源的内容通常比较多，所以把一级资源归入依赖资源的容器处理
    Dictionary<string, CompactFileBundleInfo> allAssetsDict = dependenciesAssets;
    foreach (var directAssets in directAssetDict)
    {
        allAssetsDict.Add(directAssets.Key, directAssets.Value);
    }
 
    return allAssetsDict;
}
```

首先根据目录规则获取所有一级资源，如获取某个路径下的所有资源，或获取所有$符号开头的文件夹内的资源。然后给一级资源分配Bundle。

**下文假设所有一级资源单独打包，即每个一级资源都会单独打进一个Bundle。当然，即使部分文件夹下的一级资源需要合并打包，也是容易处理的，算法的基本思想一样，这里不再赘述。**

分配Bundle时，首先使用一级资源的名字作为Bundle名（之后可以使用哈希值，或者其他命名分配方式修改包名，此处不再赘述）。

为描述简洁，这里不使用资源的全路径名，而是只使用资源名。实际处理的时候，**如果不能保证文件不重名**，需要使用全路径名。如果能够保证文件不重名，则可以使用资源名（带后缀）。

|     |     |     |
| --- | --- | --- |
| 文件路径 | 一级引用信息 | Bundle名 |
| A.prefab | 无   | A\_prefab.ab |
| B.prefab | 无   | B\_prefab.ab |
| C.prefab | 无   | C\_prefab.ab |

这样，所有一级资源应该打入的Bundle就计算完毕了。之后通过查找引用，找到所有二级资源。

|     |     |     |
| --- | --- | --- |
| 文件路径 | 一级引用信息 | Bundle名 |
| role\_SkeletonData.asset |     |     |
| role.skel.bytes |     |     |
| role\_Atlas.asset |     |     |
| role.atlas.txt |     |     |
| role\_Material.mat |     |     |
| role.png |     |     |
| role2\_SkeletonData.asset |     |     |
| role2.skel.bytes |     |     |
| role2\_Atlas.asset |     |     |
| role2.atlas.txt |     |     |
| role2\_Material.mat |     |     |
| role2.png |     |     |

接下来需要为这些文件分配相应的Bundle名。首先生成每个文件的一级反向引用信息。即生成反向引用的时候，只关注哪些一级资源引用了自己，不关注其他资源的引用情况。

|     |     |     |
| --- | --- | --- |
| 文件路径 | 一级引用信息 | Bundle名 |
| role\_SkeletonData.asset | A.prefab, B.prefab |     |
| role.skel.bytes | A.prefab, B.prefab |     |
| role\_Atlas.asset | A.prefab, B.prefab |     |
| role.atlas.txt | A.prefab, B.prefab |     |
| role\_Material.mat | A.prefab, B.prefab |     |
| role.png | A.prefab, B.prefab |     |
| role2\_SkeletonData.asset | C.prefab |     |
| role2.skel.bytes | C.prefab |     |
| role2\_Atlas.asset | C.prefab |     |
| role2.atlas.txt | C.prefab |     |
| role2\_Material.mat | C.prefab |     |
| role2.png | C.prefab |     |

之后，根据不同的一级引用信息，为文件生成Bundle名。

|     |     |     |
| --- | --- | --- |
| 文件路径 | 一级引用信息 | Bundle名 |
| role\_SkeletonData.asset | A.prefab, B.prefab | A\_prefab\_\_B\_prefab.ab |
| role.skel.bytes | A.prefab, B.prefab | A\_prefab\_\_B\_prefab.ab |
| role\_Atlas.asset | A.prefab, B.prefab | A\_prefab\_\_B\_prefab.ab |
| role.atlas.txt | A.prefab, B.prefab | A\_prefab\_\_B\_prefab.ab |
| role\_Material.mat | A.prefab, B.prefab | A\_prefab\_\_B\_prefab.ab |
| role.png | A.prefab, B.prefab | A\_prefab\_\_B\_prefab.ab |
| role2\_SkeletonData.asset | C.prefab | C\_prefab.ab |
| role2.skel.bytes | C.prefab | C\_prefab.ab |
| role2\_Atlas.asset | C.prefab | C\_prefab.ab |
| role2.atlas.txt | C.prefab | C\_prefab.ab |
| role2\_Material.mat | C.prefab | C\_prefab.ab |
| role2.png | C.prefab | C\_prefab.ab |

通过简单的字符串拼接，就可以将资源正确分配到所在的Bundle。可以注意到，role2的Spine动画由于只被C.prefab引用，按照目前的命名规则，会和C.prefab放到同一个Bundle里。而role的Spine动画由于被两个一级引用持有，会创建一个名为A\_prefab\_\_B\_prefab.ab的新资源包。这样，程序就生成了本节开头的分包图。

将一级引用和二级引用的表结合起来，就得到了最终的分包策略表：

|     |     |     |
| --- | --- | --- |
| 文件路径 | 一级引用信息 | Bundle名 |
| A.prefab | 无   | A\_prefab.ab |
| B.prefab | 无   | B\_prefab.ab |
| C.prefab | 无   | C\_prefab.ab |
| role\_SkeletonData.asset | A.prefab, B.prefab | A\_prefab\_\_B\_prefab.ab |
| role.skel.bytes | A.prefab, B.prefab | A\_prefab\_\_B\_prefab.ab |
| role\_Atlas.asset | A.prefab, B.prefab | A\_prefab\_\_B\_prefab.ab |
| role.atlas.txt | A.prefab, B.prefab | A\_prefab\_\_B\_prefab.ab |
| role\_Material.mat | A.prefab, B.prefab | A\_prefab\_\_B\_prefab.ab |
| role.png | A.prefab, B.prefab | A\_prefab\_\_B\_prefab.ab |
| role2\_SkeletonData.asset | C.prefab | C\_prefab.ab |
| role2.skel.bytes | C.prefab | C\_prefab.ab |
| role2\_Atlas.asset | C.prefab | C\_prefab.ab |
| role2.atlas.txt | C.prefab | C\_prefab.ab |
| role2\_Material.mat | C.prefab | C\_prefab.ab |
| role2.png | C.prefab | C\_prefab.ab |

将Bundle置于表头：

|     |     |
| --- | --- |
| Bundle名 | 包含文件 |
| A\_prefab.ab | A.prefab |
| B\_prefab.ab | B.prefab |
| A\_prefab\_\_B\_prefab.ab | role\_SkeletonData.asset |
|     | role.skel.bytes |
|     | role\_Atlas.asset |
|     | role.atlas.txt |
|     | role\_Material.mat |
|     | role.png |
| C\_prefab.ab | C.prefab |
|     | role2\_SkeletonData.asset |
|     | role2.skel.bytes |
|     | role2\_Atlas.asset |
|     | role2.atlas.txt |
|     | role2\_Material.mat |
|     | role2.png |

按照该表打包即可。
该算法可以基于反向查找法进行优化来实现，或是直接按照上面描述的顺序来实现，这里就不提供具体代码了。
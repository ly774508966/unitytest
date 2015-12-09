# unitytest
test code for unity，实际使用中只用异步io。


## 目录区别

* Resources
    在apk文件中，必须通过Resource.LoadAsync访问
* streamingAssetsPath
    在apk文件中，通过WWW访问
* persistentDataPath
   在文件系统中，通过WWW访问
   AssetBundle.CreateFromFile不行，没有异步版本，还必须是uncompress


## Object分类

1. GameObject，Component，各种Settings，
部分属于scene里的对象，被Object.FindObjectsOfType管理，
部分是资源Asset对象

2. AssetBundle，Mesh，Texture，Shader等资源Asset对象

我们叫场景对象和资源对象吧，可以猜测有2个对象管理者。


## 运行时对象查看

1. Object.FindObjectsOfType 来找到各种加载的active的Object，但不包含assets
2. Resources.FindObjectsOfTypeAll 来找到各种加载的资源。不管active还是deactive，包括asset

通过Resources.FindObjectsOfTypeAll 可查看资源是否被卸载


## 资源加载

### WWW
* AssetBundle.LoadAssetAsync 这个可能能用AssetBundle.LoadAsset，因为没io，只uncompress，cpu消耗型，但保险起见用async版本，可充分利用多线程吧（TODO测试）
* 要不要起多个WWW，来加载资源？，不知道ssd的情况会如何，（TODO测试）

### AssetBundle.LoadAssetAsync
* 连续调多次www, AssetBundle.LoadAsset，会加载多份asset！！！也就是说从不同bundle实例（但同一个bundle文件）中load出来的asset, unity认为不同。
* 如果这个prefab的bundle依赖texture的bundle，所以要先加载texture bundle，然后再加载这个，保存asset，然后把这两个bundle Unload，用asset来Instantiate是ok的。
* 也就是说unity在prefabBundle.LoadAsset(prefab)的时候会从已加载的assetBundle中寻找依赖然后把依赖的textureBundle的asset给加载出来。
* 所以只要保存这个asset就行了。

### Resources.LoadAsync
* 这个会直接解决依赖的。

### 同步方式
* 上面都是Async的方式，如果是inEditor则可以直接用AssetDatabase.LoadAssetAtPath 来同步加载。

### 资源到场景
* 以上这些load如果load上来一个prefab后，这个go并不再scene中，
（这个go并不是只读的，如果修改go，unity竟然会自动保存到prefab里！！！）

* 通过Object.Instantiate 来克隆一个， 这个go进入scene。这时候返回的就是场景对象了。
* 如果Object.Instantiate(texture)，会提示错误，但克隆会成功，得Object.Destroy


## 资源清理

load上来的asset，怎么清理呢？

### Resources.UnloadAsset

* 如果是Texture这些没问题，资源会被unload。注意即使是已经load了GameObject，go里依赖这个texture，那么
调用UnloadAsset（texture）后，go里也没贴图了。很悲惨，不会自动再加载的。

* 如果是GameObject，Component，AssetBundle会报错，

    UnloadAsset may only be used on individual assets and can not be used on GameObject's / Components or AssetBundles


### AssetBundle.Unload

* AssetBundle应该可以用AssetBundle.Unload(false)来清理。
* 如果调用2次www来取www.assetBundle，而不调用AssetBundle.Unload(false)，则第二次会报错，同时返回www.assetBundle为null，提示错误为：

    The AssetBundle 'xxx' can't be loaded because another AssetBundle with the same files are already loaded",

### Resources.UnloadUnusedAssets

GameObject，实验表明可以用 Resources.UnloadUnusedAssets() 然后GC，清除。

    private static IEnumerator Unload()
    {
        yield return Resources.UnloadUnusedAssets();
        GC.Collect();
    }

这个还真是没办法，看来只能这样了。


### 别用对asset使用Object.Destroy

* 如果调用Object.Destroy(asset) 会

    Destroying assets is not permitted to avoid data loss.
    If you really want to remove an asset use DestroyImmediate (theObject, true);

    就是说会删除资源哦。这个无论是scene模式还是game模式，都是这个结果。

* Object.Destroy 用于清理场景对象。



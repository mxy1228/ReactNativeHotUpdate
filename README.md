# 亲手搭建ReactNative热更新

目前网上已经存在一些ReactNative热更新平台，例如[CodePush](https://microsoft.github.io/code-push/)、[Pushy](http://update.reactnative.cn/home)等。这种将代码托管给第三方对于个人开发者或中小型公司来说还可以考虑，但成熟一点的公司还是喜欢自己搞一套，想怎么玩就怎么玩。
这篇文章将从以下几个方便总结我们是怎么“玩”ReactNative热更新的。
+ 为什么要做热更新？
+ 怎么进行热更新？
+ bundle太大怎么办？
+ bundle怎么管理？


###阅读这篇文章大概需要*分钟。
- - -
##为什么要做热更新？
开发ReactNative总喜欢和Hybrid进行对比，甚至不太懂ReactNative运行原理的同学就认为RN是Hybrid的升级版。其实不然，RN页面上的控件真的就是一个个Native控件，原理是RN的Virtual DOM将JS代码转译成Native真实控件，而不是传统Web中通过JS模仿出来的控件。这样RN给用户的体验更接近Native。_（虽然现在RN的性能还有缺陷）_
Hybrid有不需要发布新版APP就可以更新页面和功能，那么RN能做到这点么？
答案肯定是可以！因为就算不用RN，现在国内做的风生水起的插件机制就能做到。但插件技术有着各种缺陷，比如必须要用户重启APP插件代码才生效，在Manifest中注册的组件用插件更新比较蹩脚等。而RN就不一样了，RN的更新和插件比起来更加“轻量级”更加灵活。设置在用户停留在某个页面时，服务器下发push通知用户该页面有更新，新RN页面下载完成后就可以在本页面进行刷新，让用户看到新页面使用新功能。
笔者甚至在[专访包建强](http://www.infoq.com/cn/news/2016/04/baojianqiang-interview)的文章说看到这句话:
> “React Native的稳定，同时意味着插件化的落幕。这就是Android插件化的未来。”

至于RN是不是Android插件化未来我无法判定，但RN热更新的灵活性就决定了值得一试。（希望FB看到RN热更新的需求，最好能在后续的版本中开放这个功能）
- - -

##怎么进行热更新？
研究怎样做的思路和研究插件的思路一样。看系统加载的是什么？插件加载的是一个个类，所以从ClassLoader入手。RN加载的是一个个bundle，那么我们就从bundle入手。
插件机制的思路是替换class，那么是不是替换bundle文件就可以实现RN的热更新？
bundle是JSX代码通过RN服务器编译后的产物。通过阅读RN源码：
```java
public static JSBundleLoader createFileLoader(final Context context, final String fileName) {
        return new JSBundleLoader() {
            public void loadScript(ReactBridge bridge) {
                if(fileName.startsWith("assets://")) {
                    bridge.loadScriptFromAssets(context.getAssets(), fileName.replaceFirst("assets://", ""));
                } else {
                    bridge.loadScriptFromFile(fileName, "file://" + fileName);
                }

            }

            public String getSourceUrl() {
                return (fileName.startsWith("assets://")?"":"file://") + fileName;
            }
        };
    }
```
当打出的包是release模式时，RN会通过uri找bundle文件并加载。（debug模式RN会直接从配置的RN服务器拉取bundle文件）这中间没有进行编译等附加逻辑。所以理论上只需替换bundle文件即可。
#####_上述源码也暴露了RN支持两种bundle目录，一种是将bundle内置到assets中，一种是存储到本地存储空间中。_
那么问题来了，替换了bundle文件后，怎么通过RN重新加载呢？这个笔者没找到官方提供的接口。（再次呼吁官方能开放接口支持刷新）通过[使用React-Native实现app热更新的一次实践](https://github.com/fengjundev/React-Native-Remote-Update)找到用反射强制RN更新的方法。
```java
/**
         * 通过反射重新Load Bundle到ReactInstanceManager中
         *
         * @param bundleLocation
         */
        private void resetBundleReflected(String bundleLocation) {
            if (mRNManager == null
                    || TextUtils.isEmpty(bundleLocation)) {
                return;
            }
            Class<?> RCTManagerClazz = mRNManager.getClass();
            try {
                Method method = RCTManagerClazz.getDeclaredMethod("recreateReactContextInBackground"
                        , JavaScriptExecutor.Factory.class
                        , JSBundleLoader.class);
                method.setAccessible(true);
                method.invoke(mRNManager
                        , new JSCJavaScriptExecutor.Factory()
                        , JSBundleLoader.createFileLoader(mApplicationCtx, bundleLocation));
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
```
bundle文件的路径被包装成JSBundleLoader，然后作为参数之一给recreateReactContextInBackground调用。
> mRNManager.getClass()获取的实际是ReactInstanceManagerImpl，而不是ReactInstanceManager。通过类名就能这个这两个类的属性及关系。

至于recreateReactContextInBackground方法做了什么，感兴趣的同学可挪步到ReactInstanceManagerImpl中一探究竟。
- - -
##bundle太大怎么办？
通过以上结论，已经大致摸清RN热更新的思路。巴特！在实践中发现，一个普普通通的bundle少则大几百K，多能上M。这体积显然不能忍，否则乐了运营商坑了用户。那么有办法减小bundle的体积么？有！FB提供了‘--minify’来减小编译的bundle体积。但结果还是不太理想。在一筹莫展之际，不妨看看bundle文件是个什么鬼，说不定能找到突破点。
其实bundle的本质是一个大的文本文件。
![1.png](quiver-image-url/9B7F0C4D723CA9514861D4593D7061DB.jpg)
这就有办法了，可以针对文本文件进行差量更新！
笔者之前做某应用市场时做过APK增量更新，当时用的是开源的[bsdiff/patch](http://www.daemonology.net/bsdiff/)。它的原理是对文件进行二进制差异分析。理论上用这个库可以完成bundle文件的增量差异分析。但笔者谷歌到另一个专门对文本文件进行差异分析的库[google-diff-match-patch](https://code.google.com/p/google-diff-match-patch/)。而且它支持Java, JavaScript, Dart, C++, C#, Objective C, Lua and Python多个平台。比以so文件包装的bsdiff易用性好很多。而且使用这个库可以让bundle差异生成的patch包同时支持iOS和Android两个平台。
```java
private diff_match_patch mDiff = new diff_match_patch();
···
// 读取patch文件文本内容，diff_match_patch会将其解析成diff_match_patch.Patch列表。
LinkedList<diff_match_patch.Patch> patches = (LinkedList<diff_match_patch.Patch>) mDiff.patch_fromText(content);
// 读取老版本bundle文本内容（currentVersionContent），将diff_match_patch.Patch列表和其合并。
Object[] results = mDiff.patch_apply(patches, currentVersionContent);
// 合并完成后会生成一个length为2的Object数组。Object[0]是合并后的结果，是个String。Object[1]是个Boolean类型的数组，每一项标记着每个diff_match_patch.Patch的合并结果。true为成功，false为失败。只有当Object[1]的每一项都为true时才表明patch文件和老版本的bundle真正合并成功。最后将Object[0]的内容写到文件中即可。这个文件就是我们最终想拿到的新bundle。
```
我们把bundle抽象成下图。
![1.png](quiver-image-url/C09504AE7DBDAEF4F23A0CE6111659E1.jpg)
把bundle分解成上下两部分，上面是和RN自身逻辑有关的代码，下面才是真正业务逻辑相关代码。因为每个bundle中上半部分RN代码可以复用，所以理论上，只需替换下半部分的业务代码，即可生成一个新的可加载的bundle。这也就是增量更新的思路。
![2.png](quiver-image-url/3C7645AA126BC624E6BAF57EDDE5E6A0.jpg)
根据我们亲自测试，一次正常业务升级产生的patch包体积为200K~300K。乍一看还是有点大，不过zip一下只有50K左右。也就是一个bundle热更新一次只会消耗用户50k左右的流量。
热更新看似能走通了，但是还要解决另一个问题。上面有提到，一个bundle小则600k左右，大则上M。虽然可以在用户在打开RN页面时再下载bundle，但如果用户在非wifi网络环境下，这些下载流量还是得让用户心里默默跑来一群草泥马神兽的。RN支持加载内置bundle，但如果将bundle都内置到APP中对APP的体积又是个不小的影响。
那么为什么不将内置的bundle像热更新一样进行拆分呢？
发现一个业务bundle中，有很大一部分代码都和RN自身框架有关，那么理论上把只属于RN自身框架的代码摘出来作为common.bundle，剩下的业务代码打包成patch，这样内置N个资源由（N x bundle）变成了（common.bundle + N x patch）
![3.png](quiver-image-url/2084A7708BE9229E1EA26729F96647B0.jpg)
在APP安装第一次启动时再分别将业务patch包和common.bundle进行合并成业务bundle。
这种策略让我们内置六个bundle的zip文件由600k降为400k，压缩率提高33%。
- - -
##bundle怎么管理？
怎样能快速找到业务要加载的bundle文件？bundle的版本怎么管理？在common.bundle和patch都有更新的情况下怎么保证common.bundle的下载更新任务优先级最高？
这些问题还得从内置patch资源的结构说起。
下图为我们内置资源zip包的目录结构。
![4.png](quiver-image-url/8D002C0DB19EB5B1181B6E830E950CA4.jpg)
首先根据内置的bundle的id为命名建立文件夹，每个文件夹里再创建名称为‘patch’的文件夹放置业务对应的patch包。common.bundle和属性配置文件config.json在zip包最外层。config.json里的数据标记了common.bundle的版本号，及patch包对应的bundleId及版本号。
APP第一次打开时会将该zip包解压，读取config.json，根据里面记录的属性数据，将common.bundle和各个patch释放到磁盘存储。common.bundle最终会以“base_version.bundle”命名，patch包则以“version.patch”命名。patch和common.bundle合并的业务bundle以“bundleId_version.bundle”命名。
以内置bundleId为106及112为例，内置资源释放后的文件目录结构入下图。
![5.png](quiver-image-url/F55379CA249D9189AEFF32010BF0724A.jpg)
这样构建bundle目录的好处在于RN载体页只要知道要加载的bundleId，就能很快拼出该bundleId对应的目录路径，并拿到要加载的bundle。
这样构建还有个好处，可以根据patch包的文件名及common.bundle的文件名快速获得业务bundle及common.bundle的版本号。
释放内置资源流程如图。
![6.png](quiver-image-url/C6C460FA2265B3BBA957AB42A225B030.jpg)
当热更新下发新patch版本时，以上图为例，本地bundleId为106的bundle版本为31，而服务器通知Native当前最新版本为32，这时Native会将最新的patch包下载并保存到/106/patch/目录下，以32.patch命名。然后将32.patch和base_7.bundle进行合并生成106_32.bundle并替换106_31.bundle。这样RN载体页再次加载106时将加载版本为32的bundle。那么可能会有读者问到，patch文件夹下是否只保存最新版本的patch包？目前我们的策略是会保存所有下载的patch版本。为什么呢？因为服务器下发给Native的所有文件数据或文件都是不可靠的，所以Native要做得健壮些，增加容错机制。
还是以上面情景为例。当下发的32版本合并成功为106_32.bundle后，因为某RD粗心编写JS文件时隐藏了一个导致崩溃的BUG，导致RN载体页加载106_32.bundle失败，这时我们全局捕捉了RN抛出来的JavascriptException，当捕获这个异常后，会将32.patch删除，并将比32版本号小的patch（这里是31.patch）重新和base_7.bundle进行合并，重新生成106_31.bundle来保证RN载体页下次再加载106时可以正常加载。
热更新流程如图。
![7.png](quiver-image-url/787F4EC2F7952E5BB3453E12885B68B2.jpg)
在common.bundle和patch都有更新的情况下怎么保证common.bundle的下载更新任务优先级最高？这套热更新系统全部由RxJava编写，common.bundle更新任务交由BehaviorSubject执行，所有patch包的下载合并任务都由Subject通过concatMap连接，若有common.bundle更新时BehaviorSubject先行执行，没有的话BehaviorSubject会将现有的common.bundle路径发射给后面执行的Observable。
**说到这，用RxJava时一直有个困扰问题，RxJava要是遇到分支判断写？没找到哪个操作符支持if···else···似的分支判断逻辑。**
- - -
##踩过的坑
1、当升级到RN 0.28版本后，打混淆包后运行RN页面崩溃。参照官方Demo工程的proguard也无解，最终迫不得已将com.facebook.react包下的所有class全部keep才得以解决。
```java
-keep class com.facebook.react.** { *; }
```
2、当APP覆盖安装时需把之前释放的内置资源清空，因为新APP的内置资源有可能升级。
- - -
##总结
**安卓RN版本有风险，升级需谨慎！！**
- - -
参考文章：
+ [为什么说Virtual DOM 是React的精髓所在](http://www.howcode.cn/article/reactnative/3953630da28e5181cffca1278517e3cf.php)
+ [Android插件化：从入门到放弃](http://www.infoq.com/cn/articles/android-plug-ins-from-entry-to-give-up)
+ [使用React-Native实现app热更新的一次实践](https://github.com/fengjundev/React-Native-Remote-Update)

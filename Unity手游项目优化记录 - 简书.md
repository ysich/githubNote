# Unity手游项目优化记录 - 简书
Unity手游项目优化记录 - 简书

[](/)

[登录](/sign_in)[注册](/sign_up)[写文章](/writer)

[首页](/)![](https://www.jianshu.com/bn-static/assets/web/banner/ilicT7sdlj6HoP.svg)
[下载APP](/apps?utm_medium=desktop&utm_source=navbar-apps)[会员](/vips)[IT技术](/techareas)

Unity手游项目优化记录
=============

[![](https://upload.jianshu.io/users/upload_avatars/7875603/08ae0111-1e2e-4ac9-8cc8-bf0b6ecd9def.png?imageMogr2/auto-orient/strip|imageView2/1/w/80/h/80/format/webp)
crossous](/u/50094e11ed08)关注赞赏支持

Unity手游项目优化记录
=============

[![](https://upload.jianshu.io/users/upload_avatars/7875603/08ae0111-1e2e-4ac9-8cc8-bf0b6ecd9def.png?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)
](/u/50094e11ed08)

[crossous](/u/50094e11ed08)关注IP属地: 北京

22021.08.09 11:14:46字数 7,498阅读 7,324

前言
==

  游戏的优化能从各个角度入手，渲染、逻辑、内存、IO、GC、功耗等等，未经过优化的游戏往往在每个方面都会出现问题，需要分别找出问题并逐个解决。这篇文章将分享我在项目中遇到的问题、优化的方法、使用的工具等。

一、如何查找瓶颈
========

  优化渲染效率，首先是要找出约束当前效率的瓶颈，是CPU，还是GPU，还是IO。  
  首先要尽量开放帧率，假如游戏只能跑到43帧，那就把帧率开放到60，或者用4、5年前的低配机运行。游戏可能最终只需要跑到30帧，但更高的帧率更容易暴露问题。  
  CPU的工作时，准备好一帧的渲染数据和指令，交给渲染线程，等待GPU渲染完空出资源后，将指令和数据提交给GPU。GPU没空出资源时，渲染线程还需要等待GPU渲染完毕。  
  GPU需要CPU传递过来的数据才能工作，如果CPU没有及时将指令和数据传递到GPU，则GPU无法工作。  
  CPU与GPU异步执行，传递数据、指令时需要同步，出现瓶颈时，肯定有一方等待另一方，至于怎么查看，我们可以用Unity自带的Profiler看Wait信号量，然后去Google看看到底是谁等谁，问题出在CPU还是GPU。当然Unity的性能检查工具UPR也可以，在CPU-线性时序中查看。

  

![](https://upload-images.jianshu.io/upload_images/7875603-da9c7c453ba77cb8.png)

Gfx.WaitForPresentOnGfxThread显示CPU在等待GPU完成工作

  

![](https://upload-images.jianshu.io/upload_images/7875603-b50c5377ea64de65.png)

Profiler也能看到同样的等待信号

  找到了问题在哪里，就要着手优化。

二、渲染相关瓶颈
========

  优化时要确定一个优化手段能解决什么样的问题，最好是通过实验确定，不然程序、美术废了很大劲去改资源、逻辑，效率却没有上升，那就是白忙活。

### 1\. Batch

  Unity有三个Batch：Static Batch、Dynamic Batch、SRP Batch以及一个Instance。  
  搞清楚Batch前，要清楚什么是DrawCall，什么是SetPassCall，这一点可以用RenderDoc查看。  
  DrawCall就是一个API、一个指令，告诉GPU要画几个索引指定的顶点：

  

![](https://upload-images.jianshu.io/upload_images/7875603-55e2775d2bc6f523.png)

画108个索引指定的顶点，1代表1个

  

  而具体是哪些索引、哪些顶点，用什么Shader、Shader用的数据存在哪里，就需要其他API、指令来指定了，这就是SetPassCall。

  

![](https://upload-images.jianshu.io/upload_images/7875603-babe8b8dade2b607.png)

设置Shader、设置常量缓冲区、上传数据

  
  DrawCall就是一个指令而已，如果渲染1000个完全一样的物体，那就可以只用一个SetPassCall，然后设置1000个DrawCall给GPU（假定不用Instance），这些DrawCall全看作数据，其实也不是很大。  
  但如果每个物体的网格、Shader、常量缓冲数据都不同，一个DrawCall前都要加一堆SetPassCall，不但数据很大，GPU切换渲染状态时也会费力。  
  所以我们一般不优化DrawCall，而是优化SetPassCall。

##### 1.1 Static Batch

  顾名思义，静态Batch，把静态网咯提前合并成一个，多个DrawCall可以用一个DrawCall完成，SetPassCall自然也少了，但很少会用。  
  你网格A使用ShaderA，网格B使用ShaderB，Shader都不同，怎么用一个DrawCall一起渲？那Shader相同呢？网格A使用材质A，网格B使用材质B。那么这一个DrawCall到底是用哪个材质的纹理和数据？理论上能做到区分，但Unity显然不会“智能”到这种程度。  
  所以Static Batch只适用于不同网格(相同网格用Instance)，材质相同的静态物体。  
  理论上会合并DrawCall，但根据测试，在Unity2019中，一个不变物体加上两个可以切换LOD的物体，不变物体和其中之一组合可以一个DrawCall渲染，与另一个组合需要两个DrawCall，第二个DrawCall不需要SetPass。在Unity2021中，不变物体可以与两个LOD物体分别组合，只需要一个DrawCall即可渲染出来。

##### 1.2 Dynamic Batch

  Dynamic Batch会把你相机中使用相同材质的网格，实时合并并上传到GPU。  
  因为是合并DrawCall，所以缺陷和Static Batch的差不多，不可能合并不同材质。并且因为要实时上传网格数据，还对网格顶点大小有要求。

##### 1.3 Instance

  Instance是底层图形API提供的功能，同一网格，同一Shader，即可通过一次调用进行多个物体渲染，往往用来渲染草地。  
  Instance对材质不做要求，可以是使用同一Shader的不同材质。

##### 1.4 SRP Batch

  前面两个合并DrawCall的Batch有那么多限制，相比之下SRP Batch只合并SetPassCall。

###### 常量缓冲区

  首先明确常量缓冲区，是指一次渲染时，所有像素共用的数据。用renderdoc能看到当前阶段所用到的常量缓冲区：

![](https://upload-images.jianshu.io/upload_images/7875603-4161c5c638df57ec.png)

  
  常量缓冲区名字可自定义，但Unity SRP项目的常量缓冲区一般用**UnityPerDraw、UnityPerMaterial**和**$Globals**。  
  **UnityPerDraw**一般是**ObjectToWorld、WorldToObject**矩阵，每个物体的数据都不同，由管线源码定义、传输数据。  
  **UnityPerMaterial**用于SRPBatch，我们需要将**UnityPerMaterial**和Shader的**Properties**中定义的变量一一对应，才能进行SRPBatch，所以这个缓冲区由我们定义并编写。  
  没被任何常量缓冲区包含的变量则通通被包含进**$Globals**（Opengl是Uniform）。  
  当两个物体使用同一材质时，代表两者使用的Shader相同，两者材质数据相同，唯一不同的是PerDraw。所以当物体A切换到物体B时，只需要上传PerObject常量缓冲，设置PerObject常量缓冲，DrawCall即可。  
  不过每次渲染上传这一步也是可以省去的，使用Shader相同，代表常量缓冲区大小相同，完全可以在第一个物体时，将所有物体需要的PerObject以数组的方式串联到一起上传，设置常量缓冲区时，在地址后加一个偏移即可。  
  以这种想法考虑，即使物体使用的不是一个材质，只要是同一Shader的，即  
可进行SRP Batch，因为虽然材质数据不同，但PerMaterial常量缓冲区大小是一样的，一样是一起上传，在渲染时传递一个索引即可。  
  可以看一个栗子：

![](https://upload-images.jianshu.io/upload_images/7875603-c03a6837aadb65bb.png)

  场景中三个立方体共用一个Shader，但材质不同。  
  渲染第一个立方体时，调用了这些API：

![](https://upload-images.jianshu.io/upload_images/7875603-bcb5c7f26e4d0cc4.png)

  OMSetBlendState设置渲染状态，RSSetState设置光栅化状态，VSSetShader设置Shader，Map/Unmap将数据从CPU传递到GPU、VSSetConstantBuffers设置常量缓冲区等等等。  
  第二次DrawCall时，只调用如下API：

![](https://upload-images.jianshu.io/upload_images/7875603-117b3defa0e0ee52.png)

  
  对比三次DrawCall时VSSetConstantBuffersAPI：

![](https://upload-images.jianshu.io/upload_images/7875603-99a56807f3a58005.png)

可以发现是同一Buffer传递的不同偏移值和大小。  
  **注意，同一Shader还要是同一变体，不同变体Define可能导致常量缓冲区大小不同。**   
  SRP Batch需要Shader编写规范，是否能SRP Batch，以及不能SRP Batch的原因可以看Shader面板：

![](https://upload-images.jianshu.io/upload_images/7875603-eb9a0617ca3387d3.png)

  
  FrameDebug里能看到SRP Batch被打断的原因：

![](https://upload-images.jianshu.io/upload_images/7875603-1c26d22a09b6c6dd.png)

  
  原因有很多，甚至就算使用同一变体也有可能被打断，原因可能是排序or使用了不同Job准备渲染命令等等。所以建议场景中的变体数尽量少。  

* * *

  
  上面讲了一堆Batch，但当出现GPU瓶颈时，优化Batch未必有很大效果，原因是，Batch减少的是SetPassCall，也就是CPU记录渲染命令、传递数据的次数，以及GPU切换状态的次数。如果状态切换不是瓶颈，那么Batch并不能看到良好的优化效果。  
  当然，该Batch还是要Batch的，毕竟能减少一些CPU消耗，只是不要对为什么SetPassCall少了，帧率却没有增长之类的事情抱有疑惑。  
  如果要确定性能瓶颈是否出在Batch，可以写个按钮，运行时将所有物体的材质换成统一的一个，这样肯定都能Batch上（除了超过一定数量自动打断），看下帧率有没有提升即可。  

* * *

### 2\. 渲染效率

  实时渲染对渲染效率要求很高，如果游戏想要以60帧运行，那么留给每帧的时间只有16ms，如果以30帧运行，那么时间有33ms。  

  游戏一帧时长可以用Unity的Profiler看：

![](https://upload-images.jianshu.io/upload_images/7875603-f4f430725c2b2b16.png)

  
  上图是我们游戏在PC下一帧所用时间，WaitForTargetFPS是为了将帧率锁在60，一帧总体时间是16ms。  

  如果没有限定时间内CPU没有进行下一帧的逻辑，说明主线程没运行完毕，或被其他问题阻塞，例如IO、GPU。  
  如果是GPU的问题，就要想办法优化游戏渲染效率。

##### 2.1 剔除

  能不渲染的不渲染，就是剔除，游戏中剔除的方案多种多样，视锥体剔除、遮挡剔除、Hiz剔除，还有UE5的网格级别剔除，虽然花样多，但大多数移动游戏只能做到物体级别的剔除。  
  Unity自带了遮挡剔除，可以先试试。在打开Window->Rendering->Occlusion Culling打开遮挡剔除窗口，进入Bake标签

  

![](https://upload-images.jianshu.io/upload_images/7875603-4e0983b29a97c66e.png)

将场景中静态物体标记Occluder Static/Occludee Static

  

  点击Bake烘焙，处于Occlusion窗口时，Scene窗口右下角会出现遮挡剔除标签，点击Portals前面的小方块，能看到烘焙后的包围盒：

![](https://upload-images.jianshu.io/upload_images/7875603-5fe68882bf1a6367.png)

  
  然后勾选相机组件的Occlusion Culling即可。  
  遮挡剔除的效果需要测试，假如你视野内有非常多的物体，遮挡关系不明显（例如天空相机俯视角），那么未必会有提升，甚至有性能下降的可能。

##### 2.2 Overdraw

  很多时候，像素并不能最终呈现到屏幕上，但依旧需要走一遍片元着色器，这就是Overdraw。为了解决这个问题，有很多现有的解决方案。

###### 2.2.1 Early-Z

  标准管线是执行完片元着色器后再进行深度测试、深度比较，这样可能有些浪费，因为当前片元的深度已知，可以提前知道片元是否无效，会不会显示到最后的渲染目标纹理中，所以大多数硬件都支持Early-Z，即先进行深度测试，通过后再执行片元着色器，不需要任何操作，硬件能自动支持。  
  但有几种情况Early-Z是失效的。例如在片元着色器进行深度写入：

```
struct FragOutput
{
    float4 color : SV_Target;
    float  depth : SV_Depth;
}

FragOutput frag(Varying i)
{
    FragOutput o = (FragOutput)0;
    o.depth = 1;
} 
```

  或者进行AlphaTest，执行了Clip或discard语句。  
  我的理解是，深度测试和深度写入是在一起的，Early-Z已经进行了深度写入，在片元着色器进行深度写入或片元丢弃，都会导致深度不一致。  
  虽然有上述问题，但大多数的物体都是能正常走Early-Z的。

###### 2.2.2 排序

  Early-Z只能将当前片元深度和当前深度缓冲区比较，此时我们想一个最坏的可能——物体从远向近渲染。第一个物体渲染到RT上，第二个物体比第一个物体近，由于当前深度缓冲区只有第一个物体，Early-Z也无法知道未来有没有像素能覆盖当前物体的像素，所以第二个物体的全部像素也全部渲染到RT上了，以此类推，Early-Z完全没有生效。因此想要更好的利用Early-Z，需要让物体从近向远渲染，这样渲染当前物体的像素时才能知道更大概率的知道当前像素有没有被遮挡，有更好的剔除效果。

###### 2.2.3 Perpass-Z

  排序是基于物体的，如果物体本身存在遮挡关系，或物体间存在穿插关系，排序也未定好用。  
  所以一种想法是先渲染一遍所有物体的深度，然后再渲染物体，这样能更好的利用Early-Z。  
  缺点是顶点着色器的消耗直接翻倍，优点是，除了可以进行Early-Z外，部分效果可以依靠这张Perpass-Z buffer生成，例如屏幕空间阴影。

###### 2.2.4 HFS

  手机上GPU渲染架构和PC的GPU架构不同，手机的渲染基础单位是一个Tile而不是整个RT，并且没有显存，而是非常小的高速缓存On-chip Memory。执行完顶点着色器后，将数据传递到内存中，记录每个三角形影响的Tile。  
  渲染Tile时取得当前Tile中的顶点，光栅化、插值并渲染。  
  上述是TBR（Tile-Based Rendering）架构，个别芯片支持在像素渲染前进行可见性判断，就是TBDR（Tile-Based Deferred Rendering）架构，也是用来解决Overdraw问题，这一操作称为隐藏面削除HSR(HiddenSurfaceRemoval)。

###### 2.2.5注意事项

  上述的操作中，大多数不用手动实现，大多数Unity都已经处理好了，而是要我们知道后，注意Shader编写和代码实现，例如：  
  ①不要滥用AlphaTest材质，可能导致Early-Z、HSR失效，我们主界面有一个挺大的建筑，因为有植物，所以整体用了植物的Shader，而植物经常使用AlphaTest+双面渲染+顶点风场动画，导致建筑的渲染时钟数是同面积同光照模型其他物体的好几倍，这种情况建议把植物和建筑分离。  
  ②Perpass-Z在移动端性能未必好。因为部分手机有HSR。但反过来说，当HSR失效时，用Perpass-Z可能会有提升空间，例如大量草，可以先渲一遍Depth Only Pass，然后再渲染草本身，要注意的一点是，渲染草本身时，需要关闭Alpha Test，并且深度检测设置为Equal。  
  ③有HSR功能的手机不做物体深度排序，这个看URP的源码：

```
var commonOpaqueFlags = SortingCriteria.CommonOpaque;
var noFrontToBackOpaqueFlags = SortingCriteria.SortingLayer | SortingCriteria.RenderQueue |
                                           SortingCriteria.OptimizeStateChanges | SortingCriteria.CanvasOrder;
bool hasHSRGPU = SystemInfo.hasHiddenSurfaceRemovalOnGPU;
bool canSkipFrontToBackSorting = (baseCamera.opaqueSortMode == OpaqueSortMode.Default && hasHSRGPU) ||
        baseCamera.opaqueSortMode == OpaqueSortMode.NoDistanceSort;

cameraData.defaultOpaqueSortFlags =  canSkipFrontToBackSorting ? noFrontToBackOpaqueFlags : commonOpaqueFlags; 
```

  是用`SystemInfo.hasHiddenSurfaceRemovalOnGPU`这个Unity API检查的，我测试时，小米手机的Adreno GPU没有深度排序(支持HSR)，华为的Arm Mali GPU不支持HSR。  
  不过我查的资料显示是苹果、PowerVR芯片才支持HSR，有点迷。

##### 2.3 着色效率

  着色效率也和渲染时间有很大关系。  
  简单衡量着色效率，就看Shader行数，但这样并不直观，如果Shader翻译成dxbc，矩阵和向量mul能翻译成4行代码，pow能翻译成3行代码，而乘加只用一行代码。

  

![](https://upload-images.jianshu.io/upload_images/7875603-e09bf30adbc12603.png)

用renderdoc能查看dxbc

  

  dxbc行数也不能完全代表Shader的着色效率，因为dxbc指令的效率也不同，sincos和简单的加减不能直接比较。  
  更有代表性的是每个指令的时钟数。  
  另外一点影响着色效率的是物体的屏幕面积占比，一个物体占用屏幕面积越大，就需要更长的时间渲染  

  

![](https://upload-images.jianshu.io/upload_images/7875603-acd379c24a9290a6.png)

  
  RenderDoc点击小闹钟图标能看到一个DrawCall消耗时间的大致时长，但这只能比较物体之间渲染效率，并不能当作标准，因为渲染耗时由多部分组成，DrawCall之间也有并行部分。不同移动设备的架构也可能导致渲染效率的不同，建议根据GPU硬件厂商选择Profile工具，例如小米手机常用的AdrenoGPU，可以用SnapDragon抓取，查看物体渲染时钟数。  
  连接设备，然后新的快照捕捉：  

![](https://upload-images.jianshu.io/upload_images/7875603-10cd6f5d82d345f9.png)

  

  Launch Application，选择你应用的包名，Launch运行，选择要抓取的项，这里演示只选择Clocks

  

![](https://upload-images.jianshu.io/upload_images/7875603-e2ec20b11d3ea4d3.png)

  
  然后用Take Snapshot抓取快照，即可抓取。右侧可看到Clocks：  

![](https://upload-images.jianshu.io/upload_images/7875603-bf7979303cbd6389.png)

  举个我们项目的栗子，我们项目的狐妖尾巴毛是用20个Pass渲染的，当屏幕占比大时特别卡顿，甚至单个妖怪跑不到60帧（低端机），后来发现尾巴的光照模型用的是Charlie，dxbc比普通pbr多了一倍，更换后帧率显著上升。  
  另一个栗子是主场景的地板，普通PBR材质，但占地面积大，渲染时钟数特别高，将地板换成Blinn-Phong后，效果没有明显变化，但时钟数少了一个数量级。

### 2.4 LOD

  LevelOfDetail，老生常谈的优化方法，近距离模型，远距离低模，更远距离面片，有效降低场景面数，如果是玩家接近不了的地方，直接用面片代替即可。

### 2.5 带宽

   带宽衡量一定时间（每秒）不同硬件设备传递数据的多少。带宽高不一定会导致设备卡顿，但有可能导致设备发热严重，所以尽量减少带宽。

##### 2.5.1 频率、分辨率

  游戏中优化最有效果的，就是降低帧率、降低分辨率。降低分辨率能轻松提升帧率，宽高缩减一半，渲染压力就只有原来的1/4，宽高缩减四分之一，渲染压力就只有原来的1/16。降低帧率能显著降低带宽，减少发热。但这是没有办法的办法，对画质影响不小，那些拿手机降到400P跑PC 3A大作到处跟别人比，然后吹优化优劣，是没有意义的。  
  主RT降分辨率、帧率对效果影响较大，但其他方面可以视情况缩减一些。  
  比如半透特效、毛发、水面等半透物体，可以RT减半渲染，然后再Blend回ColorAttachment中；以及Cascading Shadowmap，前几级每帧更新，而渲染物体较多的后几级可以分帧更新。  
  这些缩减也不是单纯的缩减，降采样渲染混合回颜色RT时，可能需要对边缘做淡入淡出，否则会出现锯齿。分频渲染Cascading Shadowmap倒是不用额外做什么，因为采样高级别阴影图的像素离相机更远，意味着像素小、不清晰、变动慢，更新慢一些也看不出有什么问题。  
  我在项目中也实现了懒渲染功能，有一个大俯视场景有25万面，光照没有变化，影子是烘焙的，变化的只有水面和会飘的树，所以我将静态物体渲染完后，将颜色和深度保存下来，相机不动、分辨率不变的情况下，每帧Blit这张纹理到颜色和深度Attachment RT，继续渲染动态物体。帧率有效提升，更重要的是减少了渲染压力，使得发热和耗电降低很多。  
  其他游戏也有类似的操作，比如阴阳师的町中，一开始是43帧，5分钟不操作变成20帧，再次操作变为30帧。  
  降采样渲染水体半透特效在原神手机端中有实现，分帧更新ShadowMap也是，并且PC分8级，手机分4级。  
  可以感觉到降低频率、分辨率的优化方式往往有很大限制，但如果条件刚好能满足，往往能得到不少的优化，可以审视下自己的项目，看看有没有符合这些条件的场景。  
  但也要注意不要过犹不及，复杂的管线也会使得通用性不如常规管线，出现新功能时很容易出现BUG。

##### 2.5.2 发热、耗电

  温度其实分为CPU温度、电池温度等。CPU温度可以用PerfDog测试，或者用ADB代码`adb shell cat /sys/devices/virtual/thermal/thermal_zone0/temp`，得到的单位是毫摄氏度  
‘

![](https://upload-images.jianshu.io/upload_images/7875603-535f70760f4256f1.png)

  
  电池温度可以用SnapDragon测试。  
  发热一般是渲染的东西太多、像素绘制率过大、更新频率过高，我对发热也是没什么特殊方法，建议就是限帧，测试我们自己的项目时，将60帧限制到30帧，CPU温度能从60°降低到48°上下。  
  上面的PerfDog工具也可衡量耗电。  
  上图中Power是功耗，Voltage是电压，Current是电流，遵从公式P=UI。手机的电压恒定（但外部条件不同，例如运行了其他app，会导致电压不同），电流越大，功耗越高。  
  运行时点右上角的三角Play键可以记录数据，结束后数据会上传到云端，点击右上角进入云端管理界面：

![](https://upload-images.jianshu.io/upload_images/7875603-11a8fb8b0db451fd.png)

  进去后找到你的APP点进去，拉到最下面可以看到电池信息，包括统计后的平均功耗、总能量消耗、平均电压、平均电流：

![](https://upload-images.jianshu.io/upload_images/7875603-aa6cb3d70856d3f5.png)

  
  电池容量的单位是mAh：

![](https://upload-images.jianshu.io/upload_images/7875603-1c50b923fb0f84f8.jpg)

MI9

由于手机的电压基本恒定，所以能量消耗可以用电流衡量，上图MI9中3300mAh可以理解为，以3300mA可以放电一个小时。  
  所以如果不讨论其他应用、硬件设备，只看应用本身能持续多久，用电池容量除以平均电流即可。

![](https://upload-images.jianshu.io/upload_images/7875603-26721d2f68ec28a6.png)

PerfDog文档中的计算方式

  

* * *

  
  除了限帧降频外，我还建议检查管线，减少不必要的全屏Blit操作，例如相机复制出的\_CameraOpaqueTexture\\\_CameraDepthTexture，当没有效果使用这些纹理时可以勾掉；或者其他自定义的Blit，后处理Blit次数也尽量减少，例如URP低版本的Bloom有25个Pass(2340x1080分辨率)，可以想办法减少升降RT，做到效果和效率均衡。全屏操作即使什么都不做，只是单纯拷贝纹理，也有不小的性能消耗。

三、其他
====

### 1\. 卡顿

  不同于前面提到的帧率、带宽，卡顿往往就是突发的出现一次或几次，但也会影响用户体验。  

  卡顿可以用Unity的性能检查工具UPR：

![](https://upload-images.jianshu.io/upload_images/7875603-dffcee5a83439256.png)

  
  点击CPU->CPU函数性能占用->流畅度，能看到小卡顿(100ms左右)和大卡顿，点击下面的show on charts可以看到下面的Timeline中将卡顿点标记出来。  

  缩放区间，选择具体帧：

![](https://upload-images.jianshu.io/upload_images/7875603-ac432d967cbeccf7.png)

  
  向下拉可以看到这一帧的耗时分布图和各函数及调用堆栈性能占用：

![](https://upload-images.jianshu.io/upload_images/7875603-bdd9c33917dd5ce2.png)

。  
  做卡顿检查时，要把Log关闭，因为Log太多也会导致卡顿和GC，对我们优化造成影响。  
  刚开始时我们将资源加卸载集中到场景加载时，导致场景加载时间很长，于是我们让管理这方面的人将资源加卸载平摊，降低了不少场景加载时间。  

  还有一点Shader加载相关的问题，Shader被首次使用时从Bundle中加载并编译，这样可能导致运行时卡顿，所以用Unity的变体收集功能，在使用前统一预热（Warmup）。  
  但这一步没有正确使用，可能导致加载时间过长。变体收集尽量只收集项目中使用的，不使用的变体尽量去掉。我们有一个Shader剔除变体前Shader.Parse要60ms，剔除后只要0.2ms。  
  其他造成卡顿的原因也有，需要具体情况具体分析。

### 2\. 内存

##### 2.1 GC

  平时写程序没注意到，就可能产生GC，字符串拼接、一直new class、拆装包、Linq、Lambda表达式捕获外部引用等，甚至低版本UPR都存在实时GC，先用Unity的Profiler找到每帧实时GC干掉。  

  大致方法是：开启Deep Profile，选择Hierarchy，运行，选择GC排序，查看函数调用堆栈：

![](https://upload-images.jianshu.io/upload_images/7875603-b98413e12afec70a.png)

##### 2.2 检查资源

  UPR工具可以截取内存快照和对象快照，可以看到截取时的内存占用：

![](https://upload-images.jianshu.io/upload_images/7875603-8256d034d180c384.png)

  以及占用的具体资源有哪些：

![](https://upload-images.jianshu.io/upload_images/7875603-dd1ef6d6afc0e910.png)

  
  如果有资源超乎预料的大，或者在截取时应该被卸载却依然驻留内存，就需要排查资源状况了。  
  有问题的资源加卸载、错误的纹理压缩格式、未经优化的Shader变体、未优化的动画、未减面的网格，都可能导致内存占用过大。

##### 2.3内存泄漏

###### 2.3.1 初步排查

  一般内存泄漏是指资源由程序申请，但因为某些原因程序失去了资源的句柄，导致无法释放内存。比如C++一直new对象赋值一个指针变量，之前申请的对象没有delete，也没有被任何指针持有，就导致泄漏。  

![](https://upload-images.jianshu.io/upload_images/7875603-a3691fcf7e268263.png)

  
  PerfDog、UPR等工具都能看到内存占用曲线，如果感觉内存不正常的增长，有可能是内存泄漏问题导致。可以用多个工具尝试解决问题。  
  首要问题是找到内存泄漏的大致区域，为了减少干扰，建议用打出包的真机程序分析。  
  首先用PerfDog找到使内存升高的大致操作，记录下来。  
  然后用Unity的Profiler工具，点击Memory，模式改成Simple:

![](https://upload-images.jianshu.io/upload_images/7875603-3a314fc29fafd5b7.png)

  
  其中Reserved Total是Unity向系统申请的内存，而Used Total是其中被使用的内存，当Used Total中内存超过Reserved Total中内存时，内存会进行扩容。  
  理论上反复运行同一功能，资源会重复加载卸载（或者缓存），内存峰值应该是大致不变的，但如果某个地方泄漏出去，Used Total会变高，导致内存使用峰值变高，最后Reserved Total也不断升高。  
  其中各项说明可以看Unity的[官方文档](https://links.jianshu.com/go?to=https%3A%2F%2Fdocs.unity3d.com%2FManual%2FProfilerMemory.html)，注意不同Unity版本可能会有区别。

###### 2.3.2 内存快照

  查到了大致泄漏处，就要继续往深排查，这个地方最好是有对项目资源特别了解的人帮忙协助，知道什么地方应该有什么资源，什么时候某个资源应该被卸载。  
  Unity的Profiler可以截取内存，用真机截取时将不会看到那么多编辑器对象。

  

![](https://upload-images.jianshu.io/upload_images/7875603-86c283628328cf05.png)

  

  截取后可以查一查哪个对象在当前场景不应该存在，无论是网格、粒子、音频、RT等等。  
  其次还可以做两个快照的对比，Unity有一个专门用于检查内存的工具MemoryProfiler，这是个Package，需要到PackageManager中安装。  

  截取两个快照，选择后按Diff进行对比：

![](https://upload-images.jianshu.io/upload_images/7875603-887ffc3e9f1add77.png)

  
  可以用上面的表头分组，先用Diff列分组，找到New那一组，然后用Type分组。

###### 2.3.3 Mono

  Mono是被程序GC管理的内存，这方面增长可用UWA GOT工具排查。

  

![](https://upload-images.jianshu.io/upload_images/7875603-f549599cc1958525.png)

  

  模式选择Persistent，显示总值，根据persistentBytes进行排序，当函数的selfPersistentCounts不为0时，右侧会显示当前快照中当前函数的堆内存占用。

最后编辑于 ：2022.08.23 20:09:12

©著作权归作者所有,转载或内容合作请联系作者  
平台声明：文章内容（如有图片或视频亦包括在内）由作者上传并发布，文章内容仅代表作者本人观点，简书系信息发布平台，仅提供信息存储服务。

13人点赞

[Unity3D](/nb/40003904)

更多精彩内容，就在简书APP

![](https://upload.jianshu.io/images/js-qrc.png)

"小礼物走一走，来简书关注我"

赞赏支持还没有人赞赏，支持一下

[![](https://upload.jianshu.io/users/upload_avatars/7875603/08ae0111-1e2e-4ac9-8cc8-bf0b6ecd9def.png?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)
](/u/50094e11ed08)

[crossous](/u/50094e11ed08 "crossous")主C++、Python<br>目前是图形、渲染、引擎方向。<br>音游、ACT、ACGN爱好者。

总资产15共写了10.8W字获得348个赞共315个粉丝

关注

### 

全部评论2只看作者

按时间倒序

按时间正序

[![](https://upload.jianshu.io/users/upload_avatars/22684411/306893ce-19fd-432c-baff-7eefbc88df21.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/80/h/80/format/webp)
](/u/dd4182618aaf)

[牛头人酋长](/u/dd4182618aaf)IP属地: 浙江

3楼 2021.11.26 09:06

好文章，已经收藏到我的站点，感谢作者！

赞 回复

[![](https://upload.jianshu.io/users/upload_avatars/140424/eb239499c86e.gif?imageMogr2/auto-orient/strip|imageView2/1/w/80/h/80/format/webp)
](/u/e0ef35423b18)

[天壤](/u/e0ef35423b18)IP属地: 湖北

2楼 2021.08.19 11:01

nb，期待后续更多关于性能优化的分享~

赞 回复

### 推荐阅读[更多精彩内容](/)

*   [Unity 2019 新特性在次时代手游《黑暗之潮》中的应用经验及技术分享](/p/3c2b29058653)
    
    Unity 2019 新特性在次时代手游《黑暗之潮》中的应用经验及技术分享 林若峰 技术专家 ILRuntime作...
    
    [![](https://cdn2.jianshu.io/assets/default_avatar/14-0651acff782e7a18653d7530d6b27661.jpg)
    landon30](/u/21add3dce532)阅读 1,801评论 0赞 0
    
*   [\[Unity优化\] 深入浅出聊Unity3D项目优化：从Draw Calls到GC](/p/5097feffc90e)
    
    匹夫印象里遇到的童靴，提Unity3D项目优化则必提DrawCall，这自然没错，但也有很不好影响。因为这会给人一...
    
    [![](https://cdn2.jianshu.io/assets/default_avatar/8-a356878e44b45ab268a3b0bbaaadeeb7.jpg)
    hcq666](/u/8271fe4506a3)阅读 829评论 0赞 52
    
*   [【转】移动平台Unity3D 应用性能优化](/p/e2604978f35a)
    
    转载http://wetest.qq.com/lab/view/315.html 移动平台硬件架构 移动平台无论是...
    
    [![](https://cdn2.jianshu.io/assets/default_avatar/4-3397163ecdb3855a0a4139c34a695885.jpg)
    李嘉的博客](/u/aaa3e4347850)阅读 1,739评论 0赞 4
    
*   [移动平台Unity3D 应用性能优化(转)](/p/4f1f333272e9)
    
    移动平台Unity3D 应用性能优化 文章比较长，但是满满的是干货。 一、移动平台硬件架构 移动平台无论是Andr...
    
    [![](https://upload.jianshu.io/users/upload_avatars/15536448/685ab438-e2ec-4794-b921-f4a99a725e54.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/48/h/48/format/webp)
    雄关漫道从头越](/u/cd18446dfb5f)阅读 1,343评论 0赞 4
    
*   [Unity 优化归纳](/p/0d0f061a4de7)
    
    原文地址 http://www.fx114.net/qa-75-172454.aspx 使用Profiler工具...
    
    [![](https://cdn2.jianshu.io/assets/default_avatar/13-394c31a9cb492fcb39c27422ca7d2815.jpg)
    IongX](/u/17500d9bba1c)阅读 5,898评论 1赞 11
    

[![](https://upload.jianshu.io/users/upload_avatars/7875603/08ae0111-1e2e-4ac9-8cc8-bf0b6ecd9def.png?imageMogr2/auto-orient/strip|imageView2/1/w/90/h/90/format/webp)
](/u/50094e11ed08)

[crossous](/u/50094e11ed08)

关注

总资产15

[团结引擎在鸿蒙上的线程模型研究](/p/de336ca41317)

阅读 150

[OpenHarmony napi异步调用ets的方法](/p/4e49e9fcde12)

阅读 79

### 热门故事

[桂林志异：龙王起水](https://www.jianshu.com/p/9f168be225b0)

[离婚后，妈宝男前夫后悔了](https://www.jianshu.com/p/ea8d12548641)

[救了他两次的神仙让他今天三更去死](https://www.jianshu.com/p/1ded57e57939)

[我把眼角膜捐给丈夫的白月光后，他疯了](https://www.jianshu.com/p/946923c8224f)

[为了活命，我对病娇反派弟弟表白，他竟当真要做我夫君](https://www.jianshu.com/p/abb1ac30da8b)

[“有个坐过牢的富豪老公是种什么体验？”“要不然你来试试？”](https://www.jianshu.com/p/0f38a77e0bd7)

[前世渣男把我迷晕还叫我别怕，重生后我杀疯了](https://www.jianshu.com/p/e8cfb8f2154e)

[妹妹过失杀人，警察来时，我捡起了那把滴血的刀](https://www.jianshu.com/p/3a1958255a4a)

[我被校霸堵在巷口，却发现他是我谈了三个月的网恋对象](https://www.jianshu.com/p/875b89eba857)

[我首富之女的身份居然被人偷了](https://www.jianshu.com/p/dbe39342ae8e)

评论2

赞13

13赞14赞

赞赏

![](data:image/gif;base64,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)

手机看全文

扫码安装简书客户端

畅享全文阅读体验

![](https://upload.jianshu.io/images/js-qrc.png)

扫码后在手机中选择通过第三方浏览器下载
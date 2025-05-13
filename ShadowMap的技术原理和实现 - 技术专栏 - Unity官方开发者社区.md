# ShadowMap的技术原理和实现 - 技术专栏 - Unity官方开发者社区
     ShadowMap的技术原理和实现 - 技术专栏 - Unity官方开发者社区                                

[![](https://developer-prd.cdn.unity.cn/assets/styles/i/svgicons/icons-unity-logo.d83bd481b45224727fbe91edf6ce0085.svg)
](/)

[

Unity官网

](https://unity.cn/)

[

社区

](/)

[

问答

](/ask)

[

文章

](/articles)

[

课堂

](https://learn.u3d.cn/)

[

活动

](https://developer.unity.cn/events)

文档

[

Unity中文文档

](https://docs.unity.cn/cn/current/Manual/UnityManual.html)[

团结引擎文档

](https://docs.unity.cn/cn/tuanjiemanual/Manual/)

[

团结资源商店

](https://assetstore.u3d.cn/)

提问

问AI

下载团结引擎

下载Unity引擎

Unity ID

Unity ID允许您购买和/或订阅Unity产品和服务，在Asset Store中购物并参与Unity社区。

登录

![](https://developer-prd.cdn.unity.cn/assets/styles/i/cn/reputation/img-fame@2x.e68cc19a173f906559596c12a4f7d9c4.png)

![](https://developer-prd.cdn.unity.cn/assets/styles/i/cn/reputation/img-fame-strew@4x.a73f66b9ecd03738ce6c84280016f7b5.png)

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20211022/p/images/505b4d4e-bded-4c98-8c3e-b934e5363ce5_img_create_article_pattern_cover1_2x.png)

0

0

ShadowMap的技术原理和实现
=================

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230321/74aa922d-334b-4c13-9e6c-47a854fdedc2)

Careless

关注

阅读 1668

2023年4月11日

当光源发射的光线遇到一个不透明物体时，除了部分反射光外，光线无法照亮其他物体，此时这个物体就会向旁边的物体投射阴影。这些阴影区域的产生是因为光线无法到达这些区域。

> 阴影除了和光源有关，还和材质、shader有关，可以通过自定义shader来定义阴影的渲染方式。

当光源发射的光线遇到一个不透明物体时，除了部分反射光外，光线无法照亮其他物体，此时这个物体就会向旁边的物体投射阴影。这些阴影区域的产生是因为光线无法到达这些区域。

传统实时阴影（ShadowMap）


---------------------

unity使用这种ShadowMap的技术来模拟实现物理世界的投影现象，它的实现原理非常简单。

假设有一个模拟太阳位置的摄像机，在可观察的视线范围内，物体表面各个点到摄像机的距离可以组合成一张深度图，它记录了从该光源的位置出发、能看到的场景中距离它最近的表面位置。这张深度图就叫做ShadowMap。（图中紫色部分表示阴影区域）

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230411/p/images/3e78eca0-0f52-4592-a63a-14a84a4790d2_Pasted_image_20230410181403.png)

深度图是通过模拟太阳位置的摄像机获取到的，并非真正渲染场景的视角，真正渲染场景的视角，是B点所在位置的相机。

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230411/p/images/ca7b5ee6-c049-47c6-8ddc-d9d2f26d42b7_Pasted_image_20230410182312.png)

假设B观察到的三个点p1、p2、p3，在渲染的过程中，首先会把这三个点的顶点位置变换到光源空间下，这样就能得到它们在光源空间中的三维位置信息：sun(p1)、sun(p2)、sun(p3)，

使用xy分量对ShadowMap进行纹理采样，就能获得它们的深度信息sun-depth(p1)、sun-depth(p2)、sun-depth(p3)

B视角下这三个点的深度信息可以通过z分量获得，它们的深度信息是B-depth(p1)、B-depth(p2)、 B-depth(p3)

通过同一位置在不同视角下世界空间的深度值对比，就能判断出该点是否处于阴影中。如果阴影深度值小于顶点深度值，那么这个点就处在阴影中。否则这个点就被光源照亮。

*   sun-depth(p1) > B-depth(p1)，照亮
    
*   sun-depth(p2) < B-depth(p2)，阴影
    
*   sun-depth(p3) < B-depth(p3)，阴影
    

以上就是传统实时阴影shadowMap的基本技术原理。

屏幕空间阴影映射（ScreenspaceShadowMap）


----------------------------------

在Unity5中，Unity又使用了不同的阴影技术，即屏幕空间阴影映射技术（ScreenspaceShadowMap）。

当使用了屏幕空间阴影映射技术时，Unity首先会通过调用LightMode为ShadowCaster的Pass来得到ShadowMap以及摄像机的深度纹理（Camera-depth）。然后根据ShadowMap和深度纹理来得到屏幕空间的阴影图(Shadow-depth)。判断顶点是否处于阴影中，遵循以下规则：

*   Camera-depth > Shadow-depth，照亮
    
*   Camera-depth < Shadow-depth，阴影
    

再来说说一种辅助性的阴影技术：联级阴影（Cascaded Shadow Mapping）

联级阴影在unity中是配合屏幕空间阴影使用的，在渲染的过程中通过设置不同级别的阴影，通过判断用户到物体之间的距离，来切换使用不同级别的阴影。

阴影的基本设置


-----------

在设置使用哪种阴影技术之前，首先要确保光源ShadowType的基本设置。

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230411/p/images/daa1cfec-707d-4e70-9a72-ecdd1b89ffe8_Pasted_image_20230410232126.png)

其次是确认物体拥有投射或接受阴影的能力，这是通过Mesh Renderer组件中的Cast Shadows 和 Receive Shadows属性来实现的。

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230411/p/images/650958f4-0103-45b5-8b67-d3c6fe7f6c03_Pasted_image_20230410232233.png)

开启了Cast Shadows，unity就会把该物体添加到光源的ShadowMap的计算中去。而Receive Shadows则可以选择是否让物体接收来自其他物体的阴影。

默认情况下，这些属性都是已经设置好了的，这里只是提醒注意。

默认情况下，unity使用的是屏幕空间 + 联级阴影的技术实现阴影，要重现传统ShadowMap阴影技术，需要关闭unity的屏幕空间阴影映射+联级阴影技术

可以在Edit-Projecr Settings-Quality中查看shadows的设置，Shadow Distance对于阴影的质量有很大影响，距离越远越粗糙，距离越近越精细

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230411/p/images/6695005b-0c0e-4830-814b-c644bb5b287c_Pasted_image_20230410105056.png)

在Graphics中取消默认设置，取消勾选Cascaded Shadows勾选。这样unity就采用了传统的实时阴影渲染技术。

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230411/p/images/d07b8daa-46c2-4deb-9040-aa6b15eb340d_Pasted_image_20230410105400.png)

在使用屏幕空间阴影映射技术时，Quality设置中shadow cascades 选择 Two Cascades 或者 Four Cascades中，就等于是在屏幕空间坐标阴影的基础上再加上了联级阴影。

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230411/p/images/6ff4d7a5-c370-43e0-9aee-002ce349cebb_Pasted_image_20230410145359.png)

勾选这里的选项可以查看联级阴影的距离范围

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230411/p/images/6d35a247-d836-4a30-a24d-3071033f4257_Pasted_image_20230410145511.png)

以上就是三种阴影技术的基本设置。

自定义阴影Shader


---------------

先简单搭建一个场景，给立方体添加一个最简单的着色器unlit shader，再添加一个unlit shader的材质，此时，立方体表面没有任何光照信息，也不会产生阴影。

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230411/p/images/fec3e307-bade-46d7-9e6b-cecf77c7f1ae_Pasted_image_20230411090106.png)

结合上面阴影的设置，只需要加上简单的一段代码，就能实现一个简单的阴影：

```
```


Fallback "Diffuse"


```


```

Diffuse是Unity shader封装好的产生阴影投射的shader，它的优点是方便快捷，缺点是无法自定义。

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230411/p/images/76f30ef6-b31e-467b-a71a-dab7d1d59a64_Pasted_image_20230411092520.png)

要实现自定义Pass，需要使用以下代码来实现：

```
```


  // Pass Tags Tags {"LightMode" = "ShadowCaster" }  #include "AutoLight.cginc"  // struct v2f  SHADOW\\\_COORDS(5)  // vef vert  TRANSFER\\\_SHADOW(o);  // half4 frag  SHADOW\\\_ATTENUATION(i);


```


```

这段代码，主要还是用来实现传统实时阴影技术。

使用了LightMode = "ShadowCaster“这条语句，渲染引擎会先在当前shader找到对应的Pass来渲染投射阴影，如果没找到，就会在Fallback中继续查找，如果仍然没有找到，它就不会投射阴影。

自定义Pass 渲染投射阴影往往比较复杂，这涉及到光照模型的计算，有感兴趣的，可以查看以往几篇关于光照模型计算的文章。

Unity 5及以后的版本，已经采用了屏幕空间阴影+联级阴影技术，因此更多是采用以下代码来实现：

```
```


// Pass Tags Tags {"LightMode"  \=  "ShadowCaster"  }  #include "AutoLight.cginc"  // struct v2f  LIGHTING\\\_COORDS(5,6)  // vef vert  TRANSFER\\\_VERTEX\\\_TO\\\_FRAGMENT(o);  // half4 frag  LIGHT\\\_ATTENUATION(i);


```


```

以上代码的写法，渲染引擎在处理阴影时，可以帮助判断灯光类型，从而计算光照衰减范围、Cookies等等。

Frame Debugger 查看阴影绘制过程


---------------------------

我们可以通过Window - Frame Debugger中打开帧调试器，来查看阴影的渲染过程。

阴影渲染大致分为四个阶段：

第一阶段：UpdateDepthTexture，这个阶段主要是更新摄像机的深度问题。

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230411/p/images/722f4c36-953d-4585-901b-f9265d42589f_Pasted_image_20230411102925.png)

第二阶段：RenderShadowMap，这个阶段是渲染得到光源的ShadowMap

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230411/p/images/add92245-dfa3-4447-93d7-50521c53a2e3_Pasted_image_20230411103547.png)

第三阶段：CollectShadows，这个阶段是根据深度纹理和ShadowMap，计算得出屏幕空间阴影图。

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230411/p/images/caa9462c-c845-4f43-8362-4b14c2f60370_Pasted_image_20230411103646.png)

第四阶段：绘制渲染结果，这个阶段没什么好说的。

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230411/p/images/304363b4-1386-41b9-96b5-e95b70665d96_Pasted_image_20230411103725.png)

最后再给一段示范代码，看看Shadow阴影技术结合光照模型产生的一个最终效果：

```
```


// Pass1 half3 diff\\\_term \=  min(shadow,man(0.0,dot(normal\\\_dir,light\\\_dir))); half3 diffuse\\\_color \= diff\\\_term \_\\\_LightColor0.xyz\_ base\\\_color.xyz \\\* attuenation; half3 spec\\\_color \=  pow(max(0.0,RdotV),\\\_Shininess) \_diff\\\_term\_ \\\_LightColor0.xyz \_\\\_SpecIntensity\_ spec\\\_mask.rgb;  // Pass2 Tags {  "LightMode"\="ForwardAdd"  } Blend One One #pragma multi\\\_compile\\\_fwdadd  // struct v2f  LIGHTING\\\_COORDS(5,6)  //v2f vert  TRANSFER\\\_VERTEX\\\_TO\\\_FRAGMENT(o);  //half4 frag  half3 shadow \= LIGHT\\\_ATTENUATION(i); half3 final\\\_color \=  (diffuse\\\_color + spec\\\_color)\\\*ao\\\_color;  return  half4(final\\\_color,1.0);


```


```

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230411/p/images/ae04397c-2f55-46b8-ae1c-9ea9d6dfe7f7_Pasted_image_20230410153601.png)

[

shadowmap

](/search?q=shadowmap)

发布于[**技术交流**](/plate/unity-know-how)

推荐阅读

[

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230415/p/images/d5a21775-2b9a-4e02-906a-4f839ed3c7d8_Pasted_image_20230415111749.png)

球谐光照的技术实现

2023-04-15

阅读 893





](/projects/643a5fd5edbc2a001e526836 "recommend-card")[

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230415/p/images/8369aa58-c1c8-480b-bd7e-54b00bf6fae2_Pasted_image_20230412171857.png)

IBL环境贴图的技术实现

2023-04-15

阅读 925





](/projects/643a28c9edbc2a293e4baa57 "recommend-card")[

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230412/p/images/b284a324-2860-4b62-99a8-54094664da4f_Pasted_image_20230412110842.png)

立方体贴图的技术原理和实现

2023-04-12

阅读 1116





](/projects/64362f3aedbc2a001e520856 "recommend-card")[

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20211021/p/images/c1892f7d-1afd-4bd1-9d60-0b39793e535a_img_cover_article_1_3x.jpg)

ShadowMap的技术原理和实现

2023-04-11

阅读 1668





](/projects/6434d104edbc2a001ee7a032 "recommend-card")[

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230404/p/images/af5c4875-dbe6-47cd-a216-a890d8335c01_Snipaste_2023_04_04_12_13_28.png)

光照模型的原理和实现思路——Phong

2023-04-04

阅读 761





](/projects/642ba4eaedbc2a697e79c661 "recommend-card")[

![](https://u3d-connect-cdn-public-prd.cdn.unity.cn/h1/20230402/p/images/14b74421-1faf-469d-82a0-3d07b99906a9_Snipaste_2023_04_02_21_50_00.png)

Unity中的渲染路径RenderingPath

2023-04-02

阅读 865





](/projects/64298442edbc2a6cce16352f "recommend-card")

0条评论

![](https://developer-prd.cdn.unity.cn/assets/styles/i/svgicons/glyphs-social-wechat.6e27815b78d393005e6ff32406ea94a8.svg)

分享到微信

[

![](https://developer-prd.cdn.unity.cn/assets/styles/i/svgicons/glyphs-social-weibo.61d7121127ea1ebd9f6bf8a63ccd2f7f.svg)

分享到微博

](http://service.weibo.com/share/share.php?url=https://developer.unity.cn/projects/6434d104edbc2a001ee7a032 ShadowMap的技术原理和实现 - 技术专栏 - Unity官方开发者社区)

![](https://developer-prd.cdn.unity.cn/assets/styles/i/svgicons/outline-link-v2.dc5e0905579846f8e4183112fab6c25c.svg)

复制链接

Copyright © 2025 优三缔科技（上海）有限公司 版权所有

"Unity"、Unity 徽标及其他 Unity 商标是 Unity Technologies 或其附属机构在美国及其他地区的商标或注册商标。其他名称或品牌是其各自所有者的商标。

![](https://developer-prd.cdn.unity.cn/assets/styles/i/china.d0289dc0a46fc5b15b3363ffa78cf6c7.png)
沪公网安备: [31010902002961号](http://www.beian.gov.cn/portal/registerSystemInfo?recordcode=31010902002961)

[法律条款](https://unity.cn/legal/developer-terms)[隐私政策](https://unity.cn/legal/developer-privacy)[Privacy Policy](https://unity.cn/legal/privacy-policy)[Cookies](https://unity.cn/legal/china-cookie-policy)

[沪ICP备2022020141号-1](http://beian.miit.gov.cn)

[联系我们](https://unity.cn/contact)

下载Unity (提供各种版本下载)

[所有版本](https://unity.cn/releases)[LTS版本](https://unity.cn/releases/lts)[补丁版本](https://unity.cn/releases/patch)[Beta版本](https://unity.cn/releases/beta)

我们使用 Cookies 来个性化和提升您使用我们的网站的体验。请访问我们的[《Cookie政策》](https://unity.cn/legal/china-cookie-policy)了解更多信息。点击"同意"表示您同意使用 Cookies。

禁止使用cookies

同意

![](https://developer-prd.cdn.unity.cn/assets/styles/i/cn/muse-bot-avatar.4571bbc9fae91d0a67acb54c789df3af.png)

问

AI
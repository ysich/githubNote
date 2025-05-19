# 为Unity Sprite实现GPU Instancing
  为Unity Sprite实现GPU Instancing                       

OWNSELF

*   [About Me](/aboutme)
*   [Search](/search)

为Unity Sprite实现GPU Instancing
=============================

Posted on March 6, 2022

概述
==

公司里有一个使用Unity开发的2D像素风格的战棋类的游戏项目，其实现上采用了3D场景+精灵动画人物的方案，配合了Unity URP渲染管线，静态场景是可以利用SRP Batcher来提升Graphics API方面的性能表现，但是很遗憾到项目正在使用的Unity 2020.3为止，Unity的Sprite Renderer还没有实现对SRP Batcher的支持。于是我的目标就是根据他们游戏的特点尽可能的提供性能改进的方案。

资产特性
====

这次优化的目标主要围绕游戏中使用的Sprite Renderer组件，也就是游戏中人物角色的绘制，项目中与人物相关的资产具有以下一些特征：

*   游戏中所有角色绘制使用的Shader是相同的
*   角色的精灵动画使用帧动画实现
*   每个角色的精灵帧动画在同一张Atlas中，并通过Sprite Mode “Multiple”和Sprite Editor来编辑并导入工程
*   每个角色的Atlas贴图格式相同且大小均为512X512

优化思路
====

既然Shader都是相同的，那么合批是有前提保障的；而贴图方面角色贴图规格相同也使得使用TextureArray成为可能，那么剩下的问题就是Mesh了，而Sprite的Mesh刚好又是可以通过简单的Quad即可实现的，于是我们的目标就是利用自定义的MeshRenderer来代替Unity原生的Sprite Renderer了。

经过一些调试和阅读SpriteRenderer的源代码实现，我们发现即使将Sprite的MeshType设置为“Full Rect”，SpriteRender实际上也会为每一个不同的Sprite生成不同的Mesh信息，原因是不同的Sprite会有不同的尺寸（PixelPerUnit）和偏移等信息，但其计算逻辑比较简单，我们是可以通过相同的Mesh（比如Unity内建的Quad）配合不同的材质属性来在顶点着色器中实现具体的缩放和偏移的。

Unity中合批性能最好的GPU Instancing的要求之一是使用相同的模型，既然理论上可以通过相同的Quad模型，那么我们可以尝试来实现人物精灵的GPU Instancing了。

缩放与平移
=====

为了保证使用了统一Quad模型的MeshRenderer在渲染Sprite的时候可以将Sprite正确的大小和偏移表现正确，我们需要计算好尺寸与偏移，并将其保存在MaterialPropertyBlock中，以供顶点着色器可以在世界坐标变换前，先在模型空间根据这些信息进行变换以复现SpriteRenderer中原本的坐标位置。

这里需要特别注意的是SpriteRenderer中生成的Mesh和Unity内建的Quad的Mesh虽然都是两个三角形，但是顶点和索引的顺序确实完全不同的，这个暗坑耽误了我不少的时间。

![](https://www.ownself.org/wp-content/uploads/2022/03/UnitySpriteGPUInstancing.png)

计算尺寸和偏差的逻辑代码：

```
// 根据Sprite的尺寸和偏移计算Quad的缩放与平移关系
Pivot.x = sprite.rect.width / sprite.pixelsPerUnit;
Pivot.y = sprite.rect.height / sprite.pixelsPerUnit;
Pivot.z = ((sprite.rect.width / 2) - sprite.pivot.x) / sprite.pixelsPerUnit;
Pivot.w = ((sprite.rect.height / 2) - sprite.pivot.y) / sprite.pixelsPerUnit;
// 根据顶点顺序的不同重新映射正确的UV范围
newUV.x = sprite.uv[1].x - sprite.uv[0].x;
newUV.y = sprite.uv[0].y - sprite.uv[2].y;
newUV.z = sprite.uv[2].x;
newUV.w = sprite.uv[2].y; 
```

然后我们在VS中通过手工生成变换矩阵的方式在世界变换之前应用正确的缩放与平移变换：

```
half4 pivot = UNITY_ACCESS_INSTANCED_PROP(Props, _Pivot);
// 生成缩放平移矩阵
half4x4 m;
m._11 = pivot.x; m._12 = 0; m._13 = 0; m._14 = pivot.z;
m._21 = 0; m._22 = pivot.y; m._23 = 0; m._24 = pivot.w;
m._31 = 0; m._32 = 0; m._33 = 1; m._34 = 0;
m._41 = 0; m._42 = 0; m._43 = 0; m._44 = 1;
float4 vertex = UnityFlipSprite(input.positionOS.xyz, _Flip);
// 生成UV的变换矩阵
half3x3 uvm;
half4 newUV = UNITY_ACCESS_INSTANCED_PROP(Props, _NewUV);
uvm._11 = newUV.x; uvm._12 = 0; uvm._13 = newUV.z;
uvm._21 = 0; uvm._22 = newUV.y; uvm._23 = newUV.w;
uvm._31 = 0; uvm._32 = 0; uvm._33 = 1;
half3 uv = half3(input.texcoord.x, input.texcoord.y, 1);
// 执行UV变换矩阵
uv = mul(uvm, uv);
VertexPositionInputs vertexInput;// = GetVertexPositionInputs(vertex.xyz);
// 执行顶点缩放平移变换
vertexInput.positionWS = mul(m, input.positionOS); // convert as Sprite
vertexInput.positionWS = TransformObjectToWorld(vertexInput.positionWS).xyz;
// ......
// 将变换后的UV赋值给片元的数据结构
output.uv.xy = uv.xy; 
```

贴图数组
====

Texture Array的准备相对要简单很多，我们可以在Loading时候创建Texture Array并将关卡中必要的贴图拷贝进来，需要一点逻辑来维护贴图在数组中的位置以及可能动态出现的新贴图。在OpenGL 3.0中Texture Array的数量已经支持到256，Metal API更是支持到2048，相信应该是够用的，万一真不够也是可以通过逻辑管理多个贴图数组来实现最大可能的合批的。

此外需要注意最好是利用Graphics.CopyTexture()接口来初始化Texture Array，因为是在GPU上进行的，速度较快且无需拷贝回系统内存。

着色器语法
=====

这里列举Shader中一些必要的语法：

*   启用贴图数组的必要宏：#pragma require 2darray
*   启用GPU Instancing的必要宏：#pragma multi\_compile\_instancing
*   声明贴图数组：Texture2DArray \_TextureArray;
*   贴图采样：\_Textures.Sample(my\_point\_clamp\_sampler, float3(uv, \_TextureIndex)
*   以及GPU Instancing必要的MaterialPropertyBlock中对应Buffer的声明

Light Probe的支持
==============

另外Unity从2018开始已经为GPU Instancing实现了对Light Probe的支持，但支持是依赖引擎中的一些固定操作的，因此我们需要保证我们的Shader中一些变量声明符合引擎中的规范，才能保证Light Probe可以正确的配合GPU Instancing工作。我们需要将Light Probe有关的变量以明确的命名并确保他们声明在”UnityPerDraw”的CBUFFER字段中。

```
CBUFFER_START(UnityPerDraw)
// SH block feature
real4 unity_SHAr;
real4 unity_SHAg;
real4 unity_SHAb;
real4 unity_SHBr;
real4 unity_SHBg;
real4 unity_SHBb;
real4 unity_SHC;
CBUFFER_END 
```

在计算颜色的过程就可以使用引擎内置的计算方式来进行球谐的计算（运算逻辑来自URP内置球谐计算函数SampleSH9()）

```
// Linear + constant polynomial terms
float3 res = SHEvalLinearL0L1(output.normal.xyz, unity_SHAr, unity_SHAg, unity_SHAb);

// Quadratic polynomials
res += SHEvalLinearL2(output.normal.xyz, unity_SHBr, unity_SHBg, unity_SHBb, unity_SHC);

#ifdef UNITY_COLORSPACE_GAMMA
    res = LinearToSRGB(res);
#endif

output.vertexSH.xyz = max(half3(0, 0, 0), res); 
```

而在C#中，我们需要通过引擎提供的接口手动计算出球谐系数并将他们拷贝到MaterialPropertyBlock中，引擎同样提供了接口帮助我们完成；最后我们还要需要将Render的Light Probe Usage设定为”CustomProvided”，这样引擎可以正确的知道我们准备利用CBUFFER来进行球谐系数的保存。

```
// Set Light Probe info
Vector3[] position = new Vector3[1] { transform.position };
SphericalHarmonicsL2[] lightProbes = new SphericalHarmonicsL2[1];
Vector4[] occlusionProbes = new Vector4[1];
 
// Manually get proper SH values
LightProbes.CalculateInterpolatedLightAndOcclusionProbes(position, lightProbes, occlusionProbes);
materialPropertyBlock.CopySHCoefficientArraysFrom(lightProbes);
materialPropertyBlock.CopyProbeOcclusionArrayFrom(occlusionProbes);
 
// We have to set light probe mode to "CustomProvided", so Unity will copy and use them into CBUFFER
meshRenderer.lightProbeUsage = LightProbeUsage.CustomProvided; 
```

动态Atlas
=======

在我们实际将这个技术集成进项目工程时，因为开发过程中已经存在了大量不同尺寸的Sprite贴图，我们最后进一步实现了对于不同尺寸的Sprite贴图的动态合并图集的支持，这样的好处是无论工程中之前用到的贴图资源的尺寸大小，无论游戏运行时贴图的用途是角色还是场景装饰，都可以一个Instance全部画完，所有Sprite Render均通过一个Draw Call绘制完成，岂不快哉！

听上去比较唬人，但实际上的原理并不复杂：

*   Sprite Instancing会在运行时使用可能用到的最大贴图尺寸来创建初始的Texture2DArray（例1024×1024）
*   当不同尺寸的Sprite贴图加载进来的时候，我们通过Graphics.CopyTexture拷贝至Texture2dArray中，但拷贝的位置会根据算法自动分配其位于Texture2DArray中合适位置
*   需要实现算法来维护Texture2DArray中目前可以利用的贴图区域和已经被占用的区域，原理类似于Hashmap的方式，通过一个元素为LinkedList的数组实现，数组下标则是贴图高度以2为底的对数（例：下标为5的数组内的LinkedList中保存的是可用的Nx32的贴图区域）
*   当我们需要申请新的贴图的时候，根据申请的贴图的高度来在对应下标的数组的LinkedList中寻找最合适的可用区域，如果没有找到，则会下标+1，尝试在更大的区域中寻找，一旦找到后，将区域裁减开，一块返回给当前申请贴图使用，另一块根据大小再重新加回到数组对应下标的可用区域列表中，如此完成整个贴图数组的管理。

结论
==

这篇文章中的优化方案其实还是需要一些先决条件的，如果你们的工程也符合这些先决条件，又刚好需要类似的优化内容，那么希望这个方案可以帮到你，如果不能，也希望能为你提供一些思路。

项目中的资源不好公开出来，所以我将核心思路实现在了一个简单的[示例工程](https://github.com/ownself/UnitySpriteGPUInstancing)中，有需要的话，请自行参考，如果有发现问题或者更好的改进方案，也请不吝赐教。

![](https://www.ownself.org/wp-content/uploads/2022/03/SpriteInstancingResult.png)

*   [← Previous Post](/2022/book-of-year-2021.html "2021年书单")
*   [Next Post →](/2023/unity-memory-profile-on-android.html "Unity项目Android平台内存分析")

*   [Email me](mailto:liuzhenming@ownself.org "Email me")
*   [RSS](/feed.xml "RSS")
*   [GitHub](https://github.com/ownself "GitHub")
*   [Twitter](https://twitter.com/ownself "Twitter")
*   [Instagram](https://www.instagram.com/ownself "Instagram")

Jimmy Liu  •  2025  •  www.ownself.org  •  [Edit page](https://github.com/ownself/ownself.github.io/edit/main/_posts/2022-03-06-unity-sprite-gpu-instancing.md "Edit this page on GitHub")

Powered by [Beautiful Jekyll](https://beautifuljekyll.com)
# Unity SRP 实战（三）PCSS 软阴影与性能优化 - 知乎
![](https://pic1.zhimg.com/v2-52e2079108dc6de0ff33806855caf6fe_1440w.jpg)

在上一篇文章中我们借助 [Unity SRP](https://zhida.zhihu.com/search?content_id=190879012&content_type=Article&match_order=1&q=Unity+SRP&zhida_source=entity) 实现了 [CSM](https://zhida.zhihu.com/search?content_id=190879012&content_type=Article&match_order=1&q=CSM&zhida_source=entity)，但是仍然存在几个问题。首先阴影的形状非常 Aliasing，其次阴影交界边缘过于 sharp 不太符合自然规律，我们希望实现视觉上较有说服力的淡入淡出效果。再者，半影区域的大小和遮挡距离也有关，这也是我们要考虑的点。

PCF
---

先上个开胃菜。Percentage Closer Filter（PCF）是一种最常见的软化阴影的方式。在基础的 01 shadowmapping 中我们直接一刀切地将颜色二值化，而 PCF 拿到深度值并不会马上下结论断言是否在阴影中。通过检查该点周围一圈像素是否也在阴影中，根据周围像素的情况给出一个 “百分比” 来描述 shading point 的受光情况。：

![](https://pic1.zhimg.com/v2-4b923e7b79224c4d164b4538d6d9ba3c_1440w.png)

尽管从原理上说 shadowmapping 就和 “遵循物理” 无缘了，但是 PCF 算法实现简单开销小，并且给出能糊弄过去（高情商：大致令人信服）的视觉效果，在早期硬件条件较为拮据的时代还是笔非常划算的买卖。一个简单的 3x3 PCF 就能带来不错的视觉提升：

![](https://pica.zhimg.com/v2-67758adf013c0c564e1bf20dd8b86ce6_1440w.jpg)

这么做相当于从 light space 对 shadowmap 的结果做了滤波，将原来 01 分明的结果给平滑了。如果熟悉图像处理的话不难发现如果 PCF 用的 filter 半径越大，结果越模糊。PCF 虽然能实现 “软化” 但是效果不太符合物理。

真正的物理阴影和遮挡物距离 shading point 的遮挡距离有关，距离越远越模糊。如果熟悉光线追踪不难理解，阴影相当于对 facet light 的各个点都做 visibility judge，遮挡物离的越远，在法向半球上的投影区域 dw 也就越小，越 “遮不住” 所以阴影就越软：

![](https://pic1.zhimg.com/v2-1ae5a357fc9bd99a51075fe997d61158_1440w.jpg)

PCSS
----

Percentage Closer Soft Shadows（PCSS）是基于 PCF 的一种改进型软阴影算法。正如其名，PCSS 考虑了遮挡距离和阴影模糊度的关系，并通过合理的建模给出视觉上 persuadable 的结果。

刚刚我们讨论到遮挡距离决定了阴影的模糊程度，而不同的 filter 半径可以生成不同模糊程度的阴影。于是一拍脑袋很容易想到 PCSS 的工作流程：

1.  确定一个遮挡距离
2.  根据遮挡距离计算滤波半径
3.  按照 2 中的半径对做 PCF 以软化阴影

首先是遮挡距离的计算。理想的方法是将 shading point 和面光源处处做连线，这些连线和 Shadow Map 能划出一个红色区域。计算该区域内的平均遮挡距离：

![](https://pic1.zhimg.com/v2-417078f309c6049201d6e387cfc6eaa8_1440w.jpg)

使用和 PCF 类似的方法检查平均遮挡距离，这里认为 Light 放在阴影相机的近平面。首先根据光源大小和 Shadow Map 分辨率计算光源面积在 Shadow Map 上对应的像素范围，然后在这个范围内做 filter，注意仅在发生遮挡的时候才计入平均距离。

这里切记 **不能** 直接使用 shading point 的采样深度（即世界坐标直接投影到深度图上未经过任何偏移的采样深度）做遮挡平均深度。因为这么做的错误之处在于当一个点不在硬阴影中那么它的遮挡深度就是 0，相当于有一个跳变。于是阴影边缘就不会产生 “散开” 的结果而是像一刀切。说起来比较抽象看图就清楚了：

![](https://pic3.zhimg.com/v2-fe397e6bc9f04c9f4fb7c8bfbba1bb42_1440w.jpg)

最上面是 01 映射，中间是使用采样深度当遮挡深度的结果，最下边是老老实实计算平均遮挡深度的结果。计算平均遮挡深度时 searchWidth 需要根据光源大小来确定。具体操作为用的光源宽度 lightSize 除以正交投影视锥体的宽 orthoWidth 得到，两者都是世界坐标下的量。一个 7 x 7 的遮挡深度计算如下：

```text
float searchWidth = lightSize / orthoWidth;

...

float2 AverageBlockerDepth(float4 shadowNdc, sampler2D _shadowtex, float d_shadingPoint, float searchWidth)
{
    float2 uv = shadowNdc.xy * 0.5 + 0.5;
    float step = 3.0;
    float d_average = 0.0;
    float count = 0.0005;   // 防止 ÷ 0

    for(int i=-step; i<=step; i++)
    {
        for(int j=-step; j<=step; j++)
        {
            float2 unitOffset = float2(i, j) / step;  // map to [-1, 1]
            float2 offset = unitOffset * searchWidth;
            float2 uvo = uv + offset;

            float d_sample = tex2D(_shadowtex, uvo).r;
            if(d_sample>d_shadingPoint)
            {
                count += 1;
                d_average += d_sample;
            }
        }
    }

    return float2(d_average / count, count);
}
```

有了平均遮挡深度就可以计算 PCF 滤波的范围。这里使用相似三角形计算，其中 w\_light 是光源大小：

![](https://pic2.zhimg.com/v2-a62e0a65e0bf572cd2d9173ed2a86e15_1440w.jpg)

注意这些计算都是在世界坐标下进行的。首先要将平均遮挡深度转换为世界坐标下的距离，做到这一点需要知晓正交投影 near、far 平面相距多远，这里我取的是 \[-d, d\] 范围所以一个数字就能表示，然后进行一个线性放缩：

```text
// 世界空间下的距离, 计算 PCSS 用, 注意 Reverse Z
float d_receiver = (1.0 - d_shadingPoint) * 2 * orthoDistance;
float d_blocker = (1.0 - d_average) * 2 * orthoDistance;
```

然后带入相似三角形的公式，得到世界坐标下的 Filter 半径，最后转换为 \[0, 1\] 范围的纹理采样半径：

```text
// 世界空间下的 filter 半径
float w = (d_receiver - d_blocker) * lightSize / d_blocker;

// 深度图上的 filter 半径
float radius = w / orthoWidth;
```

然后根据这个半径做 PCF 就能得到最终的结果，这里我同样做 7 x 7 的。虽然阴影还是不够 “软” 但是视觉上比锯齿更好了，并且竹竿的末端是模糊的，这也符合物理规律：

![](https://pic2.zhimg.com/v2-e04bd77b0cbf4a16da936fda6d5977cb_1440w.png)

阴影品质优化
------

通过上面的代码我们很容易得到一个还凑合的效果，但离那种软的像艺术品的阴影还差得远。如果死命增大 lightSize 将会出现条纹：

![](https://picx.zhimg.com/v2-10433e378269d381285e1d4325b7b96b_1440w.jpg)

将遮挡计数 count 作为颜色输出，在 Filter 半径过大、光线方向掠视投影面、或者 Shadow Map 精度不够的时候，会发现第一个 Artifact 就是误遮挡：

![](https://pic1.zhimg.com/v2-094ccb790a3ff055450d43404a947c8e_1440w.jpg)

我们预想的结果是阴影中的像素，count 为 1 因为全遮挡了。半影中的像素有一个过度，而不在阴影中的像素遮挡计数应该为 0 才对。这里发现不在阴影中的像素仍然收到了遮挡，这是由于我们判断的时候没有考虑 Reciever Plane 的深度偏移。如图左边的三个点因为深度大于 Shading Point 而误认为被遮挡：

![](https://pic1.zhimg.com/v2-f1d485e145be58266166b527b874727a_1440w.jpg)

最直接的办法是补偿这个 offset，可以通过在 Shading Point 处，用 LookAt 建立以法向量 N 为 z 轴正方向的坐标变换矩阵 mat\_N 来实现，具体步骤为：

1.  将 Shading Point 变换到阴影相机坐标系
2.  对的 xy 进行偏移，z 则取 Shading Point 原本的深度，反投影得到偏移后的世界坐标
3.  偏移后的世界坐标乘以法向旋转矩阵 mat\_N 转换到法向量坐标
4.  因为 z 轴为 N 方向， 故此时 z 坐标就是补偿量

这套操作下来确实挺抽象的，再加上我的表述能力不是很清晰，大伙能看懂就看，看不懂就当我在这胡言乱语。不过我们在干的事情差不多是这样子：

![](https://pic3.zhimg.com/v2-4816b7d303385216077670c9aed70004_1440w.jpg)

不管从代码编写还是执行效率上，怎么说这套操作都太麻烦了，所以作为懒狗 + 摆子我选择另一种取巧的方案。回到两个半径上来，计算平均遮挡深度时的半径决定了半影区域能有多大，而 PCF 半径决定了阴影有多软

抛开物理上完全正确的 lightSize，我们使用两个额外的变量代替它从而用不同的半径做 Depth Search 和 PCF，他们都是世界坐标上的距离，因此需要结合场景大小来调整。这里我直接用 \[System.Serializable\] 把它暴露出来作为参数：

![](https://pica.zhimg.com/v2-3ae5b5cdd26512a1537c9bba5ea58ff6_1440w.jpg)

用这两个变量替代 lightSize，因为半影区域大小高度依赖我们 Depth Search 时的半径，为了保证不发生上文所讲的自遮蔽，我们人为假定光源比较小。而做 blur 的时候可以认为光源比较大以得到比较模糊的效果：

![](https://pic1.zhimg.com/v2-2f37f29b3a6afcfc9df16e609118529c_1440w.jpg)

再来看看，现在能够很明显的产生 “散开” 的效果了：

![](https://pic1.zhimg.com/v2-f834642e52cc77874e9f57faee4ca5f0_1440w.jpg)

这里 7 x 7（循环次数，非像素个数）的 PCF 效果足够好了，但是大半径的采样也带来了问题，我们需的深度计算和 PCF 两个步骤总共需要 49 + 49 将近 100 次采样太昂贵。可以通过一些特定的抽样序列，比如 Halton、Hammersley、[Poisson Disk](https://zhida.zhihu.com/search?content_id=190879012&content_type=Article&match_order=1&q=Poisson+Disk&zhida_source=entity)、sobol 来达到少量抽样 “四两拨千斤” 的效果。这里直接开摆并偷一个 16 样本的 Poisson Disk 作罢：

![](https://pic3.zhimg.com/v2-c7760fe59aa1492a9f0660d0d6de08e8_1440w.jpg)

```text
#define N_SAMPLE 16
static float2 poissonDisk[16] = {
    float2( -0.94201624, -0.39906216 ),
    float2( 0.94558609, -0.76890725 ),
    float2( -0.094184101, -0.92938870 ),
    float2( 0.34495938, 0.29387760 ),
    float2( -0.91588581, 0.45771432 ),
    float2( -0.81544232, -0.87912464 ),
    float2( -0.38277543, 0.27676845 ),
    float2( 0.97484398, 0.75648379 ),
    float2( 0.44323325, -0.97511554 ),
    float2( 0.53742981, -0.47373420 ),
    float2( -0.26496911, -0.41893023 ),
    float2( 0.79197514, 0.19090188 ),
    float2( -0.24188840, 0.99706507 ),
    float2( -0.81409955, 0.91437590 ),
    float2( 0.19984126, 0.78641367 ),
    float2( 0.14383161, -0.14100790 )
};
```

然后修改一下 PCF 和 Depth Search 的代码，原来用 (i, j) 做偏移量，现在改为 poissonDisk\[i\] 即可。看看效果如何：

![](https://pic3.zhimg.com/v2-8e200f4b5922e3e1c948410f383314ba_1440w.png)

结果非常好，我们用 16 + 16 次采样，也就是 1/3 的性能开销，就达到了 7 x 7 PCF 差不多的效果。但是这样的结果仍然有瑕疵，可以看到阴影 “条带” 分明，这是因为所有的像素都使用相同的样本。我们只需要给每个像素的样本一个随机的旋转即可。随机旋转可以来自一张 [Blue Noise Texture](https://zhida.zhihu.com/search?content_id=190879012&content_type=Article&match_order=1&q=Blue+Noise+Texture&zhida_source=entity) 或者是纯粹的伪随机数：

![](https://pic1.zhimg.com/v2-8679bc097e5d062745849a2cf7111c76_1440w.jpg)

现在（阴影）确实软了，也不分层了，但是新的问题又来了，因为每个像素使用的样本不一样，所以造成了非常 Noisy 的结果（尤其是镜头转动的时候）。解决方案是让 2 x 2 的像素都共用一个随机数以解决转动镜头时候噪点快速闪烁：

![](https://pic2.zhimg.com/v2-00bcab9feb1b1675e71b07c9266e2921_1440w.jpg)

右边是 2 x 2 sub pixel 共用一个随机数的结果

然后将结果输出到一张纹理再模糊。因为噪点不是很严重甚至不需要上 Temporal Filter（其实是我懒得弄）。此外这儿用了个小技巧将 7 x 7 的模糊拆解为横竖两次 7 + 7 的模糊，需要用 Blit 配合 Shader 输出到两个临时的纹理，最后输出到 shadowStrength 目标纹理：

```csharp
// 阴影计算 pass : 输出阴影强度 texture
void ShadowMappingPass(ScriptableRenderContext context, Camera camera)
{
    CommandBuffer cmd = new CommandBuffer();
    cmd.name = "shadowmappingpass";

    RenderTexture tempTex1 = RenderTexture.GetTemporary(Screen.width, Screen.height, 0, RenderTextureFormat.R8, RenderTextureReadWrite.Linear);
    RenderTexture tempTex2 = RenderTexture.GetTemporary(Screen.width, Screen.height, 0, RenderTextureFormat.R8, RenderTextureReadWrite.Linear);

    // 生成阴影
    cmd.Blit(gbufferID[0], tempTex1, new Material(Shader.Find("ToyRP/shadowmappingpass")));

    // 横向模糊
    cmd.Blit(tempTex1, tempTex2, new Material(Shader.Find("ToyRP/blurNx1")));

    // 纵向模糊
    cmd.Blit(tempTex2, shadowStrength, new Material(Shader.Find("ToyRP/blur1xN")));

    RenderTexture.ReleaseTemporary(tempTex1);
    RenderTexture.ReleaseTemporary(tempTex2);

    context.ExecuteCommandBuffer(cmd);
    context.Submit();
}

```

从左到右分别是原图，横向模糊和横 + 纵的结果：

![](https://pic4.zhimg.com/v2-0213657b8bccb2face5c0d57f0867677_1440w.jpg)

![](https://pica.zhimg.com/v2-aabe232e736394e2287d2f779bc527c2_1440w.jpg)

最后叠加到原画面，现在我们得到了一个还不错的结果：

![](https://pic1.zhimg.com/v2-c16f3460102df2dbf99c23288549da28_1440w.jpg)

模糊之后新的 Artifact 又出现辣，这是因为背面有些漏光且模糊会使得被模糊对象的范围扩大一圈。当物体和阴影有 Overlap 的时候就会像被光灵箭射中一样，有个亮色的描边：

![](https://picx.zhimg.com/v2-5172700c9a717b53c785fd83b1c36a67_1440w.jpg)

要想解决这个问题我们得用带权的高斯模糊，而不是无脑的叠加然后除以采样次数。不难想到这里滤波盒权重可以考虑两个因素：worldPos 和 normal

事实上直接用两次采样之间 worldPos 距离反比做权重就能很好解决这个问题。横向模糊 pass 的代码大致如下：

![](https://pic4.zhimg.com/v2-7029ca0a32ada8a670e568e33cb786bd_1440w.jpg)

搞定：

![](https://pica.zhimg.com/v2-bb79b47560b81553f144068538d2647c_1440w.jpg)

使用半径过大的模糊又会带来新的问题，阴影近遮挡物的地方也被软化了，而这部分本来应该是硬阴影：

![](https://pic2.zhimg.com/v2-090c0b048155eddecbdb597dfcc3fc8b_1440w.png)

模糊半径小了又噪点，大了又效果不好，我们采样两边折中各退一步的方式。首先模糊的半径减少一点，其次我们还要想办法让噪点也减少。最直接的办法是增大计算平均深度和做 PCF 的时候多用几个样本，我们上面才用了 16 + 16 个 Poisson Disk 样本，可以尝试 64 个样本以减少噪声。下图是没没有降噪就直接输出的 PCSS 结果：

![](https://pic3.zhimg.com/v2-5dfc9862c3c8106d8380057b821933ec_1440w.jpg)

然后再上个 3x3 的带 worldPos 权重的模糊，现在终于顺滑无噪声了，阴影的根部也不会软掉了：

![](https://pic1.zhimg.com/v2-52e2079108dc6de0ff33806855caf6fe_1440w.jpg)

阴影性能优化
------

没有免费午餐。高品质必然伴随着低帧率，在上面部分我搁这典型的拆东墙补西墙。为了软化阴影同时提升效率而引入了 Poisson Disk 和随机数，为了 Denoise 又引入了模糊，为了干掉模糊又选择了加采样次数减模糊半径的折中策略，结果开销不降反升。先不谈我被自己的铸币操作气笑了

![](https://pica.zhimg.com/v2-368b58206cd9f7365a021f47242dd674_1440w.jpg)

于是痛定思痛引入一个终极优化：因为 PCSS 非常昂贵，那些在半影之中的点才需要用 PCSS，不难想到通过两步来完成。第一个 Pass 做 Pre ShadowMapping 生成一张 Mask 标记半影区域，第二个 Pass 只对半影内的像素做 PCSS 以节省开销。对于其他的点则直接返回 0 或者 1

因为二值阴影映射只能标记阴影和非阴影，我们还需要想办法生成半影。这里直接抄某原的算法，将原画面 4 x 4 个像素的区域一同计算阴影然后 Down Sample 到 1/4 分辨率的 Mask 贴图：

![](https://pica.zhimg.com/v2-3c9798d5c1500df8be304275fdb5df3a_1440w.jpg)

![](https://pic4.zhimg.com/v2-7fcc3f15038aea8d65ce313fe131ceed_1440w.jpg)

我们也如法炮制来一个，这里和 ppt 稍微有点不同，我老老实实（偷懒）地对 4 x 4 区域的每个像素都做了采样。这里注意一下分辨率和采样坐标的转换就没问题，虽然我们在 1/4 分辨率下计算，但是采样 Gbuffer 的时候还得按原分辨率：

![](https://picx.zhimg.com/v2-95d973ee5a92659c58a91fb74951afc3_1440w.jpg)

此外这里直接平均所有的样本而不是使用 worldPos 做权重去筛那些在 Screen Space 很接近但是实际上相差很远的点，所以球体的边缘也会被标记：

![](https://pica.zhimg.com/v2-a9692c0bd0cdbd0184635aec5b9c5b72_1440w.jpg)

对于大多数情况（比如大世界的太阳平行光）来说这张 Mask 足够使用了，但是因为 Down Sample 和 Blur 都在屏幕空间进行，所以当镜头离的很近的时候 Mask 范围会逐渐变小从而有悖于真实的半影大小：

![](https://pic3.zhimg.com/v2-1a78d42209b59550269e527cb70b1f68_b.gif)

如果不追求那种像核弹爆炸现场一般的超大范围软阴影到这里就可以作罢手工开摆了，但是这里也可以加个奇技淫巧。加个系数使得距离越近 Blur 的半径越大就能比较好的平衡这个异常情况。这里我用的平方反比，这些魔数也要结合实际场景大小调整。同样拆开用横纵两个 Pass 对 Mask 进行滤波：

![](https://pic1.zhimg.com/v2-bd8479d3d780e644983adb327adf3900_1440w.jpg)

调整过后的效果，放大之后仍然保留极大的半影区域，阴影支持很软：

![](https://pic3.zhimg.com/v2-950b20e58db47c2ef5e6dc4403afc878_b.gif)

这里半径函数的选择也要结合阴影软硬程度考虑，总之就是嗯调这些参数。因为太过膜法，再加上在我的 3060 机器上采样 64 + 64 次 Poisson Disk 并非登天难事，所以我在 commit 到 Github 上的时候默认关闭了整套 Mask + Blur + ShadowMapping + Blur 的操作。可以通过 Pipeline Asset 的检视界面打开：

![](https://pica.zhimg.com/v2-cef5a87198d3f6b6cceed19fb0994bb2_1440w.jpg)

截至目前完整的阴影管线如下，水平竖直两个 Pass 合起来是 1/4 分辨率下 7 x 7 的模糊，而 PCSS 之后直接全分辨率 3 x 3 一个 Pass 搞定。加上 Mask 生成时采样 4 x 4 区域，最终对于每个屏幕像素的额外开销为 (7 + 7 + 16)/16 + 9 = 11 次左右，总的来说还行：

![](https://picx.zhimg.com/v2-36301f2606cc65d559d33cd2fe0bc795_1440w.jpg)

总结
--

经历了相当多的设置，编写了又臭又长的代码，总算是做出了一个稍微没那么辣眼睛的软阴影。好吧，我承认我代码写的是一坨屎，我做出来的效果是一坨屎，我代码的运行效率也是一坨屎，我的人生也是一坨屎，我这辈子也就这鸟样了，开摆！

![](https://pica.zhimg.com/v2-4538574496fd7fa0dade1dbf49f88162_1440w.jpg)

另：“震惊！在光栅管线下折腾半天的软阴影，在光线追踪管线下软阴影和 AO 竟是白送的”

参考与引用
-----

\[1\] opengl-tutorial, ["Tutorial 16 : Shadow mapping"](https://link.zhihu.com/?target=http%3A//www.opengl-tutorial.org/cn/intermediate-tutorials/tutorial-16-shadow-mapping/)

\[2\] NVIDIA, ["Percentage-Closer Soft Shadows"](https://link.zhihu.com/?target=https%3A//developer.download.nvidia.cn/shaderlibrary/docs/shadow_PCSS.pdf)

\[3\] Vilem Otte, ["Effect: Area Light Shadows Part 1: PCSS"](https://link.zhihu.com/?target=https%3A//www.gamedev.net/tutorials/programming/graphics/effect-area-light-shadows-part-1-pcss-r4971/)

\[4\] kakaroto, ["实时渲染｜Shadow Map：PCF、PCSS、VSM、MSM"](https://zhuanlan.zhihu.com/p/369710758)

\[5\] John R. Isidoro, ["Shadow Mapping: GPU-based Tips and Techniques"](https://link.zhihu.com/?target=http%3A//amd-dev.wpengine.netdna-cdn.com/wordpress/media/2012/10/Isidoro-ShadowMapping.pdf)

\[6\] Kevin Myers, ["Integrating Realistic Soft Shadows into Your Game Engine"](https://link.zhihu.com/?target=https%3A//developer.download.nvidia.cn/whitepapers/2008/PCSS_Integration.pdf)

\[7\] TheMasonX, ["UnityPCSS"](https://link.zhihu.com/?target=https%3A//github.com/TheMasonX/UnityPCSS)

\[8\] xiaOp, ["实时阴影技术总结"](https://link.zhihu.com/?target=https%3A//xiaoiver.github.io/coding/2018/09/27/%25E5%25AE%259E%25E6%2597%25B6%25E9%2598%25B4%25E5%25BD%25B1%25E6%258A%2580%25E6%259C%25AF%25E6%2580%25BB%25E7%25BB%2593.html)

\[9\] 闫令琪老师, [“GAMES202-高质量实时渲染”](https://link.zhihu.com/?target=https%3A//www.bilibili.com/video/BV1YK4y1T7yY)
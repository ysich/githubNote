# 基础且直白的Unity渲染-利用CommandBuffer制作一些有趣的效果。 - 知乎
ConmmandBuffer是Unity提供的命令缓存区。用以储存渲染绘制命令。commandbuffer设置渲染目标或用来绘制给定网格。可以设置在摄像机上不同的点执行。因此其拓展性可自由性非常的强大。

简单来说就是一个可以根据我们需要进行相当程度自定义话处理的一个命令缓冲区。可以利用它做出非常多有趣的操作。

制作一个最简单的CMB渲染脚本：
----------------

从最简单的脚本开始制作可以方便我们理清思路：

```csharp
using UnityEditor;
using UnityEngine;
using System;

public class CMBRenderTest : MonoBehaviour
{
    private MeshRenderer render;
    private Material material;

    private CommandBuffer cmdBuffer;

    private void Awake()
    {
        render = GetComponent<MeshRenderer>();//获得渲染组件。
        material = new Material(Shader.Find("Unlit/TestUiltShader"));
        material.SetColor("_MainColor", Color.green);
    }

    private void OnEnable()
    {
        cmdBuffer = new CommandBuffer();//新建命令
        cmdBuffer.DrawRenderer(render, material);//执行buffer绘制。材质球输入render；
        Camera.main.AddCommandBuffer(CameraEvent.AfterImageEffects, cmdBuffer);//在渲染中插入需要在特定阶段执行的CommandBuffer            
    }

    private void OnDisable()//注销命令时
    {
        if(Camera.main != null)
        {
            Camera.main.RemoveCommandBuffer(CameraEvent.AfterImageEffects, cmdBuffer);//移除Commandbuffer数据。
        }
    }
}




```

![](https://pic2.zhimg.com/v2-f5afb36404dfb7a702db1e76c5e8b05d_1440w.jpg)

可以发现CMB帮助我们快速获得了一个自定义的层并将Cube的Rnderer有重新渲染了一次。自然的Pass和DrawCall也多了一次。

利用CMD来进行特定的效果渲染：
----------------

CMD可以帮助我们拓展Unity的默认管线。控制渲染的量级和精度。也就是说可以在原先的模型的渲染上增加更多有趣的效果。且只会对某一个[gameobject](https://zhida.zhihu.com/search?content_id=212498843&content_type=Article&match_order=1&q=gameobject&zhida_source=entity)或者UI进行变化。而且还可以精确地控制到在哪一层进行渲染。非常的方便。**只是对于Shader初学者来说学习成本会比较高,因为涉及到了API的运用。而且还是需要进行打包测试来看最后的具体消耗。** 

### **利用CMD制作一个类似[战争迷雾](https://zhida.zhihu.com/search?content_id=212498843&content_type=Article&match_order=1&q=%E6%88%98%E4%BA%89%E8%BF%B7%E9%9B%BE&zhida_source=entity)的效果：** 

![](https://picx.zhimg.com/v2-c2a00e7c02581e58c9f317daf77c2717_1440w.jpg)

![](https://picx.zhimg.com/v2-28c670948c2a9ee40b60dab80bcd3625_1440w.jpg)

首先我们要明确用什么方法去让雾气遮挡住距离玩家过远的范围。我的办法是利用CMD渲染一整个画面的雾气。然后再利用一个面片进行裁剪再利用CMD的原理将这一块儿的雾气剔除掉。

首先书写一个全屏的雾气：

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Rendering;

[ExecuteInEditMode]
public class CameraWarFog : MonoBehaviour
{
    public Camera camera;
    public Material mat;
    public Color color;

    //CMD设置。
    private CommandBuffer commandBuffer;
    public Material material;//cmb用的材质；
    public MeshRenderer meshRenderer;//cmb需要用到的meshrenderer；

    private void Start()
    {
        
        CMD();

    }


    RenderTexture source; //设置RT
    RenderTexture destination;
    private void OnPreRender()
    {
        source = RenderTexture.GetTemporary(Screen.width, Screen.height, 16); //定义一下RT的长宽和位数
        camera.targetTexture = source;//给予摄像机这个RT；

    }

    private void OnPostRender()
    {
        camera.targetTexture = null; //将摄像机的RT清空；
        mat.SetTexture("_MainTex", source);
        mat.SetColor("_MainColor", color);
        Graphics.Blit(source, destination, mat, 0);//渲染这个材质；

        RenderTexture.ReleaseTemporary(source);//清理缓存；
        RenderTexture.ReleaseTemporary(destination);
    }

    void CMD()
    {
        commandBuffer = new CommandBuffer();
        commandBuffer.name = "Mask";//定义一下cmb的名字

        commandBuffer.DrawRenderer(meshRenderer, material);//绘制这个cmb到指定的mesh上；
        camera.AddCommandBuffer(CameraEvent.AfterForwardOpaque, commandBuffer);//将他加入到摄像机的指定渲染层


        Debug.Log("AddCMD");
    }


}

```

我们用来做裁剪的模型本身还是要给一个空材质。不然会变紫色。

![](https://pic4.zhimg.com/v2-cca20e5dd617925d4e39054f1c24974f_1440w.jpg)

![](https://picx.zhimg.com/v2-ead5815ac5096b704781427d089fa541_1440w.jpg)

```text
Shader "WarFog/Null"
{
	Properties
	{
		
	}
	
	SubShader
	{
		
		
		Tags { "RenderType"="Opaque" }

		CGINCLUDE
		#pragma target 3.0
		ENDCG
		Blend One One
		AlphaToMask Off
		Cull Back
		ColorMask 0
		ZWrite Off
		ZTest LEqual
		BlendOp min
		
		
		
		Pass
		{
			Name "Unlit"
			Tags { "LightMode"="ForwardBase" }
			CGPROGRAM

			

			#ifndef UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX
			//only defining to not throw compilation error over Unity 5.5
			#define UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX(input)
			#endif
			#pragma vertex vert
			#pragma fragment frag
			#pragma multi_compile_instancing
			#include "UnityCG.cginc"
			

			struct appdata
			{
				float4 vertex : POSITION;
				float4 color : COLOR;
				
				UNITY_VERTEX_INPUT_INSTANCE_ID
			};
			
			struct v2f
			{
				float4 vertex : SV_POSITION;
				float3 worldPos : TEXCOORD0;
				
				UNITY_VERTEX_INPUT_INSTANCE_ID
				UNITY_VERTEX_OUTPUT_STEREO
			};

			
			
			v2f vert ( appdata v )
			{
				v2f o;
				UNITY_SETUP_INSTANCE_ID(v);
				UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);
				UNITY_TRANSFER_INSTANCE_ID(v, o);
				
				o.vertex = UnityObjectToClipPos(v.vertex);


				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;

				return o;
			}
			
			fixed4 frag (v2f i ) : SV_Target
			{
				UNITY_SETUP_INSTANCE_ID(i);
				UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX(i);
				fixed4 finalColor;

				float3 WorldPosition = i.worldPos;

				float4 color1 = IsGammaSpace() ? float4(1,1,1,1) : float4(1,1,1,1);
				float4 appendResult4 = (float4((color1).rgb , color1.a));
				
				
				finalColor = appendResult4;
				return finalColor;
			}
			ENDCG
		}
	}
	
	
}
```

雾气的[shader](https://zhida.zhihu.com/search?content_id=212498843&content_type=Article&match_order=1&q=shader&zhida_source=entity)就更简单了:

```text
Shader "WarFog/TestFog"
{
	Properties
	{
		_MainColor("MainColor", Color) = (1,1,1,1)
		_MainTex("MainTex", 2D) = "white" {}
		[HideInInspector] _texcoord( "", 2D ) = "white" {}

	}
	
	SubShader
	{
		
		
		Tags { "RenderType"="Opaque" }
	LOD 0

		CGINCLUDE
		#pragma target 3.0
		ENDCG
		Blend Off
		AlphaToMask Off
		Cull Back
		ColorMask RGBA
		ZWrite Off
		ZTest LEqual
		
		
		
		Pass
		{
			Name "Unlit"
			Tags { "LightMode"="ForwardBase" }
			CGPROGRAM

			

			#ifndef UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX
			//only defining to not throw compilation error over Unity 5.5
			#define UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX(input)
			#endif
			#pragma vertex vert
			#pragma fragment frag
			#pragma multi_compile_instancing
			#include "UnityCG.cginc"
			

			struct appdata
			{
				float4 vertex : POSITION;
				float4 color : COLOR;
				float4 ase_texcoord : TEXCOORD0;
				UNITY_VERTEX_INPUT_INSTANCE_ID
			};
			
			struct v2f
			{
				float4 vertex : SV_POSITION;

				float3 worldPos : TEXCOORD0;

				float4 ase_texcoord1 : TEXCOORD1;
				UNITY_VERTEX_INPUT_INSTANCE_ID
				UNITY_VERTEX_OUTPUT_STEREO
			};

			uniform float4 _MainColor;
			uniform sampler2D _MainTex;
			uniform float4 _MainTex_ST;

			
			v2f vert ( appdata v )
			{
				v2f o;
				UNITY_SETUP_INSTANCE_ID(v);
				UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);
				UNITY_TRANSFER_INSTANCE_ID(v, o);

				o.ase_texcoord1.xy = v.ase_texcoord.xy;
				
				//setting value to unused interpolator channels and avoid initialization warnings
				o.ase_texcoord1.zw = 0;

				o.vertex = UnityObjectToClipPos(v.vertex);


				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;

				return o;
			}
			
			fixed4 frag (v2f i ) : SV_Target
			{
				UNITY_SETUP_INSTANCE_ID(i);
				UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX(i);
				fixed4 finalColor;

				float3 WorldPosition = i.worldPos;

				float2 uv_MainTex = i.ase_texcoord1.xy * _MainTex_ST.xy + _MainTex_ST.zw;
				float4 tex2DNode5 = tex2D( _MainTex, uv_MainTex );
				float3 lerpResult10 = lerp( (_MainColor).rgb , (tex2DNode5).rgb , ( 1.0 - ( tex2DNode5.a * _MainColor.a ) ));
				
				
				finalColor = float4( lerpResult10 , 0.0 );
				return finalColor;
			}
			ENDCG
		}
	}

}

```

接下来书写一个CMB用来进行裁剪用的shader

```text
Shader "WarFog/CubeAdd"
{
	Properties
	{
		_Color("Color", Color) = (1,1,1,0)
		_TextureSample0("Texture Sample 0", 2D) = "white" {}
		[HideInInspector] _texcoord( "", 2D ) = "white" {}

	}
	
	SubShader
	{
		
		
		Tags { "RenderType"="Opaque" }
	LOD 100

		CGINCLUDE
		#pragma target 3.0
		ENDCG
		Blend One One
		AlphaToMask Off
		Cull Off
		ColorMask A
		ZWrite Off
		ZTest LEqual
		BlendOp min
		
		
		
		Pass
		{
			Name "Unlit"
			Tags { "LightMode"="ForwardBase" }
			CGPROGRAM

			

			#ifndef UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX
			//only defining to not throw compilation error over Unity 5.5
			#define UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX(input)
			#endif
			#pragma vertex vert
			#pragma fragment frag
			#pragma multi_compile_instancing
			#include "UnityCG.cginc"
			

			struct appdata
			{
				float4 vertex : POSITION;
				float4 color : COLOR;
				float4 ase_texcoord : TEXCOORD0;
				UNITY_VERTEX_INPUT_INSTANCE_ID
			};
			
			struct v2f
			{
				float4 vertex : SV_POSITION;
				float3 worldPos : TEXCOORD0;
				float4 ase_texcoord1 : TEXCOORD1;
				UNITY_VERTEX_INPUT_INSTANCE_ID
				UNITY_VERTEX_OUTPUT_STEREO
			};

			uniform float4 _Color;
			uniform sampler2D _TextureSample0;
			uniform float4 _TextureSample0_ST;

			
			v2f vert ( appdata v )
			{
				v2f o;
				UNITY_SETUP_INSTANCE_ID(v);
				UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);
				UNITY_TRANSFER_INSTANCE_ID(v, o);

				o.ase_texcoord1.xy = v.ase_texcoord.xy;
				
				//setting value to unused interpolator channels and avoid initialization warnings
				o.ase_texcoord1.zw = 0;
				o.vertex = UnityObjectToClipPos(v.vertex);

				o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;

				return o;
			}
			
			fixed4 frag (v2f i ) : SV_Target
			{
				UNITY_SETUP_INSTANCE_ID(i);
				UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX(i);
				fixed4 finalColor;

				float3 WorldPosition = i.worldPos;
				float2 uv_TextureSample0 = i.ase_texcoord1.xy * _TextureSample0_ST.xy + _TextureSample0_ST.zw;
				float4 appendResult3 = (float4((_Color).rgb , ( _Color.a * tex2D( _TextureSample0, uv_TextureSample0 ).r )));
				
				
				finalColor = appendResult3;
				return finalColor;
			}
			ENDCG
		}
	}
	
	
}

```

其实最终就是利用了一个面片就帮助我们完成了简单战争迷雾效果。当然还有更复杂的效果

![](https://pic1.zhimg.com/v2-32735beb11bab24278522841eeac9084_1440w.jpg)

### 录屏：

![](https://pic1.zhimg.com/v2-d6ace1bb0e75be4853e2a6aa39ef8be8.jpg?source=25ab7b06)

00:25

记录日期：
-----

2022/11/24
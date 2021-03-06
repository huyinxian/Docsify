# 简单Shader的编写

从这一篇笔记开始我们将正式的编写 UnityShader 文件，实现一些简单的效果。

## 最基本的顶点/片元着色器

---

先来给演示一下最简单的顶点/片元着色器怎么写。首先创建一个 ShaderLab 文件，编写代码：

```
Shader "Custom/SimpleShader"
{
	SubShader
	{
		Pass
        {
            CGPROGRAM

            #pragma vertex vert
            #pragma fragment frag

            float4 vert(float4 v : POSITION) : SV_POSITION
            {
                return mul(UNITY_MATRIX_MVP, v);
            }

            fixed4 frag() : SV_TARGET
            {
                return fixed4(1.0, 1.0, 1.0, 1.0);
            }

            ENDCG
        }
	}
}
```

新建一个材质，把写好的 Shader 赋给它。然后在场景中新建一个球体，将球体的材质改为我们刚刚弄好的材质。效果如下：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/%E7%AE%80%E5%8D%95Shader%E7%9A%84%E7%BC%96%E5%86%99/01.png)

效果虽然简单，但这确实是我们编写的第一个顶点/片元着色器。代码中我们只写了一个 SubShader，因为我们可以不选择声明任何材质属性。由于我们没有设置渲染状态和标签，因此 SubShader 将使用默认设置渲染。接下来最重要的就是 CGPROGRAM 和 ENDCG 中的 Cg 代码了。

我们首先写了两条预编译指令：

```
#pragma vertex vert
#pragma fragment frag
```

这两条代码的作用是告诉 Unity 哪个函数包含了顶点着色器代码，哪个函数包含了片元着色器代码。通用的代码格式如下：

```
#pragma vertex name1
#pragma fragment name2
```

下面来看一下顶点着色器代码：

```
float4 vert(float4 v : POSITION) : SV_POSITION
{
    return mul(UNITY_MATRIX_MVP, v);
}
```

`POSITION` 和 `SV_POSITION` 是 Cg/HLSL 代码的语义，它们将规定系统用户需要输入哪些值，以及用户的输出是什么。上述代码中，POSITION 表示需要把模型的顶点坐标传进来，SV_POSITION 表示方法会返回裁剪空间中的顶点坐标。这些语义是不能够省略的，否则渲染器无法得知输入参数和输出值。

?> 注意，高版本的 Unity 会把 `mul(UNITY_MATRIX_MVP,*)` 自动替换为 `UnityObjectToClipPos(*)`。

再来看一下片元着色器的代码：

```
fixed4 frag() : SV_TARGET
{
    return fixed4(1.0, 1.0, 1.0, 1.0);
}
```

frag 函数没有任何输入，只有一个 fixed4 类型的返回值。`SV_TARGET` 是 HLSL 中的语义，表示将用户的输出颜色存储到一个渲染目标中。这里的颜色范围是 [0, 1]，`(1,1,1)` 代表白色（其实就是RGBA）。

## 获取更多的模型数据

---

上面的例子中我们获得了模型的顶点位置，但假如我们想要获取更多的数据怎么办？比如我们需要获取模型上每个顶点的纹理坐标和法线方向，这个时候就要用到结构体：

```
Shader "Custom/SimpleShader"
{
	SubShader
	{
		Pass
        {
            CGPROGRAM

            #pragma vertex vert         // 指明哪个函数包含了顶点着色器代码
            #pragma fragment frag       // 指明哪个函数包含了片元着色器代码

            struct a2v
            {
                float4 vertex : POSITION;       // 模型空间的顶点坐标
                float3 normal : NORMAL;         // 模型空间的法线方向
                float4 texcoord : TEXCOORD;     // 模型的第一套纹理坐标
            };

            struct v2f
            {
                float4 pos : SV_POSITION;       // 裁剪空间的位置信息
                fixed3 color : COLOR0;          // 颜色信息
            };

            v2f vert(a2v v)
            {
                v2f o;
                // 这里的mul(UNITY_MATRIX_MVP,*)被替换为了UnityObjectToClipPos(*)
                o.pos = UnityObjectToClipPos(v.vertex);
                // v.normal包含了顶点的法线方向，范围在[-1,1]
                // 下面的代码把范围映射到了[0,1]
                o.color = v.normal * 0.5 + fixed3(0.5, 0.5, 0.5);
                return o;
            }

            fixed4 frag(v2f i) : SV_TARGET
            {
                // 将插值后的i.color输出到屏幕上
                return fixed4(i.color, 1.0);
            }

            ENDCG
        }
	}
}
```

这样我们就得到了一个彩色的小球：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/%E7%AE%80%E5%8D%95Shader%E7%9A%84%E7%BC%96%E5%86%99/02.png)

我们先来看一下结构体的语法：

```
// 注意，语义是不能够省略的
struct StructName
{
    Type Name : Semantic;
    Type Name : Semantic;
    ...
}
```

我们首先为 vert 函数定义了一个结构体 a2v，这个结构体会接受模型上的每个顶点坐标、法线方向、纹理坐标。这些数据都是由材质的 Mesh Renderer 组件提供的，在每帧调用 DrawCall 时，Mesh Renderer 就会把它负责渲染的模型数据发送给 UnityShader。一个模型通常包含多个三角面，一个三角面由三个顶点构成，而每个顶点又包括顶点坐标、法线、切线、纹理、颜色等数据。通过上述方法，我们就可以在顶点着色器中访问这些数据。

接下来就是顶点着色器和片元着色器之间的通信了。我们新定义了一个结构体 v2f，用于在顶点着色器和片元着色器之间传递信息。顶点着色器的输出中必须要包含一个变量，它的语义应该为 SV_POSITION，否则片元着色器将无法得到裁剪空间中的顶点坐标，也就无法把顶点渲染到屏幕上。`COLOR0` 中的颜色可以自定义，类似的还有 `COLOR1` 等等。

着色器之间的简单通信就大致如此了。需要注意的是，顶点着色器是逐顶点调用的，而片元着色器是逐片元调用的。片元着色器的输入实际上是把顶点着色器的输出进行插值后得到的结果，因此小球才会是彩色。

## 添加属性

---

我们其实可以为 Shader 设置一些属性，这样能够方便开发者调节 Shader 的显示效果。比如我可以声明一个 Color 属性：

```
Shader "Custom/SimpleShader"
{
    Properties
    {
        _Color ("Color Tint", Color) = (1.0, 1.0, 1.0, 1.0)
    }

	SubShader
	{
		Pass
        {
            CGPROGRAM

            #pragma vertex vert         // 指明哪个函数包含了顶点着色器代码
            #pragma fragment frag       // 指明哪个函数包含了片元着色器代码

            // 在Cg代码中，我们需要定义一个与属性一样的变量才能使用属性
            fixed4 _Color;

            struct a2v
            {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                float4 texcoord : TEXCOORD;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
                fixed3 color : COLOR0;
            };

            v2f vert(a2v v)
            {
                v2f o;
                // 这里的mul(UNITY_MATRIX_MVP,*)被替换为了UnityObjectToClipPos(*)
                o.pos = UnityObjectToClipPos(v.vertex);
                // v.normal包含了顶点的法线方向，范围在[-1,1]
                // 下面的代码把范围映射到了[0,1]
                o.color = v.normal * 0.5 + fixed3(0.5, 0.5, 0.5);
                return o;
            }

            fixed4 frag(v2f i) : SV_TARGET
            {
                // 设置RGB颜色和Alpha透明度
                fixed3 c = i.color;
                // 使用_Color属性控制输出的颜色
                c *= _Color.rgb;
                return fixed4(c, 1.0);
            }

            ENDCG
        }
	}
}

```

注意，声明的属性是无法直接在 Cg 代码中访问的，我们需要在 Cg 代码中提前定义一个名称、类型与属性一样的变量。ShaderLab 中的属性类型和 Cg 变量类型的匹配关系如下：

| <center>ShaderLab属性类型</center> | <center>Cg变量类型</center> |
| :--- | :--- |
| Color，Vector | float4，half4，fixed4 |
| Range，Float | float，half，fixed |
| 2D | sampler2D |
| Cube | samplerCube |
| 3D | sampler3D |

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/%E7%AE%80%E5%8D%95Shader%E7%9A%84%E7%BC%96%E5%86%99/03.png)

这样一来我们就可以直接在材质面板中更改颜色。

## Unity内置文件和变量

---

Unity 提供了许多内置文件，里面包含了函数、变量、宏等，可以帮助我们快速编码。这种文件类似于 C++ 头文件，在 Untiy 中它们的后缀为 `.cginc`。使用方式很简单：

```
CGPROGRAM
#include "UnityCG.cginc"
ENDCG
```

在 Windows 上，这些文件的路径是：Unity的安装路径/Data/CGIncludes。

下面给出几个主要的文件以及其用处：

| <center>文件名</center> | <center>描述</center> |
| :--- | :--- |
| UnityCG | 包含了最常用的帮助的函数、宏、结构体等 |
| UnityShaderVariables | 编译 UnityShader 时会自动包含进来，内置了许多全局变量，比如 UNITY_MATRIX_MVP |
| Lighting | 包含了各种光照模型，编写表面着色器时会自动包含进来 |
| HLSLSupport | 编译 UnityShader 时会自动包含进来，内置了许多跨平台编译的宏和定义 |

下面给出一些 `UnityCG.cginc` 中常用的结构体和变量：

| <center>名称</center> | <center>描述</center> | <center>包含的变量</center> |
| :--- | :--- | :--- |
| appdata_base | 顶点着色器的输入 | 顶点位置、法线、第一组纹理坐标 |
| appdata_tan | 顶点着色器的输入 | 顶点位置、切线、法线、第一组纹理坐标 |
| appdata_full | 顶点着色器的输入 | 顶点位置、切线、法线、四组（或更多）纹理坐标 |
| appdata_img | 顶点着色器的输入 | 顶点位置、第一组纹理坐标 |
| v2f_img | 顶点着色器的输出 | 裁剪空间中的位置、纹理坐标 |

| <center>函数名</center> | <center>描述</center> |
| :--- | :--- |
| float3 WorldSpaceViewDir(float4 v) | 输入一个模型空间的顶点位置，返回世界空间中从该点到摄像机的观察方向 |
| float3 ObjSpaceViewDir(float4 v) | 输入一个模型空间的顶点位置，返回模型空间中从该点到摄像机的观察方向 |
| float3 WorldSpaceLightDir(float4 v) | 仅用于前向渲染。输入一个模型空间的顶点位置，返回世界空间中从该点到光源的光照方向。未进行归一化 |
| float3 ObjSpaceLightDir(float4 v) | 仅用于前向渲染。输入一个模型空间的顶点位置，返回模型空间中从该点到光源的光照方向。未进行归一化 |
| float3 UnityObjectToWorldNormal(float3 norm) | 把法线方向从模型空间转换到世界空间 |
| float3 UnityObjectToWorldDir(float3 dir) | 把方向矢量从模型空间变换到世界空间 |
| float3 UnityWorldToObjectDir(float3 dir) | 把方向矢量从世界空间变换到模型空间 |

## Unity提供的Cg/HLSL语义

---

我们在之前的代码编写中用到了不少的语义，比如我们需要在输入输出变量的后面跟上一个 `SV_POSITION` 或者别的名称。这些大写的名称实际上是 Cg/HLSL 提供的语义，如果你之前有了解过 DirectX 的话就不会感到陌生。语义的作用是表达它所修饰的参数的含义，让 Shader 知道从哪里读取数据，并把数据输出到哪里。

下面将展示 Unity 中常用的语义。首先是应用阶段传给顶点着色器常用的语义：

| <center>语义</center> | <center>描述</center> |
| :--- | :--- |
| POSITION | 模型空间中的定点位置，通常为 float4 |
| NORMAL | 顶点法线，通常为 float3 |
| TANGENT | 顶点切线，通常为 float4 |
| TEXCOORDn，例如 TEXCOORD0 | 该顶点的纹理坐标，TEXCOORD0 表示第一组纹理坐标。通常为 float2 或者 float4 |
| COLOR | 顶点颜色，通常为 fixed4 或者 float4 |

TEXCOORDn 的数目与 ShaderModel 有关，版本越高 n 越大，具体数值需要看对应版本的文档。

接下来是顶点着色器到片元着色器的语义：

| <center>语义</center> | <center>描述</center> |
| :--- | :--- |
| SV_POSITION | 裁剪空间中的顶点坐标，结构体中必须要包含一个用该语义修饰的变量。等同于 DirectX9 中的 POSITION |
| COLOR0 | 用于输出第一组顶点颜色 |
| COLOR1 | 用于输出第二组顶点颜色 |
| TEXCOORD0~TEXCOORD7 | 用于输出纹理坐标 |

除了 SV_POSITION 具有特殊的含义之外，用其它语义修饰的变量可以存储任意值。例如当我们需要把一些数据从顶点着色器传到片元着色器时，可以用 TEXCOORD0 来存储。

| <center>语义</center> | <center>描述</center> |
| :--- | :--- |
| SV_TARGET | 将数据输出到渲染目标中。等同于 DirectX9 中的 COLOR |

?> 注意，在 DirectX10 中出现了以 `SV_` 开头的语义，用于替代旧版本的某些语义。我们在编写代码时最好使用最新的语义，否则有可能会出现平台不兼容的问题。
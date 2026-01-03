# Love2D光照系统的实现

*实现光照系统有多种不同的方法，这里仅是其中一种方法。

### 光照原理
真实世界中，点光源的亮度遵循以下衰减规律：
$$
    I = \frac{I_0}{d^2}
$$

但是由于计算机中，d趋近于0时，$I_0$会去趋近于无穷大，所以会出现过曝现象。为了解决这种现象，就需要引入限制变量，防止亮度过高。

不过这里并不完全使用上述方案，而是略微有所不同：

$$
L(d)=
\begin{cases}
1, & d \le r_i \\\\\\

A(t), & r_i < d < r_o \\\\\\

0, & d \ge r_o
\end{cases}
$$
$$
A(t)=\frac{(1-t^2)^2}{1+t^3}
$$
$$
t=\frac{d - r_i}{r_o - r_i}
$$

- $d$  ：像素到光源的距离  
- $r_i$：光源内半径  
- $r_o$：光源外半径
- $t$  : 归一化距离

如果先前了解过相关知识的话，就会发现并没有提到内半径外半径之分，那这是怎样一回事呢？

不过若 $r_i=0$ 时，$t=\frac{d}{r}$，就和其他方案中计算归一化距离的方式一样了。

题主并非一开始就使用的是上述公式，而是使用 smoothstep( ) 的方式去处理亮度衰减。

glsl语言中，smoothstep(a,b,c)的用法是：
$$
F(c)=
\begin{cases}
0, c \le a\\
S(c), a<c<b\\
1, c \ge b
\end{cases}
$$

根据输入的值，输出一条平滑的曲线 $S(c)$。与之对应的是在公式中，$a=r_i，b=r_o，c=d$。

smoothstep的衰减方式与现实光源的衰减方式存在比较大的差别，所以才会更改为其他方式去计算亮度衰减。

通过新增内半径 $r_i$ 的方式，能够在点状光源的基础上进行扩展，实现圆状光源。

### 语言特性
love2d怎样向glsl传递参数：

通常情况下，love2d内传递单一变量是这样：

```glsl
uniform float test;
vec4 effect(vec4 color, Image tex, vec2 tc, vec2 sc){
    float t=test;
    return color;
}
```

- PS: glsl中接受外部参数, 使用uniform与extern均可

```lua
local shader=love.graphics.newShader("shader.glsl")
shader:send("test",12)
```

传递单光源时，上述方案其实也够用了，传递多光源时，则需要在glsl中定义列表：

同时值得注意的是，传递变量时并不需要使用“{...}”将变量括起来。
```glsl
uniform test[8];
vec4 effect(vec4 color, Image tex, vec2 tc, vec2 sc){
    float t=0;
    for (int i=1;i<8;i++){
        t=test[i];
    }
    return color;
}
```
```lua
local shader=love.graphics.newShader("shader.glsl")
shader:send("test",12,22,34)
```

传递vec2或其他类型变量时也是同理：
```lua
local shader=love.graphics.newShader("shader.glsl")
shader:send("pos",{100,100},{100,300})
```

除此之外, 有教程也提到了其他传递方式：
```glsl
struct TE{
    float test;
    vec2 pos;
};
uniform TE te[8];
vec4 effect(vec4 color, Image tex, vec2 tc, vec2 sc){
    vec2 pos=te[0].pos;
    float test=te[0].test;
    return color;
}
```
```lua
local shader=love.graphics.newShader("shader.glsl")
shader:send("te[0].test",1)
shader:send("te[0].pos",{100,100})
```
相较于第一种方式，通过构造struct的方式由于涉及到多次传递变量，故而对性能的消耗也要更大一些。

### 代码实现
glsl部分代码实现：
```glsl
const int MAX_LIGHTS=32;

uniform int light_count;
uniform vec2 light_pos[MAX_LIGHTS];
uniform float light_in_radius[MAX_LIGHTS];
uniform float light_out_radius[MAX_LIGHTS];
uniform vec3 light_color[MAX_LIGHTS];

vec4 effect(vec4 color, Image tex, vec2 tc, vec2 sc){
    vec4 pixel=Texel(tex,tc);
    vec3 total_light=vec3(0.0);
    for (int i=0;i<MAX_LIGHTS;i++){
        if (i>=light_count)break;
        float out_radius=light_out_radius[i];
        float in_radius=light_in_radius[i];
        float d=distance(sc,light_pos[i]);

        float is_inside=step(d,in_radius);
        float is_outside=1.0-is_inside;
        float dist_ratio=(d-in_radius)/(out_radius-in_radius);
        float is_valid_ratio=step(1.0,dist_ratio);
        // 亮度衰减
        float attenuation=(1.0-dist_ratio*dist_ratio)*(1.0-dist_ratio*dist_ratio)/(1.0+dist_ratio*dist_ratio*dist_ratio);
        // 画布上对于光源的处理
        total_light+=light_color[i]*(is_inside+is_outside*(1.0-is_valid_ratio)*attenuation);
    }
    // 所有光源处理结果的相加, 以及增加环境光 0.01
    vec3 c=pixel.rgb*(min(total_light,vec3(1.0))+0.01);
    return vec4(c,pixel.a);
}
```
lua部分只提供传递参数的内容：
```lua
if #cur_lights>0 then
    shader:send("light_count",#cur_lights)
    shader:send("light_pos",unpack(pos))
    shader:send("light_in_radius",unpack(rin))
    shader:send("light_out_radius",unpack(rout))
    shader:send("light_color",unpack(color))
end
```

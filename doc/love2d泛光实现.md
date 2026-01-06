# Love2D泛光效果的实现

### 简介
简单泛光效果的实现步骤简单可分为以下几点：

1. 提取亮部
2. 对亮部进行高斯模糊处理
3. 进行叠加

### 提取亮部
对像素的RGB进行加权求和得到变量 luma，当 luma 某一值时, 返回该点的颜色，也可以使用 smoothstep 进行一些过渡：
```glsl
vec4 effect(vec4 color, Image tex, vec2 tc, vec2 sc){
    vec4 pixel=Texel(tex,tc);
    float luma=dot(pixel.rgb, vec3(0.2126, 0.7152, 0.0722));
    float w=smoothstep(0.4,1,luma);
    return vec4(pixel.rgb*w,1);
}
```
### 高斯模糊
高斯模糊分为两部分，先横向再竖向或者相反。
```glsl
uniform float sigma;
uniform bool mode;

vec4 effect(vec4 color, Image tex, vec2 tc, vec2 sc){
    if (sigma<=0){
        return Texel(tex,tc);
    }

    vec2 direction=mix(
        vec2(1.0/love_ScreenSize.x,0.0),
        vec2(0.0,1.0/love_ScreenSize.y),
        float(mode)
    );

    vec4 sum=vec4(0.0);
    float all_weight=0;
    int sample=int(ceil(sigma*3));
    for (int i=-sample; i<=sample; i++){
        float offset=float(i);
        float weight=exp(-offset*offset/(2*sigma*sigma));
        sum+=Texel(tex,tc+direction*offset)*weight;
        all_weight+=weight;
    }
    return sum/all_weight;
}
```
代码中通过 mode 来控制横向或竖向，而 sigma 则控制模糊程度。值得注意的是，两次模糊使用的 sigma 值不一致时，可以更突出在某一方向上的泛光。

### 叠加
先绘制原画布，再绘制泛光画布。为了性能考虑，在进行泛光操作时，将临时画布缩小为原画布的1/4大小，以节省性能。
```lua
love.graphics.draw(canvas)
love.graphics.setBlendMode("screen","premultiplied")
love.graphics.draw(canvas_bloom,0,0,0,2)
love.graphics.setBlendMode("alpha")
```
值得注意的是，love.graphics.setBlendMode( ) 中传入的第一个值在不同场景下的表现有所不同：

在原画布上使用love2d自带的工具进行绘制时，更推荐使用"screen"，可以防止像素点亮度过高，最后使得亮部均为白色。而绘制贴图时，则使用"add"参数会获得较好观感。

### 其他
对于不同效果的青睐因人而异，推荐尝试不同的 luma 计算方法与亮部传递方式以及高斯模糊的次数，来获得最适合自己的那一份方案。
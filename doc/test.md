光照效果glsl代码部分

```glsl
uniform vec2 position;
uniform float in_radius;
uniform float out_radius;

vec4 effect(vec4 color, Image tex, vec2 tc, vec2 sc){
    vec4 pixel=Texel(tex,tc);
    float l=1.1-smoothstep(in_radius,out_radius,distance(sc,position));
    l = clamp(l, 0.0, 1.0);
    return vec4(pixel.rgb*l,pixel.a);
}

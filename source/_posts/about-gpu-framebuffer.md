---
title: 'GPU帧缓冲内存: 了解图块化 翻译后记(WIP)'
date: 2020-09-14 10:22:50
tags:
---

状态: 进行中, 完成Phong插值部分

# Phong-interpolated surface normal
使用Phong插值的表面法线, 现在很少有地方能看到这个术语了, 因为它的广泛应用已经被当成理所当然的事情.

这里就要说一些历史, 说到法线, 显然应该是只有一个面才有法线, 点是没有的, 早期的对一个三角面设置一个法线值, 那么计算渲染方程时有一个问题就是每一个三角面的颜色都是一样, 所以这个着色方式也被叫做平面着色法(flat shade). 

![一个三角面一个法线的情况](/gpu-framebuffer/images/normal_vector.png)

图片来自: [Smooth “Phong” Shading](http://learnwebgl.brown37.net/10_surface_properties/smooth_vertex_normals.html)

那么显然在计算漫反射的时候, 平面着色法的效果就无法接受, 于是出现了高洛德着色法([Gouraud shading](https://en.wikipedia.org/wiki/Gouraud_shading)), 该方法对三角面的每个顶点设置法线值, 逐顶点的计算颜色, 最后使用双线性插值计算三角面内的像素颜色.

![高洛德着色法的效果](/gpu-framebuffer/images/D3D_Shading_Modes.png)

图片来自: [Gouraud shading](https://en.wikipedia.org/wiki/Gouraud_shading)

裴祥风(Bùi Tường Phong)在1973年提出了Phong着色法([Phong shading](https://en.wikipedia.org/wiki/Phong_shading)). Phong着色法也叫作Phong插值(Phong interpolation), 或者法线向量插值着色法(Normal-vector interpolation shading), 也就是给每个顶点设置法线, 三角形内部点的法线使用线性插值进行计算. Phong着色法是对高洛德着色法的一个优化, 主要是处理使用了Phong反射模型后镜面反射部分的效果

![3个着色法的比较](/gpu-framebuffer/images/Shading-Methods-Comparison.png)

图片来自: Intergraph Computer Systems

其他参考文章:
[Flat、Gouraud、Phong Shading 着色方法对比](https://hijiangtao.github.io/2019/09/19/Shading-Methods-Comparison/)

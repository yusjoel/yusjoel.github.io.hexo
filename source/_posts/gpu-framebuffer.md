---
title: 'GPU Framebuffer Memory: Understanding Tiling(WIP)'
date: 2020-09-03 10:43:09
tags:
---
现代的图形硬件在描绘的操作过程中有大量的内存的带宽需求. 增加外部的内存带宽代价非常的昂贵, 因为需要增加额外的空间和能源, 而对于移动设备的渲染来说尤其困难. 这篇文章要讨论基于图块的渲染, 这种渲染方法在大多数的移动图形硬件上使用, 并且逐步地向桌面硬件发展.

## 立即模式光栅器

传统的图形API给出的接口是将三角形按顺序提交, 然后GPU渲染器依次渲染各个三角形. 光栅化的过程如下图所示:

_下面的图片, 包括之后所有的图片, 都是左侧显示颜色缓冲, 右侧显示深度缓冲_

![简单的立即模式渲染过程](/gpu-framebuffer/images/tech_GPUFramebuffer_01.gif)
简单的立即模式渲染过程

这些三角形一提交就会被硬件处理, 如上图所示, 称之为立即模式渲染器(IMR). 过去, 桌面和主机的GPU的做法都可以概略地认为是这样.

_在立即模式渲染器中, 图形渲染管线从上至下地处理各个原语, 逐原语的方式访问内存._

![IMR管线](/gpu-framebuffer/images/tech_GPUFramebuffer_03-cn.svg)
IMR管线

## 立即模式渲染器的内存使用

一个单纯的IMR的实现可能会花费大量的内存带宽. 下面这张图展示了即便对帧缓冲的颜色与深度做了一个简单的缓存, 也会造成光栅化过程中大量的内存数据传送. IMR在访问内存时的顺序是不可预测的, 取决于三角形提交的顺序.

_在下面这张图中, 图片的上方显示了内存中连续4个的缓存行在渲染过程中的情况. 在每个缓存行上方有一个小的矩形, 代表这个缓存行落在了帧缓冲的哪个位置: 红色线条代表缓存行被写入, 处于脏的状态, 绿色代表数据和内存一致, 处于干净的状态, 随着写入的时间推移, 红色会越来越浅. 而在下方的帧缓冲图像中, 颜色缓冲上粉红色代表脏的缓存行, 深度缓冲中则是用白色来代表._

![使用缓存行进行渲染](/gpu-framebuffer/images/tech_GPUFramebuffer_06.gif)
使用缓存行进行渲染

## 图块化内存

减少内存带宽的第一步是把每个缓存行覆盖内存中一个块二维的区域(一个图块). 在空间中相互邻近的三角形往往也会一起提交, 在上面这个例子中, 可以看到物件的每个尖刺都是先整体描绘完才去画下一个的, 更好地对缓存区域进行分组能够获得更好的命中率. 使用正方形的缓存区域, 总面积和线型区域是一样的, 但是会囊括更多的描绘, 就能减少内存数据的传输的频繁程度, 这样就能减少额外的带宽了! 相同的技术也常用于纹理的存储, 因为纹理数据的读取也表现除了空间上的引用局部性.

这个例子是被简化过的 - 现在的硬件会使用更复杂的像素与内存之间的映射机制来提高引用局部性.

_如下图所示, 此时的4个缓存行分别覆盖帧缓冲和深度缓冲上的一块正方形区域. 上方显示了覆盖区域在两个缓冲中的位置. 这些缓存行覆盖的像素和之前是一样多的._

![使用正方形的缓存图块进行渲染](/gpu-framebuffer/images/tech_GPUFramebuffer_07.gif)
使用正方形的缓存图块进行渲染

## 使用图块进行光栅化

在实际情况中, 帧缓冲会比缓存图块大很多. 当要渲染一个大的三角形, 并且简单地从上之下地那么逐行渲染, 就会造成缓存抖动, 因为屏幕上的一条水平线所包含的图块要比缓存中的图块来的多. 要解决这个问题, 我们可以改变三角形中像素的光栅化顺序: 我们可以先把一个图块中属于这个三角形的所有像素先画出来, 然后再移动到下一个图块.

由于在这个简单的例子中, 缓存的图块正好能填满帧缓冲的宽, 所以使用这种方式进行光栅化并不能减少内存数据的传输数量. 但是, 我们能看到两者之间的区别 - 先把一个缓存图块内的三角形部分画完, 再去处理下一个图块.

_下面这个动画花费的时间比上一个动画长, 因为这个动画在三角形内的一个图块更新时都会截取一帧, 而上一个动画则是在一个三角形完成渲染后或者图块与内存之间进行了数据传输后才会截取一帧. 在真实的硬件上, 两者的性能应该是一致的, 并且, 如果上一个方法造成了缓存的抖动, 那么这个版本的性能会更优._

![一次渲染一个图块](/gpu-framebuffer/images/tech_GPUFramebuffer_10.gif)
一次渲染一个图块

我们还可以继续优化对内存的访问, 不要在处理完当前图块的一个三角形内的像素就处理下一个图块, 可以把场景中所有的三角形都处理完, 再移动到下一个图块. 这个就是基于图块的渲染器(TBR)所使用的优化方法.

## 分箱(Binning)

TBR的第一步就是确定每个图块分别受到哪些三角形的影响. 一个TBR的最原始的实现是对每个图块把场景中的所有三角形渲染一遍, 再将超出图块的部分裁剪掉. 但在实际使用的时候, 帧缓冲相对于图块来说非常得大, 这个方法非常的没有效率. 取而代之的方法是, 当一个三角形被提交, 先不立即对它进行光栅化, 而是分箱到一个内存中的结构中, 这个结构定义了它影响到了哪些图块. 注意, 这个操作包含了顶点着色, 因为这和三角形的位置相关, 不过和片元着色无关.

_下面这张图演示了场景中的各个三角形是如何分箱到12个图块中, 整个帧缓冲正好可以分成4x3个图块. 下方的帧缓冲显示当前提交的三角形, 上方则按照4x3的方式排列着12个缩小的帧缓冲, 每个帧缓冲只显示被分箱到对应图块的三角形, 图块对应的区域由一条红线框出._

![将三角形分箱](/gpu-framebuffer/images/tech_GPUFramebuffer_12.gif)
将三角形分箱

## 基于图块的光栅化

当三角形完成分箱后, 光栅器就可以按箱来进行处理, 每次只对一个图块的内存进行写入, 直到这个图块处理完毕. 由于每个图块只处理一次, 那么缓存就减少到了一个图块那么大. 这个顺序操作包含了清空帧缓冲, 在图块处理过程中, 整个帧缓冲都是脏的.

![按图块进行渲染](/gpu-framebuffer/images/tech_GPUFramebuffer_14.gif)
按图块进行渲染

_渲染分成了两个阶段: 分箱, 这步需要写内存; 光栅化, 需要读取箱内数据. 几何数据的中间存储相对于帧缓冲一般而言是更小的, 并且会顺序地进行访问._

![](/gpu-framebuffer/images/tech_GPUFramebuffer_16-cn.svg)

由于需要等到所有的几何数据提交完毕才能开始光栅化, 所以和立即模式相比, 基于图块的渲染会有一个延迟. 而这个延迟带来的回报则是减少带宽, 增快光栅化速度. 在某些TBR的硬件上, 分箱和光栅化是流水线化的. 因此, 任何限制了并行性的操作都会造成性能问题, 比如说一个顶点着色器需要使用前一帧的输出, 一个纹理每一帧都会修改, 并且没有使用双缓冲. 另外, 某些TBR的硬件限制了分箱阶段的几何数据的数量.

尽管如此, 带宽的节省对于移动设备来说还是最重要的, 所以几乎所有的移动设备都使用了TBR. 甚至传统的桌面IMR供应商也在他们的新硬件中部分地使用基于图块的方法. 这意味着桌面和移动端都能在新的支持图块的API中收益, 如Vulkan Subpasses.

Since we process all the geometry contributing to the image one tile at a time, it may not be necessary to read any previous value from the framebuffer - we can clear the image as part of the tile processing (as shown above) and avoid the bandwidth cost of a read unless we really need previous contents. It is often also possible to avoid writing the depth buffer to memory (not shown in the above example), since typically the depth value is only used during rendering and does not need to persist between frames.

External traffic to the framebuffer is now limited to one write per tile - although these writes include clearing the framebuffer to a background color when no other primitives were there.

## Multisampling

Tile-based rendering also provides a low-bandwidth way to implement antialiasing: we can render to the tiles normally, and average pixel values as part of the operation of writing the tile memory. This downsampling step is known as "resolving" the tile buffer. When multisampling (as opposed to supersampling), not every on-chip pixel is shaded.

If the tile buffer is of a fixed size, antialiasing means the image must be divided into more tiles, and there are more writes from tile memory to the framebuffer - but the total amount of memory transferred to the framebuffer is unaffected by the degree of multisampling. The full-resolution version of the framebuffer (the version that has not been downsampled) never needs to be written to memory, so long as no further processing is done to the same render target. This can save a lot of bandwidth, and for simple scenes makes multisampling almost free.

_In this animation, 2x2 antialiasing has made the tile coverage in the framebuffer smaller, so more passes are needed. The geometry rasterized in the tile memory is double-sized, and shrunk when written to the framebuffer. The depth buffer is only needed on-chip, not in main memory, so only the color aspect of the full framebuffer is shown - the on-chip depth value is discarded once the tile is processed._

![Multisampled tiled rendering](/gpu-framebuffer/images/tech_GPUFramebuffer_19.gif)
Multisampled tiled rendering

## Traditional deferred shading

It is not normally possible to read from the framebuffer attachment during the process of rendering to it. Nonetheless, some techniques rely on being able to read back the result of previous rendering operations.

One such technique is "deferred rendering": only basic information is recorded as each primitive is rasterized, then a second pass is made over the rendered scene, using this recorded information as input to the shading operations per pixel. Deferred rendering can reduce the number of required costly state changes, and increase the potential parallelism available to fragment shaders.

A simple implementation of deferred rendering has a high bandwidth cost, since the entire framebuffer, including all per-pixel values, must be read and written for the deferred shading pass.

_This example shows the cache behavior in a simple deferred shading implementation. The first pass simply records the Phong-interpolated surface normal. The second pass reads this information for every pixel in the image and uses the interpolated normal for lighting calculations - reading and writing every image line in the process._

![Deferred shading with an IMR](/gpu-framebuffer/images/tech_GPUFramebuffer_21.gif)
Deferred shading with an IMR

## Tiling and deferred shading

Because deferred shading (and the related deferred lighting approach) require only the contents of the current pixel to be read, the scene can still be processed one tile at a time. Only the final result of the shading pass needs to be written to the framebuffer.

To achieve this in Vulkan, the entire sequence of rendering is treated as a single render pass, and the geometry and shading operations are each contained in a subpass. In OpenGL ES, a similar approach is possible less formally with Pixel Local Storage. With these approaches, the memory access cost of deferred shading is no greater than for simple rendering - and there is still no need to write the depth buffer.

_This example shows deferred shading in a tile-based renderer: the triangle rasterisation and the subsequent shading pass proceed within the tile memory, with only the RGB shading result being written out to the framebuffer._

![Tiled deferred shading](/gpu-framebuffer/images/tech_GPUFramebuffer_23.gif)
Tiled deferred shading

## Advantages of tile-based rendering

*   Frame buffer memory bandwidth is greatly reduced, reducing power and increasing speed.

        *   Mobile memory is typically slower and lower power than desktop systems, and bandwidth is shared with the CPU, so access is very costly.

*   With API support, off-chip memory requirements may also be reduced (it may not be necessary to allocate an off-chip Z buffer at all, for example).

*   Texture cache performance can be improved (textures covering multiple primitives may be accessed more coherently one tile at a time than one primitive at a time.

*   Much less on-chip space is needed for good performance compared with a general-purpose frame buffer cache.

        *   This means that more space can be dedicated to texture cache, further reducing bandwidth.

## Limitations of tile-based rendering

While there are many performance advantages to tile-based rendering, there are some restrictions imposed by the technique:

*   The two-stage binning and fragment passes introduce latency

        *   This latency should be hidden by pipelining and improved performance, but makes some operations relatively more costly

        *   In pipelined tiled rendering, framebuffer and textures required for rendering should be double-buffered so as to avoid stalling the pipeline

*   Framebuffer reads that might fall outside the current fragment are relatively more costly

        *   Operations such as screen-space ray tracing require writing all the framebuffer data - removing the ability to discard full-resolution images and depth values after use

*   There is a cost to traversing the geometry repeatedly

        *   Scenes that are vertex-shader bound may have increased overhead in a tiler

*   The binning pass may have limitations

        *   Some implementations may run out of space for binning primitives in very complex scenes, or may have optimizations that are bypassed by unusual input (such as highly irregular geometry)

*   Switching to a different render target and back involves flushing all working data to memory and later reading it back

        *   For a tiler, it is especially important that shadow and environment maps be generated before the main frame buffer, not "on demand" during final rendering (though this is good advice for most GPUs)

*   Graphics state (such as shaders) may change more frequently and less predictably

        *   Geometry that is "skipped" means that states do not necessarily follow in turn, making incremental state updates hard to implement

In most cases, the behavior of a tile-based GPU should not be appreciably worse than for an immediate-mode renderer using similarly limited hardware (indeed, some hardware can choose whether or not to run in a tiled mode), but it is possible to remove the performance benefits of tile-based rendering with the wrong use pattern.

## Summary

Tile-based rendering is a technique used by modern GPUs to reduce the bandwidth requirements of accessing off-chip framebuffer memory. Almost ubiquitous in the mobile space where external memory access is costly and rendering demands have historically been lower, desktop GPUs are now beginning to make use of partially-tile-based rendering as well.

Vulkan has specific features intended to make the best use of tile-based renderers, including control over whether to load or clear previous framebuffer content, whether to discard or write attachment contents and control over attachment resolving, and subpasses. OpenGL ES can achieve similar behavior with extensions, but these are not universally supported. To get the best performance from current and future GPUs, it is important to make proper use of the API so that tile-based rendering can proceed efficiently.
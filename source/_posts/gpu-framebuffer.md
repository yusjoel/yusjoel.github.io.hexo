---
title: 'GPU Framebuffer Memory: Understanding Tiling(WIP)'
date: 2020-09-03 10:43:09
tags:
---
现代的图形硬件在描绘的操作过程中有大量的内存的带宽需求. 增加外部的内存带宽代价非常的昂贵, 因为需要增加额外的空间和能源, 而对于移动设备的渲染来说尤其困难. 这篇文章要讨论基于图块的渲染, 这种渲染方法在大多数的移动图形硬件上使用, 并且逐步地向桌面硬件发展.

## 立即模式光栅器

传统的图形API给出的接口是将三角形按顺序提交, 然后GPU渲染器依次渲染各个三角形. 光栅化的过程如下图所示:

_下面的图片, 包括之后所有的图片, 都是左侧显示颜色缓冲, 右侧显示深度缓冲_

![Simple ](/gpu-framebuffer/images/tech_GPUFramebuffer_01.gif)
简单的立即模式渲染过程

这些三角形一提交就会被硬件处理, 如上图所示, 称之为立即模式渲染器(IMR). 过去, 桌面和主机的GPU的做法都可以概略地认为是这样.

_在立即模式渲染器中, 图形渲染管线从上至下地处理各个原语, 逐原语的方式访问内存._

![Pipeline of an ](/gpu-framebuffer/images/tech_GPUFramebuffer_03.svg)
IMR管线

## 立即模式渲染器的内存使用

一个单纯的IMR的实现可能会花费大量的内存带宽. 下面这张图展示了即便对帧缓存的颜色与深度做了一个简单的缓冲, 也会造成光栅化过程中大量的内存数据传送. IMR在访问内存时的顺序是不可预测的, 取决于三角形提交的顺序.

_在下面这张图中, 图片的上方显示了内存中连续4条的缓冲线在渲染过程中的情况. 在每条缓冲线上方有一个小的矩形, 代表这个缓冲线落在了帧缓存的哪个位置: 红色线条代表缓冲线被写入, 处于脏的状态, 绿色代表数据和内存一致, 处于干净的状态, 随着写入的时间推移, 红色会越来越浅. 而在下方的帧缓存图像中, 颜色缓存上粉红色代表脏的缓冲线, 深度缓存中则是用白色来代表._

![Rendering with linear cachelines](/gpu-framebuffer/images/tech_GPUFramebuffer_06.gif)
Rendering with linear cachelines

## 图块化内存

减少内存带宽的第一步是把每个缓冲线覆盖内存中一个块二维的区域(一个图块). 在空间中相互邻近的三角形往往也会一起提交The first step towards reducing memory bandwidth is to treat each cache line as covering a two-dimensional rectangular area (a "tile") in memory. Triangles that are near to each other in space are often submitted near each other in time (in this example, each "spike" of the object is drawn before moving on to the next), so better grouping of the cache area results in more cache hits. With square cache areas that are the same size as a linear cache, more rendering happens within the cache, and transfers to memory are less frequent - we've reduced external memory bandwidth! A similar technique is often used in texture storage, since the reading of texture values similarly shows spatial locality of reference.

This example is simplified - actual hardware may use more complex mappings between pixels and memory in order to further improve locality of reference.

_This time the four cache lines (spaced out horizontally) each cover a square area in the framebuffer and depth buffer. The cached framebuffer area is now shown above the corresponding cached depth buffer area. The cache lines hold the same number of pixels as for the linear cache in the previous example._

![Rendering with square cache tiles](/gpu-framebuffer/images/tech_GPUFramebuffer_07.gif)
Rendering with square cache tiles

## Rasterizing within tiles

In a real-world situation, the framebuffer would likely be larger relative to the cached tiles. One problem with the technique we've shown so far is that a large triangle might thrash the cache if drawn in simple top-to-bottom order, since each horizontal line on the screen might cover more tiles than can fit in the cache. We can solve this problem by changing the order in which the pixels within a triangle are rasterized: we can draw all the pixels that the triangle covers within one tile before moving on to the next tile.

Since in our simple example the cache can cover the entire framebuffer width, this approach doesn't reduce the number of memory transfers that are performed. However, we can see the difference - rendering for a triangle completes in one cached tile before moving to the next.

_Note: This animation takes longer than the last one because the image is updated after each tile has processed a triangle; the previous animation updated only after each triangle was completely rendered and during transfers between tiles an memory. In real hardware, performance would be the same - and, if the previous approach caused the cache to thrash, the performance of this version would be better._

![Rendering a tile at a time](/gpu-framebuffer/images/tech_GPUFramebuffer_10.gif)
Rendering a tile at a time

We would get even better memory access if, rather than just processing all the pixels corresponding to one triangle before moving on to the next tile, we processed the pixels for all the triangles in the scene. This is the optimization performed by tile-based renderers (or "TBRs").

## Binning

The first step of tile-based rendering is to determine which triangles affect each tile. A primitive implementation of a tile-based renderer could simply render the entire scene for each tile, clipped to the area covered by the cache. In practice, with large framebuffers and relatively small tiles, this would be very inefficient. Instead, when geometry is submitted for rendering, rather than being immediately rasterized, it is "binned" to a structure in memory that determines which tiles it could affect. Note that this process involves vertex shading, since this affects the location of triangles, but not fragment shading.

_This diagram shows each triangle of our scene being "binned" into twelve tiles that cover the frame buffer in a 4x3 pattern. The "framebuffer" at the bottom shows the triangle as it is submitted. Above the full-size framebuffer is a 4x3 arrangement of miniature framebuffers, each showing only the triangles that have been "binned" into the corresponding rectangular tile of the image; the area corresponding to the tile is faintly outlined._

![Sorting triangles into bins](/gpu-framebuffer/images/tech_GPUFramebuffer_12.gif)
Sorting triangles into bins

## Tile-based rasterization

Once the geometry has been sorted into bins, the rasterizer can process the scene one bin at a time, writing only to local tile memory until processing of the tile is finished. Since each tile is processed only once, the "cache" is now reduced to a single tile. This in-order processing includes clearing the framebuffer - all of the framebuffer is "dirty" until the tile is processed.

![Tiled rendering](/gpu-framebuffer/images/tech_GPUFramebuffer_14.gif)
Tiled rendering

_Rendering has now been broken into two stages: binning, which writes to memory, then rasterization, that reads the bin contents. The intermediate store of geometry is typically small relative to the framebuffer, and is accessed in an ordered way._

![](/gpu-framebuffer/images/tech_GPUFramebuffer_16.svg)

Because the rasterization of tiles cannot begin until all the geometry has been processed, tile-based rendering introduces latency compared with immediate-mode rendering. In return, the reduction in bandwidth increases the rasterization speed. In some tile-based rendering hardware, the binning and rasterizing operations are pipelined. Therefore any operation that limits this parallelism (such as a vertex shader that depends on the previous frame's output, or a texture which is modified frame-by-frame and is not double-buffered) introduces a "bubble" that reduces performance. Additionally some tile-based rendering hardware is limited in the amount of geometry which can be processed in the binning pass.

Nonetheless, the bandwidth reduction of tile-based rendering means that almost all mobile hardware is based on tiling. Even traditional desktop IMR vendors are moving towards a partly tiled approach with their latest hardware. This means that both desktop and mobile platforms can benefit from API changes designed to support tillers (such as Vulkan Subpasses).

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
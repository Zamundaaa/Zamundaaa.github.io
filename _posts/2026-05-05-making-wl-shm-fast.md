---
layout: post
title: "Making wl_shm fast"
date: 2026-05-06 15:25 +0200
categories: Wayland
---

While most new applications use the GPU for rendering to achieve better performance and battery life, there
are some new applications and a lot of older applications that still use CPU rendering.
More specifically relevant for KDE, while QtQuick is GPU accelerated, QtWidgets uses CPU rendering.

With CPU rendering, instead of sharing GPU buffers with the compositor, wl_shm is used to present images.
"shm" stands for "shared memory", and is literally just some system memory allocated by
the app and shared with the compositor.

# Why is it slow?
The rendering speed of an application using CPU rendering depends a lot on what the application
is doing exactly, but a very large factor is simply the sheer number of pixels and thus bytes it manipulates.
With high resolution screens, especially single threaded CPU rendering can get pretty slow.

Optimizing the application side isn't my area of expertise though, and not what I'm primarily interested in as a compositor developer.
My main goal is to let the application render at whatever speed it can, and to efficiently transfer the results onto the screen.

On the compositor side we can't normally use shm buffers directly. For the GPU to be able to access the data,
we first need to copy it to a different buffer that meets the requirements of the GPU. This copy is often done in two steps:
1. copy the data to a GPU-accessible buffer on the CPU
2. copy that GPU-accessible buffer to another buffer in GPU memory

With both OpenGL and Vulkan, that first copy is blocking the main thread until it's complete. You *can* offload the copy to a different thread with some additional code, but that would just move the CPU usage, rather than reduce it.

The second copy is more acceptable, since the GPU does it asynchronously and more efficiently, but on integrated GPUs, this would still end up copying data from system memory to a different region of system memory, for no good reason.

The result of these copies is that on high resolution screens with applications using shm buffers, performance noticeably suffers and CPU usage is much higher than it has any right to be.

On my laptop with a still relatively new and high end Ryzen 7840U, I could see the cursor sometimes skip frames when quickly moving it over project files in KDevelop, since KWin's main thread was being blocked by these texture uploads. Normally that's not really noticeable, but with the power profile set to "power save", it felt really sluggish.

# Vulkan will fix it... right?
> When you hold a hammer, every problem starts to look like a nail.

Since we recently started using Vulkan in KWin to fix some other problems caused by OpenGL's inadequacies[^5], I obviously looked for a Vulkan solution first. And lo and behold, `VK_EXT_external_memory_host` does exist, and it's perfect for this! Or at least it looked like it would be...

The extension allows wrapping a "host pointer" (aka a normal pointer to CPU memory) in a `VkBuffer` or even `VkImage`[^1]. With a pretty low amount of new Vulkan code, the GPU could asynchronously copy the `VkBuffer` to a GPU-local buffer.

*Unfortunately,* the implementation at least on AMD comes with some limitations. Because of potential security issues, pointers to anything associated with a file descriptor (which shm buffers always are) can't be imported this way by amdgpu.

There is also the more recent `VK_EXT_host_image_copy` for optimizing image uploads, but it would only allow removing the second copy rather than the first, so it's not exactly what I needed.

# udmabuf to the rescue
udmabuf is a Linux driver that can wrap memfd-allocated memory in a dmabuf. A dmabuf is a handle to GPU memory, and memfd is how shm buffers are usually allocated by Wayland clients... so it's a perfect fit for what I wanted to do.

There's one caveat to this: In order to be able to create a udmabuf from it, the allocated memory must be a range of memory pages[^2], so location and size have to be a multiple of the page size. Applications didn't allocate their buffers with that in mind so far, since there was no benefit to it. Fixing that isn't difficult though! Assuming one memfd per shm buffer (which at least Qt does), fulfilling the page size requirement should even be free[^4].

With the udmabuf successfully created, we can wrap it into a `VkBuffer` and do an asynchronous copy to a GPU-local buffer with Vulkan. However, we can do even better: If the stride[^3] of the buffer matches the requirements of the driver, we can directly use the udmabuf with the GPU.

This stride requirement is a bit more of a tradeoff than the page size one, since some additional memory may need to be allocated as padding at the end of each row in the image. Since most GPUs seem to be fine with a multiple of 256, the amount of "wasted" memory is still pretty low however - for example with a 3841x2160 image, it would be 0.55MB or 1.6% more memory used per buffer.

So I added code to KWin to attempt to create a udmabuf for each shm buffer, and then import that into the GPU driver. If it fails, we just fall back to the old upload code, but if it succeeds, we don't need to do any copies at all.

The compositor side didn't take a lot of code, but the application side was much simpler still. Including a comment explaining the reasoning, it merely took a grand total of 18 changed lines of code.

# The result
With the same example of KDevelop I mentioned before, the cursor is now always completely smooth. In terms of concrete numbers, KWin's CPU usage while scrolling in KDevelop went from 80-90% on one core down to 20%!

These improvements will be in Plasma 6.7 and Qt 6.11.2. I would recommend other toolkits and applications that use shm buffers to make the same changes as I did in Qt, it can make a really noticeable difference.

---
[^1]: With a Vulkan renderer, the `VkImage` would mean the second copy could be skipped as well
[^2]: A page is the smallest chunk of contiguous memory managed by the OS
[^3]: Stride is how many bytes are used by each row of pixels in an image. There can be unused padding after each row, which is included in the stride.
[^4]: The kernel allocates in pages, so the amount of memory used should be the same either way
[^5]: I'll write a blog post about it once there's more to talk about

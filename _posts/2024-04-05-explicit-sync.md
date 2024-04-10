---
layout: post
title: "Explicit sync"
date: 2024-04-05
categories: Wayland
---

Recently news went around about explicit sync being merged into Wayland protocols, and in the wake of that I saw a lot of people having questions about it, and why it was such a big deal... So here's a short-ish explanation of what it is, why it's needed and what the benefits are over the old model.

# Why is synchronization needed?
When applications "render" things, that rendering doesn't happen immediately. Instead, they effectively record a list of commands with OpenGL or Vulkan for the GPU to execute, and that list then gets handed to the GPU to execute at its own pace.

This is needed for performance reasons: If the CPU had to wait for the GPU to execute each command one by one, both CPU and GPU would often sit around, doing nothing except waiting for the other one to finish its task. By executing commands on the GPU while the CPU does other things, like preparing new commands for the GPU, both can do a lot more work in the same time.

However, in practice, rendering commands don't stand alone on their own. You might be running one task to render an image, and another one to process the result into something else, or to read it back to the CPU, so that it can be saved as a file on disk.
If you do that without synchronization, you might be reading from the image in the middle of rendering, or even before the GPU has started to work on the buffer at all.

# The "old" model: Implicit sync
Traditionally with graphics APIs like OpenGL, the necessary synchronization has been done *implicitly,* without the application's involvement. This means that the kernel and/or the userspace graphics driver look at the commands the application is sending to the GPU, check which images the commands are using, which previous tasks have to be completed before it, and potentially make the application wait until the dependencies of the commands it wants to execute are resolved.

The so-called dma buffer infrastructure that the Linux graphics stack uses for exchanging images between applications - like Wayland apps and the compositor - also uses the same model. When the render commands from the compositor try to read from an app's buffer, the kernel will delay the command's execution until the app has completed its rendering to the buffer.

This model makes it easy for application developers to write correctly working applications, but it can also cause issues.
The most relevant of them for Wayland is that the application isn't aware of which tasks it's synchronizing to, and it can happen that you accidentally and unknowingly synchronize to GPU commands that don't have any relevance to your task.

This has been a problem that Wayland compositors have been affected by for a long time:
When presenting application images to the screen, compositors picked the latest image that the application has provided, which could still have GPU tasks running on it,
instead of an earlier image that's actually ready for presentation.
This meant that sometimes presentation was delayed by the kernel, and you'd see a frame be dropped entirely,
instead of just a slightly older image.
This issue has been solved for most compositors in the last two years using the kernel's [implicit-explicit sync interop mechanism](https://www.collabora.com/news-and-blog/blog/2022/06/09/bridging-the-synchronization-gap-on-linux/)[^1]; I won't explain the details of that here, but you can read [Michel Dänzer's blog post](https://blogs.gnome.org/shell-dev/2023/03/30/ensuring-steady-frame-rates-with-gpu-intensive-clients/) about it instead.

# The "new" model: Explicit sync
The name already suggests exactly what it does: Instead of the driver or the kernel doing potentially unexpected things in the background,
the application *explicitly* tells the relevant components (driver / kernel / compositor / other apps) when rendering is complete and what tasks to synchronize to in the first place, using various synchronization primitives.

On the application side, explicit sync is used in Vulkan, and the Wayland protocol specifically is used internally by OpenGL and Vulkan drivers to synchronize with the Wayland compositor.

This explicit way of synchronizing GPU commands doesn't just help avoid accidental synchronizations, it also helps improve performance by reducing the work drivers have to do.
Instead of having to figure out the dependencies of tasks from a relatively opaque list of commands, apps just tell them directly.

An important thing to mention here is that we already had a protocol for explicit sync, [zwp_linux_explicit_synchronization_unstable_v1](https://gitlab.freedesktop.org/wayland/wayland-protocols/-/blob/main/unstable/linux-explicit-synchronization/linux-explicit-synchronization-unstable-v1.xml),
but it shared a limitation with implicit sync: In order to get a synchronization primitive, it still required the GPU commands to be first submitted to the kernel.
The new protocol in contrast allows to create and share synchronization primitives without submitting work to the GPU first, which - at least in theory - will allow applications to squeeze a little bit more performance
out of your hardware in the future.

Do keep in mind though that these performance improvements are *minor*.
While there may be some special cases where implicit sync between app and compositor was the bottleneck before, you're unlikely to notice the individual difference between implicit and explicit sync at all.

# Why the big fuzz then?
If we already have most of the compositor-side problems with implicit sync solved, and explicit sync doesn't bring major performance improvements for everyone, why is it such big news then?

The answer is simple: The proprietary NVidia driver doesn't support implicit sync at all, and neither commonly used compositors nor the NVidia driver support the first explicit sync protocol, which means on Wayland you get significant flickering and frame pacing issues. The driver also ships with some workarounds, but they don't exactly *fix* the problem either:
- it delays Wayland commits until rendering is completed, but it goes against how graphics APIs work on Wayland and can cause serious issues, even crash apps in extreme cases
- it delays X11 presentation until rendering is completed, but as Xwayland copies window contents sometimes, that still often causes glitches if Xwayland is also using the NVidia GPU for those copies

There's been a lot of discussions around the internet between people experiencing the issues constantly, and others not seeing any, and now you should know why it doesn't seem to affect everyone:
It's not a deterministic "this doesn't work" problem but a lack of synchronization, which means that a lot of factors - like the apps you use, the CPU and GPU you have, the driver version, the kernel, compositor and so on - decide whether or not you actually see the issue.

With the explicit sync protocol being implemented in compositors and very soon in Xwayland and the proprietary NVidia driver, all those problems will finally be a thing of the past,
and the biggest remaining blocker for NVidia users to switch to Wayland will be gone.

---
[^1]: this was referred to as "explicit sync through a backdoor" in an earlier version of this post

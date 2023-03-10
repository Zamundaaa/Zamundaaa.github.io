---
layout: post
title:  "Porting away from gbm_surface"
date:   2023-03-10
categories: Wayland
---

Until recently, the drm backend of KWin used `gbm_surface`s for getting buffers to display on the screen. This is a relatively simple API that allows to extend what one can do with a EGL surfaces - you get to directly choose buffer format and modifiers and you get a gbm buffer after rendering a frame, which you can then use with the drm API to actually get the image on the screen.

This also has a few drawbacks however:
- instead of being able to directly allocate a buffer, the backend has to render something first. While simply clearing the image counts as rendering, it causes unnecessary overhead
- the buffer allocation code for EGL can't be re-used for CPU or Vulkan rendering
- in order to create buffers after startup you need to create the rendering backends first
- it locks us to a maximum of allocated buffers decided by the EGL implementation (usually 4), which gets in the way of zero copy screencasting
- if the egl context is switched during rendering, `eglSwapBuffers` fails, so the rendering code needs to ensure this never happens. KWin doesn't have a 100% perfect track record for this though, which caused some fatal bugs with the Klassy window decoration

So instead of continuing to use `gbm_surface`, we decided to allocate buffers manually, import them into EGL as an `EGLImage` and use that as the color attachment for a fbo to render into. Sounds relatively simple, and it is. The problems were elsewhere...

# EGL_KHR_partial_update

Before even starting to port anything, there is one advantage to `gbm_surface`s that I left out before and which was a blocker for a long time: You can use `EGL_KHR_partial_update` to optimize which parts of the buffer are loaded into the GPU when rendering. This is irrelevant (and not implemented) for desktop GPUs but can yield significant power savings for some intergrated GPUs found in phones and tablets. As `EGL_KHR_partial_update` only works for EGL surfaces and no equivalent for fbos exists, using our own buffer management would drop support for this optimization.

How much improvement this extension actually yields was and still is not well known, some guesstimations by experts and very unscientific tests showed that its impact is very low though. So we made the decision to drop support for it, at least until an alternative for fbos is created.

# Coordinate systems

After porting everything to use the directly allocated gbm buffers, an issue turned up that took an annoyingly long time to fix: mismatching coordinate systems.

There's a few coordinate systems when concerning yourself with rendering, but the one that matters in this case is the one at the very end of the rendering pipeline - screen space coordinates. In OpenGL, this coordinate system has the origin in the *bottom* left of the screen, with the X axis pointing to the right and the Y axis pointing *up*, but for pretty much everything else, including drm, the Y axis is pointing *down*, with the origin in the *top* left.

This wouldn't be a problem on its own, but when using the EGL surface the rendering is automatically transformed to yield the results one would expect with the OpenGL coordinate system. For fbos though, Mesa can't know what it's used for, so this no longer applied. The result was that when I started KWin with the changes applied, everything was mirrored upside down.

In most rendering systems (like in games) this wouldn't be a problem. Nearly all rendering code uses a so-called *projection matrix*, which can do arbitrary transformations on the rendered geometry, like mirroring or rotating the whole screen. KWin uses this too, and adjusting it to do this is very easy. This *mostly* worked, but that *mostly* is doing a lot of heavy lifting in this sentence... KWin has a lot of optimizations to limit rendering to specific parts of the buffer, to reduce power use when only part of the screen changes. These optimizations mainly use `glScissor` and `glBlitFramebuffer` that both work in screen space coordinates, which meant that with the buffer being mirrored, they now referred to the wrong part of the screen.

# Global state

Before being able to properly fix it, there was one big problem: Some viewport, scale and now Y axis mirroring information was globally accessed state. During a compositing cycle, some effects render windows or the complete screen into an offscreen texture to use later, so this now became a big problem, with some effects suddenly rendering upside down into their offscreen texture. To fix this, I changed effects to explicitly pass a `RenderTarget` and a `ViewPort` object in all painting methods, which carry the relevant information. When an effect wants to override the properties now, it just creates its own `RenderTarget` and `ViewPort` and passes that to rendering methods instead of having to deal with global state, both fixing the problem and making the APIs a little bit more predictable.

# The blur effect

With a few effects, some of their rendering was mirrored even after adjusting the projection matrix, because they created their own projection matrices or did more complex stuff. For fixing the blur effect specifically however, I first needed to understand how it actually worked. It's not the most straightforward code, but on a high level it turned out to be not that complicated:
1. copy the part of the screen that needs to be blurred to the matching area of an offscreen texture
2. with a shader, copy that to the matching area on a smaller offscreen texture. Repeat with even smaller textures for higher levels of blur
3. do (2) again but copy from the smaller textures to the bigger ones again, and with a different shader that's better suited for this direction
4. copy the final result from the offscreen texture to the screen

The important bit is that this needs to do a lot of rendering into different coordinate spaces. This was made worse by it doing two stupid things that I ported away from first:
- it used one big texture for all screens, rather than using one per screen. This was mostly a relic from X11 times and required some additional transformations to watch out for, plus unnecessary additional VRAM usage with many multi monitor layouts
- it uploaded the same geometry with different scaling multiple times... just to then use multiple different projection matrices to undo that scaling again

The copying in step (1) was done using `glBlitFramebuffer`, which luckily is able to do mirroring for us. However, for reasons I'll get to in a moment, a different approach turned out to be better: As we're managing buffers ourselves, we also get a texture to read from, instead of only having the default framebuffer when using EGL surfaces. So to do this copy I could now adapt the code to just render a rectangle with the texture of the screen, and apply any arbitrary transformations while doing that. Once this was wired up and steps 2-4 were adjusted to use fitting projection matrices, the blur effect worked fine again.

# Screen rotation

This might seem a bit off-topic in this post at first, but with the different approach with the blur effect, porting away from `gbm_surface` managed to bring up another advantage. Before doing this, when a screen was rotated, KWin would do one of the following:
1. tell the display hardware to rotate the buffer on its own (hardware rotation)
2. render into an offscreen texture, then render the texture rotated to the screen

Hardware rotation isn't always possible though as not all hardware can do all transformations, and in the past it was also riddled with bugs on the driver side, meaning we couldn't have it be enabled by default. So in practice, if your screen was rotated, KWin would do an extra fullscreen copy of the whole screen every single time it rendered to apply the rotation. This is neither efficient nor good for performance.

There is one possible third way to handle this rotation though: Have KWin directly apply the rotation when rendering, just like when dealing with the mirrored Y axis. Actually making this work required a lot of debugging, especially for the blur effect and for screen casting, but starting with Plasma 6, the mentioned fullscreen copy for compositing will be gone.

Note that hardware rotation is still relevant even with this optimization, as it allows doing direct scanout with a rotated screen. This is relatively important for performance and battery life on devices with rotated screens, like the Steam Deck.

# Going forward

The long term plan is to make the backends exclusively deal with buffers and never interact with graphics APIs. At the moment the existing support for Xorg makes that impossible to do well, but with plans to split the code base for X11 and Wayland and eventually remove support for Xorg entirely, this will change. Even without that however, there's a bunch of code in the backends that has the potential to be shared and simplified with this change, and a few features that can be implemented easier or at all, like the already mentioned zero copy screencasting.

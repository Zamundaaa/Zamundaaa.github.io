---
layout: post
title: "HDR and color management in KWin, part 4: nonlinear blending"
date: 2024-11-02
categories: Wayland
---

In Plasma 6.2, KWin switched from doing linear blending with HDR to blending in a gamma 2.2 space. Let's take a look at what that means, and why it was done.

# What is blending?
When KWin composites, it paints window by window, going by the order of how the windows are stacked - the bottom-most window first, the topmost window last. When a window is opaque, you just overwrite the pixels in the framebuffer with the ones from the window. When a window is semi-transparent though, we need to additionally do blending.

To do blending, the GPU calculates the value that the framebuffer should have with some equation that gets the pixel from the window and the existing value in the framebuffer, and outputs some appropriate value. Usually that equation is[^1]
```
framebuffer = framebuffer.rgb * (1 - window.alpha) + window.rgb * window.alpha
```
where `window.alpha` is a per-pixel value that describes how opaque the pixel is.

# Blending with color management
In Plasma 5, compositing happened in the display color space. As displays could be assumed to be *roughly* the same, with brightness levels encoded in sRGB, that resulted in blending looking very similar everywhere.

With HDR in Plasma 6 however, that assumption was no longer true, so we had to do something else. As it was considered the most "correct" thing and allows us to ensure the exact same blending result even with displays that have wildly different colorspaces, linear blending was chosen for Plasma 6.0. The result of linear blending would look different from sRGB, but it would at least be consistent everywhere.

With linear blending, instead of just taking the rgb values as is, you first convert them into light-linear values, for example with sRGB you'd just do
```
rgb_linear = pow(rgb_sRGB, 2.2)
```
then apply the blending equation, and at the end convert it back to whatever encoding the screen needs.

Because of limitations in KWin's renderer and the ancient OpenGL versions KWin supports, the only way to do this was to render first into a so-called shadow buffer[^2] with linear values, and after compositing a second shader pass would run, taking that shadow buffer as the input and outputting a buffer with non-linear values that's suitable for sending to the screen.

# The caveats of linear blending
It'll be pretty obvious to most that doing a fullscreen copy each frame is not ideal for performance or battery life, but there was even a second problem: If you use only 8 or 10 bits per color (bpc) to store linear brightness values, you get visible banding in the dark areas of the image, as human vision is very non-linear. So on top of doing fullscreen copies, KWin also had to use a floating point buffer with 16 bpc, which uses twice the memory bandwidth vs. 8 bpc and makes performance and power usage even worse.

With that performance hit, we couldn't enable this on all hardware, but had to restrict linear blending to those displays where HDR or an ICC profile is enabled. This threw the whole consistency benefit out of the window, because now a semi-transparent surface would look a lot more transparent just because you enabled HDR... causing the very problem we wanted to avoid!

SDR | HDR
--- | ---
![gamma 2.2 blending](/assets/part 4/Screenshot_20240704_073001.webp) | ![linear blending](/assets/part 4/Screenshot_20240704_073144.webp)

We had to do *something* when blending in HDR though, so a different approach had to be taken.

# Custom transfer functions
When storing colors in buffers, usually we use non-linear encodings with transfer functions like sRGB or PQ. All the transfer functions have some sort of luminance levels attached to them, for example an sRGB value of 1.0 is defined to result in a luminance of 80 nits in its reference viewing environment[^3].

There's no reason to be restricted to those definitions though - we can make transfer functions that mean whatever we want or need. In KWin's case, we switched the shadow buffer from using a linear encoding to a gamma 2.2 encoding, in which 1.0 means the maximum luminance of your monitor. This means we do blending in a very similar way as on SDR screens[^4], and translucent surfaces on the screen look pretty much the same again, no matter what display settings you have.

As the gamma 2.2 encoding allocates more numbers to darker parts of the image, we can also avoid banding with fewer bits per color. Whenever HDR is enabled and the driver supports buffers with 10 bpc, KWin now prefers to use that instead of 16 bpc, alleviating some of the performance issues. There was still more that could be done though...

# KMS offloading
On most graphics cards, the scanout hardware that takes care of sending the image to the display has some fixed function blocks to change the colors in various ways - the most common being a simple look up table (LUT) per color channel.

Through the DRM/KMS API, the compositor can set this LUT, so we can use it to change the encoding from whatever we did blending in to the one the display needs. With HDR screens, this means we
- convert from our shadow buffer encoding with gamma 2.2 and the maximum luminance of the screen to linear
- possibly apply rgb factors for night light
- convert from linear to the PQ encoding the screen needs

With that LUT in place, we don't have to run a shader pass to convert from the shadow buffer encoding to the screen anymore, so the whole fullscreen copy falls away.

Except for a few additional instructions in the shaders used for compositing, enabling HDR in Plasma 6.2 thus has no performance impact anymore on the vast majority of hardware!

<br/>
<br/>

---
<br/>

[^1]: in practice, `window.rgb` is already "pre-multiplied" with `window.alpha`, but that doesn't really matter here
[^2]: it's called that because it's never actually shown on the screen
[^3]: the viewing environment basically defines a standardized room, in which the content can be viewed as intended without doing any brightness adjustments
[^4]: it's not exactly the same though. If you need some exactly specific blending behavior in your application, it's best if you do it yourself

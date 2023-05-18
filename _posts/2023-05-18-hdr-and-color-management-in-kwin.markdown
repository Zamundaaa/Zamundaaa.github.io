---
layout: post
title: "HDR and color management in KWin"
date: 2023-05-18
categories: Wayland
---

In this post I'll talk a bit about HDR and color management, and where we are with implementing them in KWin. Before jumping into the topic though, I need to add a disclaimer: I will be simplifying a lot of things significantly, leaving others out entirely and as I am by far not a color expert, almost certainly write a few things that are wrong. If you want more credible sources and dive into the details of how all the color stuff works, I recommend you have a look at the [color-and-hdr repository](https://gitlab.freedesktop.org/pq/color-and-hdr/-/blob/main/doc/learn.md) instead of this post.

# What is HDR? What is color management?
To explain what these things mean, it's important to take a step back and talk about how colors and brightness have traditionally been handled:

With traditional display pipelines, applications provide color data in the form of three channels, usually[^1] in the form of a red, a green and a blue value. These three values are in most cases sent to the display without modifications, which then applies the so-called "electro-optical transfer function" (EOTF) to the "electrical" values that are being sent through the cable, to get the desired brightness values and powers its red, green and blue LEDs (or equivalent[^2]) accordingly. These red, green and blue light sources then emit some amount of light, the mixture of which your eyes and brain interpret as some color and brightness.

For content to look the same on all displays, both content and displays need to use the same colorspace, or in other words, the same definitions of what exact colors the red, green and blue LEDs are and what EOTF is used. The most common colorspace is called "sRGB" and is used by most content and displays.

This way of dealing with colors is relatively simple and useful, but it also comes with some significant problems. Displays don't and often *can't* adhere to the sRGB colorspace: Due to manufacturing constraints, one display might have a little more "reddish" red, another might have a less "bluish" blue, so the same content will inevitably look different on different displays.

There's also the problem that we don't want to be restricted to sRGB. Below is a chromaticity diagram with the sRGB color gamut in it; explaining what exactly this means is a bit more than I want to go into here, but the important part is that the triangle represents the colors that can be used with sRGB (also called the *color gamut* of sRGB), with the corners being the colors of the red, green and blue LEDs of the display. The shape around it represents all colors that an average human can perceive. So as you can see, sRGB limits us to a small subset of all visible colors.

<a title="PolBr, CC BY-SA 4.0 &lt;https://creativecommons.org/licenses/by-sa/4.0&gt;, via Wikimedia Commons" href="https://commons.wikimedia.org/wiki/File:SRGB_chromaticity_CIE1931.svg"><img width="512" alt="SRGB chromaticity CIE1931" src="https://upload.wikimedia.org/wikipedia/commons/thumb/9/91/SRGB_chromaticity_CIE1931.svg/512px-SRGB_chromaticity_CIE1931.svg.png"></a>

If you now go and try to expand that triangle with the traditional way of (not) dealing with colors, you'll find some problems: sRGB content on a screen that can show more colors will look oversaturated, and content made for a screen that can show more colors will look wrong on an sRGB display. In order to fix this, we need color management.

With color management, software is aware of the color spaces the content is made for, aware of the colors that the display can show, and it does conversions between them if necessary. If it wants to show sRGB content on a screen with a wider color gamut, it can apply a (relatively simple) function to find the value that represents a given sRGB color in the wider color gamut, and thus it will look like intended on that screen. In the opposite direction, things are more complicated - if the display physically can't show some color, you have to do some lossy mapping of the bigger colorspace to the smaller one, and what kind of mapping you do depends heavily on the content, what parts of the colorspace you want to preserve best, how much processing power you have available, and much more. That's another topic too complex to really get into here though... and one I'm not familiar enough with yet either.

There is a very similar story with brightness: sRGB defines a brightness range of 0-80 nits, which is far below the up to thousands of nits some HDR displays can do. Keep in mind that this is *not* an absolute rating; almost all brightness values discussed around the topic of HDR are relative to a reference viewing environment, and relative to user preferences too, so the meaning of that value doesn't translate perfectly to an actual brightness you set your screen to. Still, you can only do so much with that small range of brightness values, and content wanting to go beyond this "standard dynamic range" has no way to express that.

There are a few high dynamic range (HDR) encodings of brightness values, which allow going beyond what sRGB can do, both for more brightness and for more resolution in the dark. The probably most widely used one is the "Perceptual Quantizer" (PQ), which is an EOTF that maps values from 0-1 to brightness values of 0-10000 nits in a way that fits the sensitivity curve of the human eye best, so that the maximum amount of resolution is used for the parts that you can actually see.

Like with the differences in colors, it's necessary to handle these different encodings and the differences in the brightness range of content vs the brightness range a given display can show, and map content values to what actually works with the display.

# The old approach
In Plasma 5 you can use colord to set an ICC color profile for a display, which describes the colorimetry of the display and contains color correction curves that adjust white balance and the EOTF.

On X11 this works on wide color gamut displays with applications that use the profile, but the colors of applications that ignore it (the majority of normal applications and games) will look wrong, and additionally you're limited to 8 bits per color channel in practice[^3], so you may see color banding even in applications that do color management.

On Wayland it could be used the same way in theory, but there is no (Wayland) API to get the profile used on a given display, which means that no applications actually make use of it. You can still set a profile for color correction (which is handled by KWin) but it won't make any applications do the color transformations required to get correct colors on a wide color gamut display.

So in summary, if you want to work with a wide color gamut you need to use X11, you need to get accustomed to bad colors in a lot of applications and there is no support for HDR whatsoever. Obviously that's not great.

# The new approach
With the Wayland protocol that's being worked on, applications tag their content with a colorspace and some other metadata, and the compositor will do whatever conversions are necessary to show it correctly on the used display, using shaders or more efficient fixed function hardware blocks on the GPU.

As all applications that do color management tag their content with a colorspace, the compositor can assume that applications that don't tag their surfaces are using sRGB, and it automatically does the required conversions to make it look good on the display you're using. Effectively you'll get the wide color gamut and increased dynamic range of the display in applications that support it, without having any downsides in applications that don't support it.

# Where are we today?
While the Wayland protocol isn't completely ready yet, there is a lot of progress happening for the protocol, kernel and compositors.

Three weeks ago I was at the HDR hackfest in Brno, organized by Red Hat. It was great meeting a lot of the people involved in this topic, people I've been only chatting with so far, and we were quite productive, discussing topics around color management, HDR and VRR, and making plans for all of them. To keep this post short I'll just link you the blog posts of [Simon Ser](https://emersion.fr/blog/2023/hdr-hackfest-wrap-up/) and [Jonas Ådahl and Sebastian Wick](https://blogs.gnome.org/shell-dev/2023/05/04/vivid-colors-in-brno/) for more details.

We didn't do a lot of hacking at the hackfest, but I did manage to drive an HDR screen with a wide color gamut and with HDR mode enabled, while having KWin do the required color conversions to make SDR content look correct.

![SDR content on an HDR screen](/assets/hdr_first.jpg)

Last week I was also at the Plasma 6 sprint in Augsburg, which was also amazing, and while it was mostly unrelated to HDR, Kai Uwe happened to have a portable OLED monitor... so of course I immediately started testing KWin with HDR on it. Piling some more hacks on top of what I put together at the hackfest, I could show a video in "HDR" surrounded by SDR content:

!["HDR" content on an HDR screen](/assets/hdr_second.jpg)

I write "HDR" in quotes because I didn't actually have the time to implement a proper HDR test client (yet) and only hardcoded KWin to boost the brightness range of the video player. Even this super simple hack already looks amazing though, especially on the OLED screen.

Since then I polished up the code, fixed lots of KWin effects to do the required color conversions, and now the first bits of basic HDR and color management support are merged in KWin! If you have a screen capable of HDR and/or a wide color gamut, and a Plasma 6 session built from git master, you can test it yourself by simply enabling the features with `kscreen-doctor` like so:
```
kscreen-doctor output.1.hdr.enable
kscreen-doctor output.1.sdr-brightness.300
kscreen-doctor output.1.wcg.enable
```
(a GUI for that will come later). In an ideal world, after adjusting the SDR brightness level to something you're comfortable with, it should look exactly like having the features disabled[^4]... which brings us to the last topic for today.

# When will it be actually useful?
Enabling HDR and a wider color gamut just to get an image that looks the same is pretty lame for an end user, the actually interesting parts are when it comes to actually gaming in HDR, playing HDR videos or painting in Krita... for those use cases however, a lot more has to fall into place than KWin being able to do color conversions. There is no way to give a good estimate for when the Wayland protocol will be ready, let alone when applications will be using it, so I'm not even gonna try.

I am however quite optimistic about the future of HDR and color management on Linux. It's all progressing pretty quickly and even just being able to fix the colors for sRGB content on wide color gamut displays with a one click solution is already a pretty good step up over what we had before.

<br/>
<br/>

---
<br/>

[^1]: there are other ways of representing color that can be used on Wayland today, but they're not super relevant for this post and not widely used (yet)
[^2]: technically speaking that's completely wrong with the most widely used display technology today (LCD). It makes things simpler to look at it this way though, so please ignore it
[^3]: you *can* set Xorg to use 10 bits per color channel but it breaks some applications
[^4]: we don't live in an ideal world, and your monitor may be doing color conversions that prevent it from looking the same

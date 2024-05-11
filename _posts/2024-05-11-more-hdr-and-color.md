---
layout: post
title: "HDR and color management in KWin, part 3"
date: 2024-05-11
categories: Wayland
---

Since the last two posts about this topic ([part one](/wayland/2023/05/18/hdr-and-color-management-in-kwin.html), [part two](/wayland/2023/12/18/update-on-hdr-and-colormanagement-in-plasma.html)) there has been some more progress, so let's take a look.

# Brightness Control
In Plasma 6.0, when HDR is enabled, you get to choose the brightness of SDR content in the display settings:

![HDR settings](/assets/HDR settings 6.0.png)

This is however not convenient to change quickly and doesn't apply to HDR content, which so far is always presented at 100% brightness. In Plasma 6.1, you'll be able to use the same ways to control brightness as on laptops or with displays that support DDC/CI in SDR mode: Just drag the brightness slider, press keyboard shortcuts or scroll on the brightness icon in the system tray, and it'll dim the whole screen like you'd expect.

![brightness slider and the system tray](/assets/brightness slider and system tray.png)

Powerdevil unfortunately only supports controlling the brightness of a single screen right now, and that limitation also applies here. The API it uses to communicate with KWin is per display though, so once powerdevil supports multiple screens, it'll work for HDR displays too.

# More accurate colors without a Colorimeter
Many new displays have a color gamut much wider than sRGB, and assume that their input signal is fitting for their color gamut - which means that, unless you use an ICC profile, colors will be much more saturated than they should be. There's a few ways to correct this:
- find an sRGB option in your monitor's OSD. This is usually pretty accurate if it's available, but also limits all apps to sRGB
- buy a Colorimeter and profile your display. That costs money though, and the display profiling situation on Linux isn't in great shape at the moment (DisplayCAL on Flathub is 4 years old, my last attempt at building it on Fedora didn't work, and it only works correctly on Xorg atm)
- find an ICC profile for your display on the Internet and use that, hoping the display doesn't deviate too much from the one that was profiled

There is a fourth option though: Use the color information from the display's EDID. In Plasma 6.1, you can simply select this in the display settings.

![color profile settings](/assets/color profile settings.png)

Note that it comes with some caveats too:
- the EDID only describes colors with the default display settings, so if you change the "picture mode" or similar things in the display settings, the values may not be correct anymore
- the manufacturer may not measure every panel and just put generic values for the display model into the EDID
- the manufacturer may put completely wrong values in there (which is why this is disabled by default)
- even when correct values are provided, ICC profiles have much more detailed information on the display's behavior than the EDID can contain

So if you care about color accuracy, this is not a way out of getting a Colorimeter and profiling your display... but if you just have two screens and you're annoyed that one of them has much more intense colors than the other, this option is an easy and fast way to fix it.

# Gamescope

Joshua Ashton implemented a new backend in gamescope, that uses Wayland subsurfaces to forward content to the host compositor instead of compositing it all into one image, and includes direct support for the `frog_color_management_v1` protocol. The result of this is that with a new enough gamescope you don't have to use any Vulkan layers to have gamescope pass HDR content to KWin, and you don't have to use the `--hdr-debug-force-output` option anymore. If you want to play a game in HDR, you can now just put
```
gamescope -W 5120 -H 1440 --hdr-enabled --fullscreen %command%
```
into its launch options in Steam (with width and height adjusted to your screen ofc) and you're done.

![Doom Eternal launch options](/assets/Doom launch options.png)

The backend also reduces the image copies made in comparison to the previously default SDL backend; the game's buffers are directly passed to the host compositor, in most cases even while overlays are visible.

# GPU Drivers
The driver situation is something I've been kind of ignoring in previous posts, but it's obviously pretty important. First, let's talk about the KMS API for setting a display into HDR mode. It consists of two parts:
- the `HDR_OUTPUT_METADATA` property, which compositors use to set mostly brightness related metadata about the image, like which transfer function is used and which brightness and color values the content roughly contains
- the `Colorspace` property, which compositors use to set the colorspace of the image, so that the display interprets the colors correctly

When you enable HDR in the system settings in Plasma, KWin will set `HDR_OUTPUT_METADATA` to the Perceptual Quantizer transfer function, and the brightness and mastering display properties to almost exactly what the display's EDID says is optimal. This is done independently of the actual image content or what apps on the screen tell the compositor, to prevent the display from doing dumb things like dimming down just because an HDR video you opened claims it wants to display a supernova in your living room.

The one exception to that is the `max_cll` value - the maximum brightness of any pixel on the screen. It's set to the maximum average brightness the display can show, because my Samsung C49RG94SSR monitor reduces the backlight brightness below SDR levels if you set the `max_cll` value it claims is ideal... With that one exception, this strategy has worked without issues so far.

On the `Colorspace` front, KWin sets the property to `Default` in SDR mode, and to `BT2020_RGB` when HDR is enabled. Sounds simple enough, but it's of course actually more complicated.

Like any API that didn't actually get used in practice for a very long time (if ever), both the API and the implementations were and are quite broken. The biggest issues I've seen so far are:
- AMD's implementation of the `Colorspace` property for DisplayPort was broken, which caused colors to be washed out in HDR mode (fixed in Linux 6.8)
- the NVidia driver doesn't force a modeset when `HDR_OUTPUT_METADATA` changes the transfer function or `Colorspace` changes its value, which causes temporary glitches when enabling HDR on some displays
- the Intel driver claims to support both properties for HDR laptop displays, but the implementation is missing entirely (this is being worked on)
- the `Colorspace` property implementations from Intel and NVidia cause washed out colors on many displays, because the API requires the compositor to change the property value depending on whether or not communication with the display uses RGB or YUV encoding... which the compositor doesn't actually know anything about. The AMD implementation works around this by translating the property to the correct value in the kernel
- multiple laptop- or display specific issues that you can look up in the bug trackers if you want to

To summarize, if you want to use HDR, it's best to use an AMD GPU with either kernel 6.8, or if you really must use an older kernel, HDMI. Even then, you might still see issues in some cases though - if you do, please make bug reports about it! Neither driver nor compositor developers can fix what they don't know is broken.

# What's next?
As always, for every bit of progress made or feature implemented, there's ten more upcoming exciting things that could be talked about or worked on... but the big next topic is offloading of color management tasks to the GPU's scanout hardware, to save power and improve performance. Next week I'll be attending the 2024 Linux Display Next hackfest, which will focus on exactly that, so stay tuned!

---
layout: post
title: "HDR and color management in KWin, part 5: HDR on SDR laptops"
date: 2024-11-06
categories: Wayland
---

This one required a few other features to be implemented first, so let's jump right in.

# Matching reference luminances
A big part of what a desktop compositor needs to get right with HDR content is to show SDR and HDR content properly side by side. KWin 6.0 added an SDR brightness slider for that purpose, but that's only half the equation - what about the brightness of HDR content?

When we say "HDR", usually that refers to a colorspace with the rec.2020 primaries and the perceptual quantizer (PQ) transfer function. A transfer function describes how to calculate a real brightness value from the "electrical" signal encoded in the content - PQ specifically has encoded values from 0 to 1 and brightness values from 0 to 10000 nits. For reference, your typical office monitor does around 300 or 400 nits at maximum brightness setting, and many newer phones can go a bit above 1000 nits.

Now if we want to show HDR content on an HDR screen, the most straight forward thing to do would be to just calculate the brightness values, write them to the screen and be done with it, right? That's what KWin did up to Plasma 6.1, but it's far from ideal. Even if your display can show the full range of requested brightness values, you might want to adjust the brightness to match your environment - be it brighter or darker than the room the content was optimized for - and when there's SDR things in HDR content, like subtitles in a video, that should ideally match other SDR content on the screen as well.

Luckily, there is a preexisting relationship between HDR and SDR that we can use: The reference luminance. It defines how bright SDR white is - which is why another name for it is simply "SDR white".

As we want to keep the brightness slider working, we won't map SDR content to the reference luminance of any HDR transfer function though, but instead we map both SDR and HDR content to the SDR brightness setting. If we have an HDR video that uses the PQ transfer function, that reference luminance is 203 nits. If your SDR brightness setting is at 406 nits, KWin will just multiply the brightness of the HDR video with a factor of 2.

This doesn't only mean that we can make SDR and HDR content fit together nicely on HDR screens, but it also means we now know what to do when we have HDR content on an SDR screen: We map the reference luminance from the video to SDR white on the screen. That's of course not enough to make it look nice though...

# Tone mapping
Especially with HDR presented on an SDR screen, but also on many HDR screens, it will happen that the content brightness exceeds the display capabilities. To handle this, starting with Plasma 6.2, whenever the HDR metadata of the content says it's brighter than the display can go, KWin will apply tone mapping.

Doing this tone mapping in RGB can result in changing the content quite badly though. Let's take a look by using the most simple "tone mapping" function there is, clipping. It just limits the red, green and blue values separately to the brightness that the screen can show.

If we have a pixel with the value [2.0, 0.0, 2.0] and a maximum brightness of 1.0, that gets mapped to [1.0, 0.0, 1.0] - which is the same purple, just in darker. But if the pixel has the values [2.0, 0.0, 1.0], then that gets mapped to [1.0, 0.0, 1.0], even though the source color was significantly more red!

To fix that, KWin's tone mapping uses ICtCp. This is a color space developed by Dolby, in which the perceived brightness (aka **I**ntensity) is separated from the chroma components (Ct = blue-yellow, Cp = red-green), which is perfect for tone mapping. KWin's shaders thus transform the RGB content to ICtCp, apply a brightness mapping function to only the intensity component, and then convert back to RGB.

The result of that algorithm looks like this:

RGB clipping | KWin 6.2's tone mapping | MPV's tone mapping
--- | --- | ---
![HDR image with clipping](/assets/part 5/clipping.png) | ![HDR image with KWin's tonemapping](/assets/part 5/kwin.png) | ![HDR image with MPV's tone mapping](/assets/part 5/mpv.png)

As you can see, there's still some color changes going on in comparison to MPV's algorithm; this is partially because the tone mapping curve still needs some more adjustments, and partially because we also still need to do similar mapping for colors that the screen can't actually show. It's already a large improvement though, and does better than the built-in tone mapping functionality in many HDR screens.

When tone mapping HDR content on SDR screens, we always end up reducing the brightness of the overall image, so that we have some brightness values to map the really bright highlights in the video to - otherwise everything just slightly over the reference luminance would look like an overexposed blob of color, as you can see in the "RGB clipping" image. There are ways around that though...

# HDR on SDR laptop displays
To explain the reasoning behind this, it helps to first have a  look at what even makes a display "HDR". In many cases it's just marketing nonsense, a label that's put on displays to make them seem more fancy and desirable, but in others there's an actual tangible benefit to it.

Let's take OLED displays as an example, as it's considered one of the display technologies where HDR really shines. When you drive an OLED at high brightness levels, it becomes quite inefficient, it draws a lot of power and generates a lot of heat. Both of these things can only be dealt with to a limited degree, so OLED displays can generally only be used with relatively low average brightness levels. They *can* go a lot brighter than the average in a small part of the screen though, and that's why they benefit so much from HDR - you can show a scene that's on average only 200 nits bright, with the sky in the image going up to 300 nits, the sun going up to 1000 nits and the ground only doing 150 nits.

Now let's compare that to SDR laptop displays. In the case of most LCDs, you have a single backlight LED for the whole screen, and when you move the brightness slider, the power the backlight is driven at is changed. So there's no way to make parts of the screen brighter than the rest on a hardware level... *but* that doesn't mean there isn't a way to do it in software!

When we want to show HDR content and the brightness slider is below 100%, KWin increases the backlight level to get a peak brightness that matches the relative peak brightness of that content (as far as that's possible). At the same time it changes the colorspace description on the output to match that change: While the reference luminance stays the same, the maximum luminance of the transfer function gets increased in proportion to the increase in backlight brightness.

The results is that SDR white gets mapped to a reduced RGB value, which is at least supposed to exactly counteract the increase of brightness that we're applying with the backlight, while HDR content that goes beyond the reference luminance gets to use the full brightness range.

Increasing the backlight power of course doesn't come without downsides; black levels and power usage both get increased, so this is only ever active if there's HDR content on the screen with valid HDR metadata that signals brightness levels going beyond the reference luminance.

As always, capturing HDR content with a phone camera is quite difficult, but I think you can at least sort of see the effect:

without backlight adjustment | with backlight adjustment
--- | ---
![without backlight adjustment](/assets/part 5/without backlight adjustment.jpg) | ![with backlight adjustment](/assets/part 5/with backlight adjustment.jpg)

This feature has been merged into KWin's git master branch and will be available on all laptop displays starting with Plasma 6.3. I really recommend trying it for yourself once it reaches your distribution!

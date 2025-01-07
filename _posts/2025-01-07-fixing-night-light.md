---
layout: post
title: "HDR and color management in KWin, part 6: Fixing night light"
date: 2025-01-07 21:00 +0100
categories: Wayland, ColorManagement
---

Most operating systems nowadays provide a feature like night light: Colors are adjusted over the course of the day to remove blue light in the evening, to potentially help you sleep[^1] and make your eyes more comfortable.

Linux is no different; there's Redshift to apply this on X11 desktops, and since many years ago desktop environments also ship it built in. However, there's always been a rather annoying problem with these implementations: None of them were color managed!

# What does it actually do
On a low level, these implementations set the video cards gamma curves to multiply the red, green and blue channels of the image that's sent to the screen.

The actual *intention* behind the feature though is to change the white point of the image. The white point is, as the name implies, the color that we consider to be "white". With displays, this usually means that red, green and blue are all at full intensity. By reducing the intensity of green and blue we can change that white point to some other color.

# What's wrong then?
The scenario I described only concerns itself with white, but there are more colors on the screen... If you multiply the color channels with some factors, then they get changed as well. To show how much and why it matters, I measured how this naive implementation behaves on my laptop.

These plots are using the ICtCp color space - the white point they're relative to is in the center of the other points, the distance to it describes saturation, and the direction compared to the center the color.

| Without night light, relative to 6504K | Plasma 6.2 at 2000K, relative to 6504K |
| ----- | ----- | ----- |
| ![without night light](/assets/part 6/absolute without night light.png) | ![with night light on 6.2](/assets/part 6/absolute 2000K on 6.2.png) |

These are both relative to 6504K, but to see the result better, let's look at it relative to their respective white points:

| Without night light, relative to 6504K | Plasma 6.2 at 2000K, relative to 2000K |
| ----- | ----- | ----- |
| ![without night light](/assets/part 6/without night light.png) | ![with night light on 6.2](/assets/part 6/2000K on 6.2.png) |

To put it into words, relative to white, red looks less intense (as white has become more red). Similarly, green becomes less intensely green... *but* it also moves a lot towards blue! Blue meanwhile is nearly in the same spot as before.
In practice, this is visible as blue staying nearly as intense as it was, and green colors on the screen get a blue tint, which is the opposite of what night light is supposed to achieve.

# How to fix it
To correct this issue, we have to adapt colors to the new whitepoint during compositing. You can read up on the math [here](http://www.brucelindbloom.com/index.html?ChromAdaptEval.html) if you want, but the gist is that it estimates what color we need to show with the new whitepoint for human eyes to perceive it as similar to the original color with the original whitepoint.

If your compositor isn't color managed, then applying this may be quite the challenge, but as KWin is already fully color managed, this didn't take a lot of effort - we just change the colorspace of the output to use the new white point, and the renderer takes care of the rest. Let's take a look at the results.

| Without night light, relative to 6504K | Plasma 6.2 at 2000K, relative to 6504K | Plasma 6.3 at 2000K, relative to 6504K |
| ----- | ----- | ----- |
| ![without night light](/assets/part 6/absolute without night light.png) | ![with night light on 6.2](/assets/part 6/absolute 2000K on 6.2.png) | ![with night light on 6.3](/assets/part 6/absolute 2000K on 6.3.png) |

| Without night light, relative to 6504K | Plasma 6.2 at 2000K, relative to 2000K | Plasma 6.3 at 2000K, relative to 2000K |
| ----- | ----- | ----- |
| ![without night light](/assets/part 6/without night light.png) | ![with night light on 6.2](/assets/part 6/2000K on 6.2.png) | ![with night light on 6.3](/assets/part 6/2000K on 6.3.png) |

Relative to the white point, red, green and blue are now less saturated but not changed as much in color, and everything looks more like expected. To put the result in direct comparison with the naive implementation, I connected desktop PC and laptop to my monitor at the same time. On the left is Plasma 6.2, on the right Plasma 6.3:

![wallpaper comparison 6.2 vs. 6.3](/assets/part 6/wallpaper 6.2 vs 6.3.jpg)

![more specific comparison 6.2 vs. 6.3](/assets/part 6/more specific 6.2 vs 6.3.jpg)

This means that in Plasma 6.3 you can use night light without colors being wrongly shifted around, and even without sacrificing too much color accuracy[^2].

# Caveats
Well, there's only one caveat really: Unless your display's native white point is 6504K, the color temperature you configure in the night light setting is a lie, even if you have an ICC profile that perfectly describes your display.
This is because instead of actually moving the white point to the configured color temperature, KWin currently offsets the whole calculation by the whitepoint of the display - so that "6500K" in the settings means "no change", independent of the display.

I intend to fix this in future Plasma versions though, so that you can also set an absolute white point for all connected displays with a convenient setting.

<br/>
<br/>

---
<br/>

[^1]: Despite the common claims about it, scientific evidence for night light improving sleep is still lacking afaik
[^2]: Adapting colors to a different white point always loses *some* accuracy though; for best color management results you should still configure the whitepoint on your display instead of doing it in software

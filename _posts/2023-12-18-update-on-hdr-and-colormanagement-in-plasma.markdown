---
layout: post
title: "An update on HDR and color management in KWin"
date: 2023-12-18
categories: Wayland
---

In my [last post about HDR and color management](/wayland/2023/05/18/hdr-and-color-management-in-kwin.html) I explained roughly how color management works, what we're doing to make it work on Wayland and how far along we were with that in Plasma. That's more than half a year ago now, so let's take a look at what changed since then!

# Color management with ICC profiles

KWin now supports ICC profiles: In display settings you can set one for each screen, and KWin will use that to adjust the colors accordingly.

Applications are still limited to sRGB for now. For Wayland native applications, a color management protocol is required to change that, so that apps can know about the colorspaces they can use, and so that KWin can know which colorspace their windows are actually using, and the upstream color management protocol for that is still not done yet. It's getting close though! For example I have an implementation for it in a KWin branch, and Victoria Brekenfeld from System76 implemented a [Vulkan layer using the protocol](https://github.com/Drakulix/VK_hdr_layer) to allow applications to use the `VK_EXT_swapchain_colorspace` and `VK_EXT_hdr_metadata` Vulkan extensions, which can be used to run some applications and games with non-sRGB colorspaces.

Apps running through Xwayland are strictly limited to sRGB too, even if they have the ability to work with ICC profiles, as they have the same problem as Wayland native apps: Outside of manual overrides with application settings there's now way to tell them to use a specific ICC profile or colorspace, and there's also no way for KWin to know which profile or colorspace the application is using. Even if you set an ICC profile with an application setting, KWin still doesn't know about that, so the colors will be wrong.[^1]

It would be possible to introduce an "API" using X11 atoms to make at least the basic arbitrary primaries + sRGB EOTF case work though, so if any developers of apps that are still stuck with X11 for the foreseeable future would be interested in that, please contact me about it!

<!-- Another important part of the story is how to *create* a profile for your screen in the first place. Profiling on Wayland *almost* works, but DisplayCAL tries to modify the gamma lut of the GPU, which it has no access to, and assumes it works anyways... The result is a profile that's wrong, so you must not use it! Fixing that is possible but not done yet. -->

# HDR

In Plasma 6 you can enable HDR in the display settings, which enables static HDR metadata signalling for the display, with the PQ EOTF and the display's preferred brightness values, and sets the colorspace to rec.2020.
<!-- In order to keep the experience in windowed mode vs fullscreen mode consistent, this signalling is constant, there is no fullscreen "pass-through" or anything like that, so games may require different brightness settings compared to Windows, consoles or gamescope to look good. -->

With that enabled, you get two additional settings:
- "SDR Brightness" is, as the name suggests, the brightness KWin renders non-HDR stuff at, and effectively replaces the brightness setting that most displays disable when they're in HDR mode
- "SDR Color Intensity" is inspired by the color slider on the Steam Deck. For sRGB applications it scales the color gamut up to (at 100%) rec.2020, or more simply put, it makes the colors of non-HDR apps more intense, to counteract the bad gamut mapping many HDR displays do and make colors of SDR apps look more like when HDR is disabled

![HDR settings page](/assets/HDR display settings.png)

There's some additional hidden settings to override bad brightness metadata from displays too. A GUI for that is still planned, but until that's done you can use `kscreen-doctor` to override the brightness values your screen provides.

KWin now also uses gamma 2.2 instead of the sRGB piece-wise transfer function for sRGB applications, as that more closely matches what displays actually do in SDR mode. This means that in the dark regions of sRGB content things will now look like they do with HDR disabled, instead of things being a little bit brighter and looking kind of washed out.[^2][^3]

<br/>

My last post ended at this point, with me saying that looking at boring sRGB apps in HDR mode would be all you could do for now... well, not anymore! While I already mentioned that Xwayland apps are restricted to sRGB, gamescope uses a Vulkan layer together with a custom Wayland protocol to bypass Xwayland almost entirely. This is how HDR is done on the Steam Deck OLED and it works well, so all that was still missing is a way for gamescope to pass the buffers and HDR metadata on to KWin.

To make that happen, Joshua Ashton from Valve and I put together a small Wayland protocol for doing HDR until the upstream protocol is done. I implemented it in KWin, forked Victoria's Vulkan layer to make my own using that protocol and Joshua implemented HDR support for gamescope nested with the Vulkan extensions implemented by the layer.

The Plasma 6 beta is already shipping with that implementation in KWin, and with a few additional steps you can play most HDR capable games in the Wayland session:

1. install [the Vulkan layer](https://github.com/Zamundaaa/VK_hdr_layer)
2. install gamescope git master, or at least a version that's new enough to have [this commit](https://github.com/ValveSoftware/gamescope/commit/9d496c8b14484949e8fed2cf551bceb6cfaf3dce)
3. run Steam with the following command:
```
ENABLE_HDR_WSI=1 gamescope --hdr-enabled --hdr-debug-force-output --steam -- env ENABLE_GAMESCOPE_WSI=1 DXVK_HDR=1 DISABLE_HDR_WSI=1 steam
```


To explain a bit what that does, `ENABLE_HDR_WSI=1` enables the Vulkan layer, which is off by default. `gamscope --hdr-enabled --hdr-debug-force-output --steam` runs gamescope with hdr force-enabled (automatically detecting and using HDR instead of that is planned) and with Steam integration, and `env ENABLE_GAMESCOPE_WSI=1 DXVK_HDR=1 DISABLE_HDR_WSI=1 steam` runs Steam with gamescope's Vulkan layer enabled and mine disabled, so that they don't conflict.

You can also adjust the brightness of sdr stuff in gamescope with the `--hdr-sdr-content-nits` flag; for a list of things you can do just check `gamescope --help`. The full command I'm using for my screen is

```
ENABLE_HDR_WSI=1 gamescope --fullscreen -w 5120 -h 1440 --hdr-enabled --hdr-debug-force-output --hdr-sdr-content-nits 600 --steam -- env ENABLE_GAMESCOPE_WSI=1 DXVK_HDR=1 DISABLE_HDR_WSI=1 steam -bigpicture
```

<!-- <br/> -->

With that, Steam starts in a nested gamescope instance and games started in it have HDR working, as long as they use Proton 8 or newer. When this was initially implemented as a bunch of hacks at XDC this year there were issues with a few games, but right now, almost all the HDR capable games in my own Steam library work fine and look good in HDR. That includes
- Ori and the Will of the Wisps
- Cyberpunk 2077
- God of War
- Doom Eternal
- Jedi: Fallen Order
- Quake II RTX

Quake II RTX doesn't even need gamescope! With `ENABLE_HDR_WSI=1 SDL_VIDEODRIVER=wayland` in the launch options for the game in Steam you can get a Linux- and Wayland-native game working with HDR, which is pretty cool.

I also have Spider-Man: Remastered and Spider-Man: Miles Morales, in both of which HDR also 'works', but it looks quite washed out vs. SDR. I found complaints about the same problem from Windows users online, so I assume the implementation in the games is just bad and there's nothing wrong with the graphics stack around them.

![jedi fallen order](/assets/Jedi Fallen Order.png)

![god of war](/assets/God of War.png)

![Cyberpunk 2077](/assets/Cyberpunk.png)

(Sorry for showing SDR screenshots of HDR games, but HDR screenshots aren't implemented yet, and it looks worse when I take a picture of the screen with my phone)

<br/>

With the same Vulkan layer, you can also run other HDR-capable applications, like for example mpv:
```
ENABLE_HDR_WSI=1 mpv --vo=gpu-next --target-colorspace-hint --gpu-api=vulkan --gpu-context=waylandvk "path/to/video"
```

![An actual HDR video](/assets/actual hdr on sdr.jpg)

This time the video being played is *actually* HDR, without any hacks! And while my phone camera isn't great at capturing HDR content in general, this is one of the cases where you can really see how HDR is actually better than SDR, especially on an OLED display:

![HDR and SDR side by side](/assets/hdr sdr side by side.jpg)

<br/>

# The future

Obviously there is still a lot to do. Color management is limited to either sRGB or full-blown rec.2020, you shouldn't have to install stuff from Github yourself, and certainly shouldn't have to mess around with the command line[^4] to play games and watch videos in HDR, HDR screenshots and HDR screen recording aren't a thing yet, and many other small and big things need implementing or fixing. There's a lot of work needed to make these things just work™ as they should outside of special cases like the gamescope embedded session.

Things *are* moving fast though, and I'm pretty happy with the progress we made so far.

<br/>
<br/>

---
<br/>

[^1]: Note that if you do want or need to set an ICC profile in the application for some reason, setting a sRGB profile as the display profile is *wrong.* It must have rec.709 primaries, but with the gamma 2.2 transfer function instead of the piece-wise sRGB one so often used!
[^2]: This is the reason for footnote 1
[^3]: As far as I know, Windows 11 still does this wrong!
[^4]: I recommend setting up .desktop files to automate that away if you want to use HDR more often. Right click on the application launcher in Plasma -> Edit Applications makes that pretty easy

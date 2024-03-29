---
layout: post
title:  "Gaming on Wayland"
date:   2021-12-14
categories: Wayland
---

A considerable amount of people assume Wayland isn't particularly suitable for gaming, usually because you can't turn off the compositor.
This post will challenge that assumption and see how the current state of gaming on Wayland is, with a focus on KWin, KDEs compositor.

### Latency, part 1

When talking about gaming on Wayland what often comes up is increased latency in comparison to uncomposited X, because of compositing and forced VSync. Before going into how good or bad latency really is on Wayland, some explanations on what all those things actually mean are in order.

What usually gets called "input latency" for gaming is the time it takes from generating some input event, like a mouse or keyboard button press, to seeing the resulting in-game event on the display.
This "input" latency can be split up into multiple parts, for example
* actual input latency: button press -> computer registers it
* input routing latency: computer registers it -> input gets forwarded to game
* rendering time: game gets input -> game reacted to the input and rendered a new frame with it
* presentation time: frame rendered -> display server sends data to the display
* display reaction time: data received by the display -> actual pixels change their color

What we have influence over with X and Wayland are input routing latency and presentation time. While input routing latency is _usually_ trivial, presentation time is not. To understand how and why you first have to know how images get from the GPU to the display.

### How displays work

Let's assume a very simplified hypothetical case for a start: The display was just turned on, nothing is being shown yet. The GPU sends some image to the display, beginning from the pixel on the top left and going through them line by line. The display updates its physical pixels one by one as the data comes in until it's through all of them - then it starts again from the top.
How often the display goes through all pixels in a given time period is called the refresh rate; for example a 60Hz display refreshes 60 times a second and thus takes about 1000ms / 60 = 16.66ms per frame.

To simplify things even more, let's assume the computer is only sending one single (random) color for the whole screen. To illustrate this I rendered a simulated display with a very low refresh rate, with the pixel on the left showing the color that's supposed to be presented and the space on the right showing the actual display:

<blockquote class="imgur-embed-pub" lang="en" data-id="a/sUkSFL8" data-context="false" ><a href="//imgur.com/a/sUkSFL8"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>

Now the resulting static image is quite boring, we want to change it frame by frame to make it seem like things are moving - in this case, we just change the color. The very simplest approach for that would be to change the image data whenever the computer is done rendering a new frame. This works but isn't the most pleasant to look at, especially if the application is faster than the display (in this case, about 3 times as fast):

<blockquote class="imgur-embed-pub" lang="en" data-id="a/WI412m5" data-context="false" ><a href="//imgur.com/a/WI412m5"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>

When we're updating the pixel data in the middle of one display refresh cycle we end up showing multiple different versions of the image at the same time, which creates a visual break. This break is called tearing and it can be prevented with vertical synchronization, VSync for short.
With VSync the GPU only changes the image data it's sending in the so-called vertical blanking interval (vblank), which is the time between the bottom right and top left pixel.

In this time the display is not actually updating any pixels, it's just waiting. Why does it wait and do nothing you might be wondering? It's mostly a remnant from CRT times - these old displays actually needed to wait until magnetic fields deciding the direction of an electron beam changed. Nowadays vblank can be decreased a lot but it's still used, both for VSync and for transmitting metadata like color information or the content type.

In the video below I visualized vblank as another row of pixels that you couldn't normally see:

<blockquote class="imgur-embed-pub" lang="en" data-id="a/AvlhKmU" data-context="false" ><a href="//imgur.com/a/AvlhKmU"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>

However, as we're updating the image only at specific points in time and never in between this can significantly increase the time between when a frame is rendered and when you see it. If the application updates slower than the display then this can also cause stutter, as the time it takes for content to reach the display varies frame by frame. If the application updates faster, then frames will be skipped and also create stutter, albeit usually less noticeable. In order to counteract this, applications additionally synchronize their rendering to the display where possible - they only render one frame for every time the display updates. As a bonus this also radically reduces resource usage and power draw.

There is another technology that interacts with this, called Variable Refresh Rate (VRR) or Adaptive Sync, although it's better known as its implementations FreeSync (from AMD) and GSync (from NVidia).
With VSync it's always the GPU which has to adjust to the display, but with Adaptive Sync the roles are partly reversed: the GPU can tell the display to make vblank longer.

Whenever an application is slower than the displays maximum refresh rate the display will wait for the next frame before it updates the pixels and thus you don't see any stutter, without introducing tearing. Additionally it has very low latency in that state because every frame is displayed right when the game is finished rendering it.
Not all displays have this functionality (yet) because it's difficult to make it work well: In many displays with different refresh rates the brightness changes a lot, so if the refresh rate jumps around you'll see the display constantly flicker in brightness.

### Display Servers

What are X and Wayland, and why do they matter?

Applications don't usually present directly to the display, that is the job of the display server. A display server takes care of compositing, which is the task of combining the images provided by applications into the one that ends up on the display, and of then putting that composited image on the screen.
In order to communicate with the display server, applications use one of several protocols; most widely used on Linux and BSD are X11 and the newer Wayland.

For X11 the main implementation is called the X.org server, often called "X" for short. X handles many things like input, window focus, basic compositing, presentation of images on the display, allocating graphics memory for applications and much more.
It doesn't handle everything though, there is two programs that extend the functionality of X:

There is the effectively mandatory window manager, which takes care of the stacking order (which windows are over which), focus changes and basic window movement and resizing. The other extension, which is optional, is X11 compositors, that can do more advanced compositing than the X server itself is capable of, like transparency, blur and a variety of other possible effects. These two programs are usually combined into a single one that does both window management and compositing.

On Wayland the display server is called the Wayland compositor, of which there are a few widely used implementations, for example KWin (KDE), Mutter (GNOME) and Sway.
Essentially Wayland compositors take care of everything the X server, a window manager and X11 compositor would do together.
In order to allow X11 applications to still function on Wayland there's also a compatibility layer called Xwayland, which is what most games currently run with if you're using Wayland.

Please note that this explanation is leaving a *lot* of things out and simplifies others, it's a much more complex situation than could be explained in a few sentences... and there's a lot of parts where my knowledge is limited, especially concerning X.

### Compositing and gaming shortcomings on X

On X, when an application submits a new image to be shown to the user, it goes to the X11 compositor (if one is active at the moment) which uses that image to composite the final image that goes to the output. Due to limits in the X11 protocol when an application does VSync, the image from it goes to the compositor exactly at the time when it's too late to present it for the current frame - it gets delayed by one additional refresh cycle. That increase in latency with in-game VSync enabled it's one of the big reasons for why X with compositing can feel sluggish to some.

There is another big reason, another limitation of X11: the handling of multiple monitors. All the monitors you have are put together into one screen for X, which means that the compositor can only present one big image for all outputs at once. The result is that compositors can't synchronize rendering for all outputs at the same time, causing stutter and limiting the refresh rate to the slowest monitors - and making adaptive sync with multi-monitor impossible.
There are workarounds for this problem but none of them can remove all problems at the same time.

### Wayland

What does do Wayland differently? Just about everything you can think of. Explaining everything in full detail would go far beyond both the scope of this post and what I know myself so I'll limit this to the most gaming relevant bits:

* the one frame of added latency with compositing and game VSync does not apply to Wayland
* due to some better design decisions Wayland is a little bit more efficient than X, which can make a difference in performance. Don't expect wonders though, the effect is _very_ small
* there is no restrictions on how screens need to be painted. Things like rendering multiple screens with different refresh rates, with variable refresh rate, with multiple threads and pretty much any graphics API are possible, which allows for many more features. [Gamescope](https://github.com/Plagman/gamescope) for example is using Vulkan with asynchronous compute to make compositing reliably as fast as possible
* there’s not really any restrictions on whether things need to be painted to a screen at all either. A full VR/AR based “desktop” environment where you can put windows wherever you want is not out of the question and could go far beyond what applications like xrdesktop can do (which is already super awesome, check it out if you have a VR headset)
* there is a lot of restrictions on what native Wayland apps can do. For example, global shortcuts like `alt`+`tab` are always guaranteed to work, no matter what the game does - the compositor has the final say in effectively everything
* application contents can be put directly on the screen with the display hardware (which is called “direct scanout”. You may have heard the X related term “unredirection” used for the same thing, too). This removes unnecessary copies and provides some nice efficiency and latency benefits - even in windowed mode, if the compositor and hardware support it
* HDR and color correction for all apps. This is still gonna take a long time to be realized but it's finally in sight. Something like Windows AutoHDR, which tries to "upgrade" SDR game content to HDR is in the realm of possibility as well
* more efficient screen recording
* great multi gpu support. To go into detail would be a bit much but the gist of it is that it's efficient, reliable and allows (at least in principle) for GPU switching on the fly

That's all great in theory, but where are we actually in practice right now?

### KWin

As of version 5.23 kwin_wayland supports
* rendering multiple screens at different refresh rates
* dynamic adjustment to the current rendering load, to minimize latency
* Adaptive Sync / FreeSync / (once NVidia supports it on Wayland) GSync
* direct scanout with fullscreen apps

Some notable gaming related things that will be added in the future:
* support for VR headsets, coming in 5.24. Basically SteamVR and Monado will 'just work'
* dmabuf feedback, also coming in 5.24. It should make direct scanout work with most applications in most situations
* allowing to disable VSync, for applications and, if practical, also globally. Note that this is only about the "allow tearing" part mentioned in `Latency, part 1`; Wayland does not and can not limit the frame rate of games if you turn their "VSync" setting off
* direct scanout for windowed operation where possible
* better multi gpu support. While you can choose the GPU KWin will use with an environment variable, that is not exactly user friendly. I'm working on implementing a GUI for choosing the GPU used for compositing - this way you can change it without needing to log out and in again, and you can disconnect GPUs from the system entirely, too, which should help with VFIO use cases
* tiled displays. This is a special type of display that is not using one but multiple display controllers - which are connected to multiple internal ports on the GPU (but may still use a single cable) - in order to support very high resolutions and refresh rates. Once support for tiled displays is complete, the same functionality will also be used to make something akin to AMD EyeFinity possible, so that you can make everything behave as if your multiple displays were a single one

### Latency, part 2: measurements

Now that all the explanations are out of the way, let's get to how it really is!
Lacking the proper tools to do it, I built myself a latency measurement tool with two microcontrollers and a brightness sensor. One microcontroller acts as the actual measurement tool that triggers a mouse click and waits for the screen to respond, the other as a logger. This allows for relatively painless testing. The test setup itself is
* Ryzen 5800X + rx 6800 XT, compute power is not a limiting factor - it's a best case scenario for mailbox and immediate mode
* Samsung Odyssee C49RG94SSR at 120Hz (8.3 milliseconds refresh cycle) with FreeSync enabled
* brightness sensor set up to measure the middle of the screen
* latest Manjaro stable on Linux 5.15 with Mesa 22.0 (devel)
* unpatched KWin git master with the latency setting set to default, and no working direct scanout for Vulkan applications. This should be representative of stock 5.23 (at least on my GPU), no relevant code paths have changed
* an app using Vulkan and glfw I wrote a while ago, modified to blank the screen white whenever the mouse is pressed

As my hacked-together measurement tool hasn't been verified for correctness beyond some logic checks, please be aware that these numbers are only usable for approximate comparison and not for judging the absolute latency of a system accurately. They're also not necessarily representative of normal latency you'd get with your system or specific games.

All values in the tables below are in milliseconds. Don't get hung up on differences of a single millisecond, even with a thousand measurements there's still some noise in it.

|X with compositing   |fifo\* |mailbox   |
|  ---                |---    |---       |
| median              |59     |37        |
| 99th percentile     |67     |46        |

|X without compositing|fifo   |mailbox   |immediate    |FreeSync |
|  ---                |---    |---       |---          |---      |
| median              |41     |38        |19           |24       |
| 99th percentile     |49     |45        |26           |32       |

|Wayland              |fifo\*\*|mailbox    |immediate\*\*\*|FreeSync |
|---                  |---     |---        |---            |---      |
| median              |49      |36         |20             |24       |
| 99th percentile     |56      |44         |33             |31       |

|Xwayland             |fifo\*\*|mailbox    |immediate\*\*\*|FreeSync |
|---                  |---     |---        |---            |---      |
| median              |49      |38         |20             |25       |
| 99th percentile     |57      |46         |33             |33       |

The presentation modes are
* `fifo`: "first in, first out". fps is limited + VSync - this is what most "VSync" settings in games do
* `mailbox`: no fps limit + VSync
* `immediate`: no fps limit, no VSync. On X without compositor this causes screen tearing, on X with compositor or on Wayland it's the same as mailbox (thus left out there)
* `FreeSync`: mailbox with the frame rate artificially limited to 115 with GOverlay / mangohud, so that the display is always synchronizing to the game

\* as already explained, one frame of latency is guaranteed. The second additional frame I can't explain well but I haven't looked into it much, X11 is neither my area of expertise nor do I see a reason to change that

\*\* due to increased buffer bloat (the queue for presentation being one frame bigger) the latency with `fifo` is higher by one frame than on uncomposited X. This should disappear once all the necessary parts for dmabuf feedback are implemented in Mesa

\*\*\* KWin was patched with a very simple but bad hack to make testing this possible

# Conclusion

As so often with these things, it comes down to requirements and trade offs. Generally Wayland already does quite well in regards to latency - and if you want features like smooth multi monitor operation, FreeSync with multiple monitors or don't want to be forced to disable the compositor for better gaming, it could be ready for you!

If you require the absolutely lowest latency possible though and don't care about screen tearing then X without compositing, with VSync disabled is a better fit for you. You can re-evaluate once the possibility to have VSync disabled is implemented in your compositor of choice and the measured increased volatility in latency (99th percentile) with immediate mode is fixed.

I personally will keep on using Wayland and leave the global frame rate limit of 115 enabled - the computer makes less noise and heat that way, while the latency is very much low enough for me to not notice it.

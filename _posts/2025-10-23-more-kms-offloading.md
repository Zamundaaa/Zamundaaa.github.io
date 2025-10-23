---
layout: post
title: "More KMS offloading, with overlay planes"
date: 2025-10-23 18:00 +0200
categories: Wayland
---

In preparation for, and at the 2025 display next hackfest, I did a bunch of hacking on using more driver features for improved efficiency and performance.
Since then, I polished up the results a lot more, held a talk about this at XDC 2025 and most of the improvements are merged and now released with Plasma 6.5. Let's dive into the details!

# Using more planes
The atomic drm/kms API exposes three important objects to the compositor:
- connectors, which simply represent the connection to the display. In most cases there's one per connected display
- CRTCs, which roughly represent the machinery generating the data stream that's sent to a display
- planes, that are used to define which buffers are composed in which way to create the image on a given CRTC. There's three types, primary, cursor and overlay planes

When trying to drive a display, the compositor needs to put buffers on at least one primary plane, connect it to a CRTC, connect that to a connector, and then do an atomic test to find out if that configuration can actually work. If the configuration doesn't work, because of hardware or driver restrictions, the compositor needs to fall back to a different (usually simpler) one.

Up to Plasma 6.4, KWin (mostly[^1]) only used primary and cursor planes. Using the GPU, it composited all the windows, decorations and co. into one buffer for the primary plane, and the cursor into another buffer for the cursor plane. Whenever the cursor plane wasn't available or the configuration using it didn't work, it would fall back to compositing the cursor on the primary plane instead (which is usually called a "software cursor").

## Why even use multiple planes?
While compositing everything with the GPU allows for fancy features, color management, blur, wobbly windows and more, it does require the GPU to process lots of data. Depending on the hardware, this may be somewhat slow or incredibly fast, but it always takes some amount of time, and often uses a lot of power.

By using a plane for the cursor for example, when you move it, the compositor doesn't have to re-render the screen, but it can just set the new cursor position on the hardware plane, which basically immediately changes the image that's sent to the screen - reducing both latency and power usage.

In other cases we don't really care about latency so much, but power usage is more important. The best situation is possible with video playback: If the application uses hardware decoding and passes the decoded video to the compositor unmodified, it can put the video directly on a plane, and the rest of the GPU can completely turn off! This already works in fullscreen, but with more planes we could do it for windowed mode as well.

So now you know *why* we want it, but getting there was a longer story...

## Preparing the backend
Our drm backend had a relatively simple view of how to drive a display:
- every output had one primary and one cursor layer, with matching getters in the render backend API
- this layer wasn't really attached to a specific plane. Even if your hardware didn't have actual cursor planes, you'd still get a cursor layer!
- when rendering, we adjusted to the properties of the actually used plane, and rejected rendering layers without planes
- when presenting to the display, the code made assumptions about which planes need which features

This was quite limiting. I refactored the backend to instead create a layer for each plane when assigning a plane to an output, have a list of layers for each output, and treat all of them nearly exactly the same. The only difference between the layers is now that we expose information about how the compositor should use them, which is mostly just passing through KMS properties, like the plane type, size limitations and supported buffer formats.

Last but not least, I changed the backend to assign all overlay planes whenever there's only one output. This meant that we could use overlay planes in the most important case - a single laptop or phone display - without having to deal with the problems that appear with multiple screens. This restriction will be lifted at some later point in time.

## Preparing compositor and scene
The next step was to generalize cursor rendering - compositor and scene both had a bunch of code specifically just about the cursor, listening to the cursor image and position changing and rendering it in a special way, even though it was really just another `Item`, just like every window decoration or Wayland surface. I fixed that by adding the cursor item to the normal scene, and adding a way for the compositor to hide it from the scene when rendering for a specific output. This allowed me to adjust the compositing code to only care about item properties, which could then be reused for different things than the cursor.

As the last cursor change, I split updating the cursor into rendering the buffer and updating its position. The former could and should be synchronized with the rest of the scene, just like other overlays, but the latter needs to be updated asynchronously for the lowest possible latency and to keep it responsive while the GPU is busy. With that done, the main compositing loop could use primary and cursor planes in a rather generic way and was nearly ready for overlays.

## Putting things on overlays
The main problem that remained was selecting what to put on the overlay planes that are available. There are some restrictions for what we can (currently) put on an overlay:
- the item needs to use a dmabuf, a hardware accelerated buffer on the GPU
- the item needs to be completely unobstructed
- the item can't be currently modified by any KWin effect

To keep things simple and predictable, I decided to just make KWin go through the list of items from top to bottom, take note of which areas are obstructed already, and find items that match the criteria and get updated frequently (20fps ore more). If there are more frequently updated items than planes, we just composite everything with the GPU.

Last but not least, we do a single atomic test commit with that configuration. If the driver accepts it, we go ahead with it, but if it fails, we drop all overlays and composite them on the GPU instead. Maybe at some point in the future we'll optimize this further, but the main goal right now is to save power, and if we have to use the GPU anyways, we're not going to save a lot of power by merely using it a little bit less.

## Putting things on underlays
The concept of underlays is quite similar to using overlay planes, just instead of putting them above the scene, you put them below. In order to still see the item, we paint a transparent hole in the scene where the item would normally be.

This is especially useful whenever there is something on top of the item that isn't updating as frequently. The best example for that is a video player with subtitles on top - the video updates for example 60 times a second, but the subtiles only once every few seconds, so we can skip rendering most of the time.

There isn't really that much more to say about underlays - the algorithm for picking items to put on overlays needed some minor adjustments to also deal with underlays at the same time, and we needed to watch out for the primary plane to have enough alpha bits for semi-transparent things (like window shadows) to look good, but overall the implementaiton was pretty simple, once we had overlay plane support in place.

The result however is quite a big change: Instead of getting an overlay in some special cases, KWin can now use planes in nearly all situations. This includes the newly introduced server-side corner rounding in Plasma 6.5! We simply render the transparent hole with rounded corners, and we can still put the window on an underlay with that.

There is one thing that did not land in time for Plasma 6.5 however: The current algorithm for underlays only works on AMD, because amdgpu allows to put overlay planes below the primary one. I have an implementation that works around this by just putting the scene on an overlay plane, and the underlay item on the primary plane, but it required too many changes to still merge it in time for Plasma 6.5.

## Required changes in applications
Most applications don't really need to change anything: They use the GPU for rendering the window, and usually just use one surface. If the entire window is getting re-rendered anyways, like in the case of games, putting the surface on an overlay or underlay is quite simple.

There are however some situations in which applications can do a lot to help with efficiency. If you're not already using the GPU for everything, you'll want to put the quickly updating parts of the app on a subsurface. For example, mpv's dmabuf-wayland backend puts
- the video background on one surface with a black single pixel buffer
- the video on separate surface
- playback controls and subtitles on another separate surface

which is the absolute best case, where we can basically always put the video on an underlay. If the video is also hardware decoded, this can save a lot of power, as the GPU can be completely turned off.

You also want to support fractional scaling properly; while some hardware in many situations is fine with scaling buffers to a different size on the screen, there are sometimes hardware restrictions on how much or even if it can scale buffers.

# Using drm color pipelines
The described overlay and underlay improvements are great... but have one big flaw: If presenting the Wayland surface requires color transformations, we have to fall back to compositing everything on the GPU.

Luckily, most GPU hardware can do some color operations on the planes. The API for those color operations has been worked on for a long time, and I implemented support for it in KWin. With the relevant kernel patches, KMS exposes color pipelines - each a list of color operations, like a 3x4 matrix or per-channel 1D and 3D lookup tables, which the compositor can program in whatever way it wants. Every time we attempt to put an item on a hardware plane, we also attempt to match the required color transform to the color pipeline.

With the patchset for that, on my AMD laptop I can open an HDR video in mpv, and even if the video has subtitles, is partially covered by another window and the screen is SDR, the video is presented correctly without the GPU being involved!

# How much does it really help?

Now the most interesting part: How much power does this actually save in the long run? I did some measurements with all the patches put together.

To test this, I played each video on a Framework 13 at 50% display brightness, with "Extended Dynamic Range" enabled and the keyboard backlight turned off, and recorded the power usage from the sysfs interfaces for battery voltage and current. The results you see in the table are the averages of 15 minutes of video playback, so the numbers should be pretty reliable.

On the application side, I used YouTube videos in Firefox with `gfx.wayland.hdr` enabled. As YouTube didn't allow me to play HDR videos for some reason, I used mpv with the dmabuf-wayland backend to play back a local video instead.

 Video              |  without overlays | with overlays |
  ---               |  ---              |  ---          |
4k SDR in Firefox   | 13.3W             | 11.5W         |
1080p SDR in Firefox| 11W               | 9.6W          |
4k HDR in mpv       | 13.4W             | 12.4W         |

Or in terms of (estimated) battery life:

 Video              |  without overlays | with overlays |
  ---               |  ---              |  ---          |
4k SDR in Firefox   | 4.6h              | 5.3h          |
1080p SDR in Firefox| 5.5h              | 6.4h          |
4k HDR in mpv       | 4.6h              | 4.9h          |

As a reference for these numbers, the laptop being completely idle with the same setup uses about 4.5W, which equals about 13.5 hours of battery life. So while this is a good start, I think there's still a lot of space to improve. At XDC this year I was told that we may be able to do something about it on the application side by using the hardware decoder more efficiently; I'll do another run of measurements whenever that happens.

# When can I start to use this?

Due to various driver issues when trying to use overlays, like slow atomic tests on AMD as well as display freezes on some AMD and NVidia GPUs, this feature is still off by default.

However, if you want to experiment anyways or attempt to fix the drivers, starting from Plasma 6.5, you can set the [KWIN_USE_OVERLAYS](https://invent.kde.org/plasma/kwin/-/wikis/Environment-Variables#kwin_use_overlays) environment variable to enable the feature anyways. If you test it, please report your findings! If there's problems in the drivers, we'd like to know and have bug reports for the GPU vendors of course, but also if things work well that would be nice to hear :)

When we can enable it by default isn't quite clear yet, but I hope to be able to enable it by default on some drivers in Plasma 6.6.

<br/>
<br/>

[^1]: If there was no cursor plane, it was able pick an overlay plane instead, but that was it.

-----

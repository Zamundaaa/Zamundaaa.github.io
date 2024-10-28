---
layout: post
title: "PSA: KDecoration API break in Plasma 6.3"
date: 2024-10-29
categories: Wayland
---

Fractional scaling is hard. Anyone that had the misfortune of working on it knows that... so it won't surprise a lot of people that it's not all figured out yet! Today I'll talk about the fractional scaling problems with KWin's server side decorations, and why we need to do an API break to fix it.

# What's the problem?
This is the simplest part. Many decorations have elements that need to be pixel perfect, like outlines that are only a single pixel wide. When they're not perfectly scaled, or positioned wrongly, that's sometimes quite visible and annoying:

<p align="center">
<img src="/assets/fractional scaling/Screenshot_20241026_103055.png" width=500/>
</p>
<p align="center">
<img src="/assets/fractional scaling/Screenshot_20241026_101804.png" width=500/>
</p>
<p align="center">
<img src="/assets/fractional scaling/Screenshot_20241026_102013.png" width=500/>
</p>

# What causes these issues?
The source of all evil with fractional scaling is also the cause of most issues here: Integer logical coordinates.

Logical coordinates are a way to represent the size of something on the screen in a mostly display-independent way and are quite useful for the size and position of things like windows or the cursor. They're calculated in a really simple way:
```
coordinate_logical = coordinate_pixels / scale
```
With just that equation, there are no problems just yet - you can just multiply the logical coordinate with the display scale, and you get back the original coordinate in pixels. When you round that logical coordinate, and do some calculations with it, things get weird though... let's look at the concrete example of a window at scale 1.25, and with a 1 pixel wide outline:

unit               | outline width | window width | outline width | total size | total size in pixels
---                | ---           | ---          | ---           | ---        | ---
(integer) pixels   | 1             | 27           | 1             | 29         | 29
fractional logical | 0.8           | 21.6         | 0.8           | 23.2       | 29
integer logical    | 1             | 22           | 1             | 24         | 30

As you might've guessed, KWin's decoration plugin API is using integer logical coordinates, and this mismatch between the window size vs. the size of its components causes most of the problems.
Just doing a straight forward int -> float conversion isn't enough to fix this though, a few more changes are needed.

# Changes in KWin
KWin will provide decorations with the fractional logical size of windows, provide them with the scale factor they should render for, and use the decoration's fractional border sizes to position the window and decoration pieces properly in the scene.

# Changes in Decorations
Because of the API break, decorations using the C++ API *need* to be updated to the new KDecoration3 API, or they will not be loaded. A minimalistic port would only need to round all the values, but there will of course still be fractional scaling issues with that.

Assuming you want to make the decoration work properly with fractional scaling, you also need to use the provided scale factor to calculate border sizes, and when painting things with QPainter, you need to take care to snap all geometries to the pixel grid, or anti-aliasing may turn single-pixel lines into a blurry mess.

Note that this work isn't completed yet, and some additional API changes may happen while we're breaking the API already. A porting guide with all the changes will be provided before the release of Plasma 6.3.

As Aurorae decorations are just svg files, they are not affected by this API break and will continue to work like before without any changes.

If you have any questions about this change, or about how to port a decoration over to the new API, please reach out to us at #kwin:kde.org on matrix!

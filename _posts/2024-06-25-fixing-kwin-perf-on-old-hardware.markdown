---
layout: post
title: "Fixing KWin's performance on old hardware"
date: 2024-06-25
categories: Wayland
---

KWin had a very long standing bug report about bad performance of the Wayland session on older Intel integrated graphics. There have been many investigations into what's causing this, with a lot of more specific performance issues being found and fixed, but none of them managed to fully fix the issue... until now.

# The source of the problem

Understanding the issue requires some minimal understanding about how displays work. [My earlier post about gaming on Wayland](/wayland/2021/12/14/about-gaming-on-wayland.html) goes into more depth about them and the different presentation modes, but the TL;DR is that most displays today require frames to be sent to it in fixed intervals if you want them to be shown without tearing artifacts.

With the vast majority of displays, that interval is 16.67ms. In order to show everything as smooth as possible, the compositor thus has to render a new frame every 16.67ms as well; it must not miss a single deadline or the user will see stutter as some frames are shown twice and others are skipped.

This problem is not unique to Intel of course, when the deadline is missed, that causes the same stutter on every system. It's just an often reported issue on old Intel processors because they're used a lot, because both CPUs and GPUs in them are pretty slow, and laptop manufacturers too often paired them with high resolution screens, requiring the GPU to render for a long time each frame.

# How KWin deals with this deadline

In the past, KWin would just start compositing immediately once the last frame was presented, to have as much time as possible for rendering; this worked pretty well but meant that it almost always rendered too early. On desktop GPUs, compositing can take as little as a few hundred microseconds; if we start compositing 16ms before the deadline, that means we're also unnecessarily increasing latency by roughly 16ms, which makes the system feel less responsive.

For KWin 5.21, Vlad Zahorodnii implemented a scheduling mechanism to do this better: Instead of assuming we always need the whole frame for rendering, KWin measures how long rendering takes and could start compositing closer to the deadline, which reduced latency. However, this strategy could still not get very close to the deadline by default, because it only measured how long rendering took on the CPU, but not how long it took on the GPU.

For KWin 6.0, I implemented the missing part, recording GPU render times. This meant that we could reduce latency more, without causing stutter... Or at least that was the idea. It turns out, render times are *very* volatile at times; KWin's rendering can be delayed by other apps using the CPU and GPU, by the hardware changing power management states, by additional windows opening, by KWin effects starting to render something heavy, by input events taking CPU time, and so on.

As a result, taking a simple average of recent render times wasn't enough, and even taking the maximum wasn't good enough to prevent all the noticeable stutter. Instead, KWin now analyzes past render times for how volatile they are, and starts compositing much earlier if they're volatile, and only moves closer to the deadline when render times are stable and predictable. This will likely be tweaked a few more times as I can collect more render time data from different PCs and optimize that algorithm with it, but so far it works pretty well, and gets us the best of both worlds: High latency when necessary to prevent stutter, and low latency when possible.

So, with these old Intel processors, KWin should now detect that rendering takes long and render times are not very stable, and start rendering as early as possible, and that should fix everything, right? Unfortunately, that's not the whole story.

# Old Intel processors are just too damn slow!

On these old processors, especially when paired with a 4k screen, rendering doesn't just take long, it often takes too long for a frame to be completed after 16.67ms! All the previous improvements were useful, but can't make the hardware faster.

In the very beginning of the post I hinted that this is a Wayland only problem though, so what's going on on Xorg? kwin_x11 has a trick that makes this problem less severe: On the start of each refresh cycle, it starts rendering a frame, even if the last frame isn't done rendering yet. This is called "triple buffering"[^1] because it uses up to three buffers at the same time (two for rendering, one for displaying) and it has two big benefits:
- while the GPU is still working on the last frame, the CPU can already prepare rendering commands for next one. As long as both CPU and GPU individually take less than 16.67ms, you can still get one image rendered for each frame the display can present
- because more frames are being rendered, the driver may increase CPU and GPU clock speeds, which makes rendering faster and might allow for hitting the full refresh rate

However, it also has some caveats:
- it increases latency in general, as rendering is started earlier than necessary
- when rendering takes more than one refresh duration, latency is increased by a whole refresh duration - even if it would only need a single millisecond more for rendering
- to avoid increasing latency for dedicated GPUs, it's only active on Intel GPUs and never used elsewhere
- it's active even on Intel GPUs that have good enough performance to not need it
- when the driver increases GPU clocks because of triple buffering, that may be enough for rendering to be fast enough to not need triple buffering anymore... which means the GPU clocks will reduce again, and frames will be dropped until triple buffering is active again, and that repeats in a cycle. This can, in some situations (like video playback), be more noticeable than a reduced but constant refresh rate.

The goal then was to implement a form of triple buffering that would come with the same benefits, without also having the same shortcomings. First, a few things needed to be patched up to allow for triple buffering to work.

# Fixing prerequisites

The DRM/KMS kernel API currently only allows a single frame to be queued for presentation at a time. When you submit a frame to the kernel, you have to wait until it's done rendering and shown on the screen, before you're allowed to commit the next frame - but for triple buffering we need to queue two frames. Luckily I had already implemented a queue for other functionality with a drm commit thread (check out [my previous post about cursor updates](/wayland/2023/08/29/getting-rid-of-cursor-stutter.html) for details), so this one was mostly taken care of already and only needed minor changes.

The queue wasn't good enough yet though. If KWin's render time prediction is too pessimistic and it starts rendering much earlier than necessary, it could end up rendering two frames that are meant for consecutive refresh cycles, and complete rendering both during the same refresh cycle... which means that GPU power is wasted. Worse, as the refresh rate of apps is coupled to KWin's, apps would try to render twice as fast too! To fix that, frames are now simply delayed until the time they're intended to be displayed.

In order to figure out how long compositing takes on the GPU, KWin uses OpenGL query objects. These are quite useful, but they have three issues for triple buffering:
- query objects only hold a single result. If you start two queries, the last one gets dropped
- if you query a timestamp for commands that haven't finished executing yet, OpenGL will do a blocking wait until that's done
- they're bound to an OpenGL context, which especially complicates things with multiple GPUs

To avoid the last problem, I did a bunch of OpenGL refactors that removed unnecessary global state from OpenGL code, and encapsulated what was left in OpenGL context objects. Render time queries in KWin now store a reference to the OpenGL context object they were created with and handle the context switching for fetching the result themselves, so code using them doesn't have to care a lot about OpenGL anymore. With that done, the other two issues were fixed by simply creating a new query for each frame, which gets queried after the frame is done rendering, and never gets reused.

# Actually implementing triple buffering

After these prerequisites were taken care of, I extended the existing infrastructure for frame tracking, `OutputFrame` objects, to encapsulate more properties of presentation requests and handle render time tracking as well as presentation feedback directly. With all information and feedback for each frame being tracked in a single object, a lot of presentation logic was simplified and allowing multiple pending frames ended up being a relatively simple change in the drm backend.

To make use of that capability, I extended the render scheduling logic to allow render times of up to two frames. It computes a target presentation timestamp by calculating how many refresh cycles have gone by since the last presented frame and how many refresh cycles rendering will take. If there's already a frame pending, it just ensures that the refresh cycle after that is targeted instead of the same one... and that's *almost* it.

Remember how I wrote that displays refresh in a fixed time interval? Well, it's not that simple after all. Timings can fluctuate quite a bit, and that could make KWin sometimes schedule two frames for the same refresh cycle, just for the older one to get dropped again and be a complete waste of energy and even cause stutter. As a fix, when the last frame completes and provides a more accurate timestamp about when the next refresh cycle begins, KWin now reschedules rendering to match it.

This is the state of things in the 6.1.0 release; since then issues on a few systems were reported and more adjustments may still be added - in particular, some sort of hysteresis to not make KWin switch between double- and triple buffering too often.

# The result

With all those changes implemented in Plasma 6.1, triple buffering on Wayland
- is only active if KWin predicts rendering to take longer than a refresh cycle
- doesn't add more latency than necessary even while triple buffering is active, at least as long as render time prediction is decent
- works independently of what GPU you have

In practice, on my desktop PC with a dedicated GPU, triple buffering is effectively never active, and latency is the same as before. On my AMD laptop it's usually off as well, only kicking in once in a while... But on some older Intel laptops with high resolution screens, it's always active and I've been told it's like having a brand new laptop - KWin goes from doing stuttery 30-40fps to a solid 60fps.

It's not just old or slow processors that benefit though, I also tested this on a laptop with an integrated Intel and a dedicated NVidia GPU. With an external display connected to the NVidia GPU, due to some issues in the NVidia driver, multi gpu copies are quite slow, and without triple buffering, the external monitor was limited to 60fps. Triple buffering can't do magic, but KWin now at least reaches around 100-120fps on that setup, which is likely the best that can be done until the driver issue is resolved and feels a lot smoother already.


<br/>

---
<br/>

[^1]: Keep in mind that "triple buffering" means a few different things to different people and in different contexts. I won't go deeper into that mess here; just be aware that it's a loaded term.




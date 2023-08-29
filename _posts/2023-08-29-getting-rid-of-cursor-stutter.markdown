---
layout: post
title: "Getting rid of cursor stutter"
date: 2023-08-29
categories: Wayland
---

# The solution and the problem: atomic modesetting
When it comes to driving displays on Linux there's some legacy APIs that we don't care about, and the drm/kms API. This API isn't exactly *one* API though, it has two variants, *legacy modesetting* and *atomic modesetting*.

*Legacy modesetting* more or less works like this: For every change a display server wants to do, there's a system call that applies this change. There's a system call that changes the display mode, a system call that changes the buffer that is presented, a system call that changes the cursor position and buffer, a system call that changes more specific properties like whether or not variable refresh rate is enabled, a system call that changes the "gamma ramp" of the output and a few more.

Sounds nice and simple, but it also comes with problems. Let's say you're using the gamma ramp for color transformations, and you're starting a game that's using HDR. If you now change the gamma ramp to work for that HDR game and the buffer that's meant to be used with that gamma ramp, then you have a short moment where the gamma ramp is changed while you're still showing the old buffer, so the image will look wrong for a moment. This results in visible glitches, which isn't great and effectively stops Wayland compositors from using these features with the legacy API. There's also some other issues that make the API less than ideal but I won't dive into them here.

The alternative that most Wayland compositors prefer to use (if the driver supports it) is *atomic modesetting.* Like the name suggests, it's a way to apply changes atomically. When you want to change color properties and the buffer at once, you put both into one "atomic commit", and the kernel will ensure that when the user sees the result, they will see both changes at once. It also provides a way of testing the changes without actually applying them, which allows you to build up changes piece by piece before applying them all at once.

However, it's not without problems either. Atomic commits are only applied at the beginning of a refresh cycle and can't be changed once they are submitted to the kernel. In practice this means that if your compositor submits a commit a long time before the start of the next refresh cycle, latency will be high, as it can't change the buffer even if an application provides a new one. That limitation isn't unique to atomic modesetting per se, legacy modesetting has the same problem, but it's made worse by the fact that *all* state changes go through that same API - including the cursor position.

So if something is slowing down the compositor, like a game fully utilizing the GPU for example, then that will slow down the cursor as well. If the compositor misses the refresh cycle deadline for any reason at all, the cursor will stutter and lag, which is obviously far from ideal.

# Why not just fix the API?
Fixing this problem in the kernel would be great, compositors wouldn't need significant changes and could just push changes that don't need to be synchronized with a special flag to indicate that they should be applied ASAP, and all would be good. This has been tried before, but noone has been successful with it so far. It isn't a simple change in the kernel, and realistically would require further API changes as well - for it to be useful for more than merely updating the cursor position, compositors would also need additional feedback for when they can start reusing buffers that were replaced and which buffers actually get presented at which time.

As a result of that, I consider fixing the kernel in this regard as just not possible at the moment.

# Fixing it in KWin
A high level description for the solution to this problem sounds pretty simple: Instead of committing a long time before the next refresh cycle, move the commit closer to the deadline, and before submitting it to the kernel, update it with newer cursor information. The actual implementation however is a lot more complicated.

The first problem to solve was that the main thread of KWin is quite often busy rendering (for the relevant output, for a different output, for screen casting, for offscreen effects etc), handling input events, communicating with applications etc, so reliably doing an atomic commit right before the deadline is just not realistic. To fix this, I introduced a separate thread that tries to submit an atomic commit at most 1.8ms before the deadline. These 1.8ms were determined experimentally and could likely be reduced with more work put into making the scheduling of the thread be more predictable, but this is good enough for now and can be improved later on.

With this infrastructure in place, whenever the cursor gets moved, a new atomic commit is prepared with updated cursor information and tested to make sure it will actually work. If it does work, then the older commit that the thread is holding is switched out for the newer one, which already visibly reduces cursor latency.

The second problem was that when you commit a buffer to the kernel, that buffer comes with some strings attached: In order to prevent glitches, the kernel needs to first wait for rendering to that buffer to complete before showing it to the user. This means that if the main content takes too long to render, the commit will be delayed by one refresh cycle and the cursor will still lag. There's two ways of taking care of this:
1. ensure that compositing just never takes too long. If we always hit the deadline, then neither the cursor nor anything else on the screen will lag
2. if we do miss the deadline, don't commit the buffer used for compositing, but only commit the newer cursor position

I've done some work to improve the first point, but the details around that will have to wait for another time. The critical part is though that while we can reduce the chance for KWin to miss the deadline, we can never completely get rid of it. Sometimes there's suddenly a big GPU load that throws our predictions for how long rendering takes off, sometimes the GPU doesn't increase its clock speeds enough, sometimes it just is too damn slow. So we still need to fix the second point in order to ensure the cursor never stutters.

As a first step in the right direction, I made the thread never submit commits with buffers that were still being rendered to. If a commit isn't ready when the next refresh cycle starts, it gets delayed by KWin instead of the kernel - which allows the thread to update it with newer cursor information until that next refresh cycle comes around, making the missed deadline less noticeable.

To fully solve it though, commits for the cursor need to be completely separate from the rest of the state updates. With that implemented, the commit thread can now end up with a queue like this:

Background (not ready) | Cursor Move 1 (ready) | Cursor Move 2 (ready) | Cursor Move 3 (ready)

It then attempts to simplify the queue by merging commits that only have buffers that are done rendering already.

Background (not ready) | Cursor Move 1 + Cursor Move 2 + Cursor Move 3 (ready)

It also attempts to reorder the queue to move commits that are ready to the front. This doesn't always work, for example it may be that the cursor changes only work on top of the background change already being committed, but not on their own. To ensure we keep a queue that is known to work, it tests the new order of commits, in this case `Cursor Move 1 + Cursor Move 2 + Cursor Move 3` and `Cursor Move 1 + Cursor Move 2 + Cursor Move 3 + Background`. If it doesn't work, then the cursor commits will just have to wait for the `Background` commit to be ready.

If it does work though, the commits get swapped around like so:

Cursor Move 1 + Cursor Move 2 + Cursor Move 3 (ready) | Background (not ready)

and with that done, the thread commits `Cursor Move 1 + Cursor Move 2 + Cursor Move 3` and reschedules the background commit to be submitted in the next refresh cycle instead.

# The result

To test if this actually works I used the [compositor-killer](https://github.com/ascent12/compositor-killer) app from Scott Anderson, which is designed to take a *very* long time rendering each frame and slow down the compositor's rendering to a crawl. In this video I used it with 700000 iterations to fully utilize my GPU and make compositing slow down to 3-4 frames per second, and despite all of that, the cursor stays butter smooth:

<blockquote class="imgur-embed-pub" lang="en" data-id="a/dNkLIFm" data-context="false" ><a href="//imgur.com/a/dNkLIFm"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>

Of course we'll also want to try to prevent apps from dragging down KWin's rendering speed in the first place, but that's not done yet and this post is really long enough already.

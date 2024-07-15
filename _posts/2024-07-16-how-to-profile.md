---
layout: post
title: "How to: Profile your display in the Plasma Wayland session"
date: 2024-07-16
categories: Wayland
---

Profiling displays is already not a super simple thing on its own, but things get more complicated when you try to profile your display in Wayland - profiling applications don't support Wayland yet, some APIs on the compositor side to make it work well are still missing, and there's a general lack of information on the topic. So today I'll show you how to profile your display in the Plasma Wayland session.

I did this in Fedora 40, but you can follow these steps in other distributions as well.

# Step 1: Install DisplayCal and start it
This sounds easy, but
- it's not packaged for Fedora. That's being worked on, but right now it's not an option \| edit: turns out there is a [COPR](https://copr.fedorainfracloud.org/coprs/ngompa/DisplayCAL) for it
- installing it with pip just gave me a bunch of compilation errors, and I haven't figured out how to fix them
- the package on Flathub is *really* old and broken

To work around that, I used distrobox to install the Arch Linux package for DisplayCAL:
```
sudo dnf install distrobox
distrobox create --name archinabox --image archlinux:latest
distrobox enter archinabox
sudo pacman -S displaycal
distrobox-export --app displaycal
exit
```

After running these commands, DisplayCAL can be started from any app launcher, like Kickoff or KRunner.

# Step 2: Setup
To get correct measurement results, the compositor needs to pass the pixel data from the profiling app directly to the display, and not do any color management itself. This will be automated at some point, but for now you need to manually ensure that
- HDR is disabled
- the color profile of the display is set to "None" in the display settings
- night light is off, or at least suspended in the system tray
- all KWin effects that modify colors, like the color blindness correction effect, are disabled
- if you're on a new-ish AMD laptop and want to profile the internal display, that you're either plugged in to a power source, or have the power profile set to performance, to disable a power saving feature that changes the colors

![display settings](/assets/how to profile/display settings.png)

<!-- ![night light and brightness](/assets/how to profile/night light and brightness.png) -->

Now start DisplayCal and head to the Calibration tab. Here it's important to set the tone curve to "as measured", and untick interactive display adjustment, as those don't work correctly right now and *will* mess up the profile.

![DisplayCAL profiling tab](/assets/how to profile/DisplayCAL profiling tab.png)

You've done everything correctly if the button on the bottom of the application shows "Profile only".

Last but not least, you also need to adjust the display settings to what you want to use with the profile later, as the profile is only correct for one specific set of display settings. This includes the brightness of the display!

# Step 3: Profile
In the profiling tab of DisplayCAL, select your desired settings - in most cases the default will be sufficient - and click "Profile only". When it asks if you want to continue with the current calibration curves, select "use linear calibration instead" and de-select "embed calibration curves in profile". Then put the colorimeter in the center of the screen, and let it do its thing.

![DisplayCAL warning](/assets/how to profile/linear calibration instead.png)

Once it's done, it'll ask you to install the profile. Installing it will not automatically enable that profile to be used, but it'll save the profile in `~/.config/color/icc/devices/display/` and you can select that file in the display settings.

# Step 4: Verification (optional)
If you'd like to make sure the profile is correct or accurate enough, you can use DisplayCAL to verify the result. Make sure you've set the profile in the display settings, switch to the verification tab in DisplayCAL and select your newly created profile in the "settings"

Here again, because DisplayCAL doesn't support Wayland yet, you need to adjust a few settings for everything to work correctly. You need to select the simulation profile "Rec.709 ITU-R BT.709", select "Use simulation profile as display profile" and set the tone curve to "Gamma 2.2". Afterwards, click on "Measurement report", choose a location to save it in, put the colorimeter in the center of the screen again and wait for it to complete.

![DisplayCAL verification tab](/assets/how to profile/DisplayCAL verification tab.png)

Don't be alarmed if the result says the whitepoint is wrong, this is simply caused by DisplayCAL assuming we want to target the whitepoint of the simulation profile, which doesn't necessarily match the whitepoint of your display.

# What about calibration though?
To calibrate the display, that is, to adjust brightness, tone curves for non color managed applications and the whitepoint of the display, DisplayCAL uses an X11 API to set the gamma lookup tables of the GPU. That API doesn't work in the Wayland session and the profiling process doesn't handle that situation properly, which is why all calibration needs to be disabled for the created profile to be correct.

DisplayCAL (or ArgyllCMS, which does the actual profiling) could add support for applying a lookup table in the application instead of having the compositor do it, but we can also handle calibration entirely on the compositor side instead, which offers a bit more flexibility.

Changing the tone curves for non color managed applications doesn't make sense in the Plasma Wayland session, as all windows are always color managed, so that part is already dealt with. Adjusting the brightness on screens that don't have any native means of brightness control is already implemented for Plasma 6.2, and I have a working proof of concept for changing the whitepoint of the display without needing a new ICC profile too, so we should be at feature parity soon. I'll talk more about these adjustments in a future post.

---
layout: post
title:  "How I Installed a 3rd Party Android TV Launcher"
date:   2021-06-22 09:45:00 -0400
categories: podman buildah container
---
There was a recent update to Android TV which added advertisements to the home screen of my Nvidia Shield. This was an unwanted surprise but luckily it was not too difficult switching to a 3rd party launcher. I am recreating the steps that I took with an Android TV device in the Android emulator.

![Android TV App Screen](/photos/androidTV/androidTV.png)

## Setup
I would like to install [ATV Launcher](https://play.google.com/store/apps/details?id=ca.dstudio.atvlauncher.free&hl=en_US&gl=US) however, I do not log into my Google Account on my Nvidia Shield device so this is a problem. As of the time of this writing, ATV Launcher is also not on [APKMirror](https://www.apkmirror.com/). I will adapt and overcome.

I downloaded the ATV Launcher application onto my Pixel phone via the Play Store. From there I enabled USB debugging, plugged my phone into my computer, and used adb to find and pull the package.

```bash
# List packages and find the launcher
[ethan@fedora ~]$ adb shell pm list packages | grep -i atv
package:ca.dstudio.atvlauncher.free

# Get the full path to the package
[ethan@fedora ~]$ adb shell pm path ca.dstudio.atvlauncher.free
package:/data/app/~~uOaPvbC3hrgIqFeWGPyV-Q==/ca.dstudio.atvlauncher.free-27cidHv-2G3iZbmOTDxUAw==/base.apk

# Pull the package off of the phone
[ethan@fedora ~]$ adb pull /data/app/~~uOaPvbC3hrgIqFeWGPyV-Q==/ca.dstudio.atvlauncher.free-27cidHv-2G3iZbmOTDxUAw==/base.apk atv_launcher.apk
/data/app/~~uOaPvbC3hrgIqFeWGPyV-Q==/ca.dstudio.atvlauncher.free-27cidHv-2G3iZbmOTDxUAw==/base.apk: 1 file pulled, 0 skipped. 36.3 MB/s (2043758 bytes in 0.054s)
```

## Installing the Launcher on the Shield TV
Now that I have the launcher package, I can install it to the Android TV device.

I connected to the Android TV device via adb and installed the launcher app that I previously pulled from my phone. The new launcher does not start by default but can be started manually via the app list. This was not good enough, I wanted the 3rd party launcher to run by default. To force the new launcher I had to forcefully disable the default.

```bash
# Install the new launcher
[ethan@fedora ~]$ adb install atv_launcher.apk
Performing Streamed Install
Success

# Find the default launcher
[ethan@fedora ~]$ adb shell pm list packages | grep -i launcher
package:ca.dstudio.atvlauncher.free
package:com.google.android.tvlauncher

# Disable the default launcher
[ethan@fedora ~]$ adb shell pm disable-user --user 0 com.google.android.tvlauncher
Package com.google.android.tvlauncher new state: disabled-user
```

![The Result](/photos/androidTV/newLauncher.png)

The end result with a few favorite applications installed and a wallpaper applied. Perfect! If I ever need to re-enable the default launcher I can do so with:
```bash
[ethan@fedora ~]$ adb shell pm enable com.google.android.tvlauncher
Package com.google.android.tvlauncher new state: enabled
```

## Conclusion
Although this change to Android TV was unwelcome, I enjoyed learning about my device and customizing it to my needs. At least customiztion is still possible but when will the next update roll out that I have to react to?

The following threads on [HN](https://news.ycombinator.com/item?id=27643208) and [Reddit](https://old.reddit.com/r/ShieldAndroidTV/comments/o6o5dz/new\_guide\_use\_a\_custom\_launcher\_shield\_tv\_with/) were good resources.
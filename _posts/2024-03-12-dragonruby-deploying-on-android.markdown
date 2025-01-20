---
layout: post
title: "DragonRuby: Deploying on Android"
date: 2024-03-12 14:22
comments: true
categories: 
---

This is a little less-polished-than-usual post about how to build/install Android applications with [DragonRuby **Pro**](https://dragonruby.org/toolkit/game). on a Linux system. The higher tiers of features in DragonRuby tend to be less well-documented, so here is a bit of a braindump on getting games running on a real Android device.

(Mostly to remind myself how I did all this.)

## Building the Package

_To be clear, you will need the "Pro" version of DragonRuby to follow this guide._

Running

```
dragonruby-publish --package
```

will generate binaries and packages for all supported platforms and dump them in `builds/`.

## Signing

### Creating a Keystore

Following the DragonRuby documentation, create a keystore file like this:

```
keytool -genkey -v -keystore YOUR_APP.keystore -alias your_app_name -keyalg RSA -keysize 2048 -validity 10000
```

This will generate a file called `YOUR_APP.keystore`.

`keytool` is probably already on your system, but if not you'll need to install a Java JDK package using your system tools.

### Getting apksigner

To install the package on your Android device, you'll need to sign the `.apk` build using `apksigner`. I'm only going to cover one specific way of getting this program on Linux. Your experience may vary.

The easiest way to get `apksigner` is probably to install [Android Studio](https://developer.android.com/studio/) and go from there. However, I prefer doing things the hard way (and not installing a whole IDE to get one binary...)

First, go to [https://developer.android.com/studio](https://developer.android.com/studio/) and scroll _allll_ the way to the bottom to "Command line tools only". Grab the .zip file from there.

Unzip it and find the `sdkmanager` binary, likely in `latest/bin`.

Run `./sdkmanager --list | grep build-tools` to find the latest version of `build-tools`.

Then run something like `./sdkmanager --install "build-tools;34.0.0"` to install.

The files will probably end up somewhere like `../../../build-tools`. In there you'll find `apksigner`!

### Signing the Package

```
apksigner sign -ks YOUR_APP.keystore builds/YOUR-GAME-android.apk
```

You'll need to figure out paths for `apksigner`, `YOUR_APP.keystore`, etc.

See the [official docs](https://docs.dragonruby.org/#/guides/deploying-to-mobile?id=deploying-to-android) for more, especially if publishing to the Google Play store.

## Installing on Device

### Enable Debug Mode

On your Android device setup developer options, enable [USB debug mode](https://developer.android.com/studio/debug/dev-options), and plug your device into your computer.

On Linux, you may need to figure out permissions. [Here's a post with instructions](https://www.janosgyerik.com/adding-udev-rules-for-usb-debugging-android-devices/) that worked for me.

### Getting adb

The `adb` tool can be downloaded as part of the [SDK Platform tools here.](https://developer.android.com/tools/releases/platform-tools#downloads.html)

### Install the APK

```
adb install builds/your-game-android.apk
```

## Viewing Logs

To view logs from the device:

```
adb logcat -e your-game
```

where `your-game` is the `gameid` or `packageid` configured in `mygame/metadata/game_metadata.txt`.

## Remote Hotload

Building, signing, and installing packages becomes a bit painful if you are doing all that during development.

How about hot-loading code on your Android device just like you can on your development machine?

### Setup

When running `dragonruby`, it opens up a webserver on port 9001. Besides what's obviously visible on the webpage, it's also how your device can connect and load code dynamically.

For that all to work, your development machine and Android device need to be on the same network, and your firewall needs to allow TCP connections on port `9001`.

To verify it's working, try opening up your development machines IP on port 9001 from your Android device (e.g., visit `https://YOUR.DEV.IP:9001` in a browser).

### Building

Run

```
dragonruby-publish --package-with-remote-hotload
```

To create a hotloading version of the game, then sign+install like above.

### Running

Make sure your are running `dragonruby` locally on your development machine.

Then open your game on your mobile device - it should flash "Remote hotload enabled" at the bottom if it has been built properly. This does _not_ ensure it actually connected to the development server, though!

To test, try making a visible change to a file on your local machine and see if the change is reflected on the Android device.

Somehow, magically, the hot-loaded changes will persist even through restarts. However, only changes made while the development server is running will be picked up.

## Bonus: Detecting the "Back" "Button"

I am old, so I still use the virtual "back" button on Android.

In DragonRuby, this can be detected with

```ruby
args.inputs.keyboard.key_down.ac_back
```

For example:

```ruby
if inputs.keyboard.key_down.ac_back
  gtk.request_quit
end
```

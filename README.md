# AndroidStudioEmulator-acquireAppsData <!-- omit in toc -->

This page shows how to use the the adb command line tool to install apps inside the Android emulator and then how to get the files produced by them to later perform a digital forensics analysis.

|         |           |
| :-:     | :--       |
| ![by-nc-sa](https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png) | This work is licensed under a [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-nc-sa/4.0/) |

## Table of Contents <!-- omit in toc -->

- [1. Install apps](#1-install-apps)
  - [1.1 Exercise](#11-exercise)
- [2. Important directories](#2-important-directories)
  - [2.1 Public data](#21-public-data)
  - [2.2 Private data](#22-private-data)
  - [2.3 APK Files](#23-apk-files)
- [3. Extract data](#3-extract-data)
  - [3.1 Exercise](#31-exercise)


> ***NOTE***
> 
> These exercises require the installation of Android Studio Emulator:
> 
> - [Set up Android Studio Emulator with GUI](https://labcif.github.io/AndroidStudioEmulator-GUIconfig/)
> - [Set up Android Studio Emulator **without** GUI (cmd-tools)](https://labcif.github.io/AndroidStudioEmulator-cmdConfig/#5-exercise)


## 1. Install apps

A system-image without `_playstore` won't have access to the Google Play Store. So, to install apps you need to go to a website, like <https://www.apkmirror.com/> and download the `APK` file of the app you want to install.

Use the `adb` commands to connect to the emulator:

```console
user@linux:ANDROID_HOME/platform-tools$ adb devices
List of devices attached
emulator-5554   device
```

Then, inside the directory where you downloaded the APK file use `adb install <file>.apk`, for example:

```console
user@linux:ANDROID_HOME/platform-tools$ adb install com.google.android.apps.authenticator2_5.10.apk
Success
```

> ***NOTE***
>
> It is also possible to install an alternative to Google's Play store. One of them is [F-Droid](https://f-droid.org/), but there are many others. The install process is the same as described above.

### 1.1 Exercise

Download and install the following apps:

- [Google Authenticator](https://www.apkmirror.com/?s=google+authenticator)
- [Termux](https://github.com/termux/termux-app/releases)

Once the apps are installed we can interact with then to generate some data.

Open `Google Authenticator` and click the `+`, then choose `Insert key`, type the name `Test` in the account name,  in the key filed type `asdfghjklqwertyuiop` and click `Add`. If this operation is successful you should see a 6 digit OTP number.

## 2. Important directories

Apps generate data into two locations:

- **Public data directory** -- data that is available even on non-rooted devices
- **Private data directory** -- data that is only available with root
- **APK files** -- binary files of the installed app

Next we show the location of each of these directories.

### 2.1 Public data

Bellow are the commands to enter the public data directory `/storage/emulated/0/Android/data/`:

```console
user@linux:ANDROID_HOME/platform-tools$ adb shell
generic_x86_64_arm64:/ $ cd /storage/emulated/0/Android/data/<app dir>
```

However, there are 4 links that can be used as alternative paths to `/storage/emulated/0/` and, therefore, to reach the public data dir:

```text
/
├── sdcard/ → /storage/self/primary/
├── mnt/
│   ├── sdcard/ → /storage/self/primary/
│   └── user/0/primary/ → /storage/emulated/0/
└── storage/
    ├── self/primary/ → /mnt/user/0/primary/
    └── emulated/0/Android/data/
```

So, you can use also a shorter path, for example:

```console
user@linux:ANDROID_HOME/platform-tools$ adb shell
generic_x86_64_arm64:/ $ cd /sdcard/Android/data/<app dir>
```

### 2.2 Private data

Bellow are the commands to enter the private data directory `/data/data/` that is only available with root privileges (notice the change from `$` to `#` in the prompt):

```console
user@linux:ANDROID_HOME/platform-tools$ adb shell
generic_x86_64_arm64:/ $ su
generic_x86_64_arm64:/ # cd /data/data/<app dir>
```

### 2.3 APK Files

Some software tools that perform automated static analysis of `APK` files, like [MobSF](https://github.com/MobSF/Mobile-Security-Framework-MobSF), don't support multi-part pakages. A multi-part package (or bundle) is an `APK` with several `APK` files inside. One way to solve this problem is:

- install the app, then
- copy the main `APK` file: `/data/app/<app dir>/base.apk`

> **_NOTE_**
>
> Since Android 8 the `<app dir>` will contain a sufix with several random characters encoded in Base64, for example: `/data/app/com.google.android.apps.authenticator2-h-U8jQG_YWGWCGK3XeVWog==/base.apk`.
>
> On Android 11 the `<app dir>` is inside an extra dir also with several random characters encoded in Base64, for example: `/data/app/~~ZILWitQqXEB3uiTnOKhlvg==/com.google.android.apps.authenticator2-h-U8jQG_YWGWCGK3XeVWog==/base.apk`
> 
> These Base64 encoded characters are used to prevent apps to get a list of all the installed apps in a device.

## 3. Extract data

1. Connect to the Android emulator and follow the steps bellow to create a `tgz` file with the contents of the private directory af an app:

```console
user@linux:ANDROID_HOME/platform-tools$ adb shell
generic_x86_64_arm64:/ $ su
generic_x86_64_arm64:/ # cd /data/data/
generic_x86_64_arm64:/data/data/ # tar -cvzf /sdcard/Download/<compressed-filename>.tgz <app-folder>/
generic_x86_64_arm64:/data/data/ # exit
generic_x86_64_arm64:/ $ exit
```

If there are file names, or directories with *spaces* on their names the regular `tar` command (as shown above) will fail. Instead try this approach:

```console
generic_x86_64_arm64:/data/data/ # find <app-folder> -print0 | tar -cvf /sdcard/Download/<filename>.tar --null -T -
```

> **_NOTE_**
>
> Always use `tar`, don't copy the files directly to your computer, specially if you're using Windows, because you might loose information:
> 
> - For example, the files `file.txt` and `File.txt` are two different files under Linux (Android uses a Linux kernel) but are the same file under Windows
> - Windows doesn't recognizes Linux's links
> - There are some characters that are allowed in Linux file names, but that aren't supported on Windows


2. Copy the `tgz` file into your computer for analysis

```console
user@linux:ANDROID_HOME/platform-tools$ adb pull /sdcard/Download/<compressed filename>.tgz
/sdcard/Download/<compressed filename>.tgz: 1 file pulled. 0.1 MB/s (180 bytes in 0.010s)
```

3. If you're using Windows set up first a Linux VM, and copy the `<compressed-filename>.tgz` inside the VM to avoid losing data during the decompression (see the note above)

4. Decompress the file with `tar -xvzf <compressed-filename>.tgz`, or other tool that supports `.tgz` files, and start the analysis


> **_NOTE_**
>
> To automate the capture process check the [Android App Acquisition Script](https://github.com/labcif/AndroidAcquisitionScript) from LabCIF. It's a `bash` script that already has support for file and folder names with spaces on it, and adds the app version and a timestamp to the acquired file name. Here's an example:
>
> ```console
> user@linux:~$ ./aquisition.sh us.zoom.videomeetings
> [Info ] Does "us.zoom.videomeetings" exist?
> [Info ] Yes!
> [Info ] Getting the version of "us.zoom.videomeetings"...
> [Info ] The version is "5.9.3.4247".
> [Info ] Copying data from "us.zoom.videomeetings" version "5.9.3.4247" ...
> removing leading '/' from member names
> data/user_de/0/us.zoom.videomeetings/
> ...
> [Info ] Copy terminated.
> [Info ] Compressing "us.zoom.videomeetings--u0-v5.9.3.4247--2022.02.10T15.52.57.tar" ...
> [Info ] Compressing terminated.
> [Info ] Copying to local storage...
> /sdcard/Download/us.zoom.videomeetings--u0-v5.9.3.4247--202....tar.gz: 1 file pulled. 25.8 MB/s (4590475 bytes in 0.170s)
> [Info ] Copy terminated.
> [Info ] Cleaning acquisition files from phone...
> [Info ] Clean terminated.
> ```


### 3.1 Exercise

Extract the private directory created by Google Authenticator and do a forensic analisys of the data.

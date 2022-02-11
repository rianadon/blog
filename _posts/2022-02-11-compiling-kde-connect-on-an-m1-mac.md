---
layout: post
title:  "Compiling KDE connect on an M1 Mac"
date:   2022-02-11
---

[KDE Connect](https://kdeconnect.kde.org/) is a beautiful app that syncs non-Apple devices to your Mac. The only problem is that building it on a Mac is a little ... intense. Especially on arm64 / Apple Silicon architecture.

So I thought I'd write up some instructions in case anyone else runs into the same issues.

It happens that the build infrastructure, which uses the [Craft](https://community.kde.org/Craft) build system, is set up for x86_64 architecture. When it downloads packages from the internet to speed up the build, it downloads the x86 versions. And when there is no download and your Mac compiles programs, it (sometimes) does so in the arm64 architecture, and these two cannot be linked!

In fact, you might get an error like this when Craft compiles dbus:
```
building for macOS-arm64 but attempting to link with file built for macOS-x86_64
```

# The fix

An arm64 build would be nice; however, I could not figure out how to make QT libraries compile for arm64. So the solution is to force a build for x86_64.

Specify `CFLAGS` and all variations to add the `-arch x86_64` flag so any `clang` build command builds for x86:

```bash
CFLAGS="-arch x86_64" CXXFLAGS="-arch x86_64" OBJCFLAGS="-arch x86_64" craft kde/applications/kdeconnect-kde
```

Make sure of course you have installed Craft and sourced your environment file (you can follow the instructions on the [Building KDE Connect on MacOS page](https://community.kde.org/KDEConnect/Build_MacOS#Setting_up_Craft_environment)).

If you are using Homebrew, you might have conflicts with Craft trying to use some of your arm64 libraries, since both the Craft libraries and Homebrew libararies are linked with pkg-config. I had to rename gcrypt and libepoxy temporarily as the build was running, since these are optional for some of the KDE Connect dependencies:
- Rename `/opt/homebrew/opt/libgcrypt` to `/opt/homebrew/opt/libgcrypt-rename` (optional dep for qca)
- Rename `/opt/homebrew/Cellar/libepoxy/` to `/opt/homebrew/Cellar/libepoxy/-rename` (optional dep for kdeclarative)

Once you successfully compile KDE Connect, create a package with Craft then install it!

```bash
craft --package kde/applications/kdeconnect-kde
```

# Making changes with a development repo

The most straightforward method is to use the `craft` command to continue building the source code. First find where `craft` downloaded the source code. This should be something like `~/CraftRoot/build/kde/applications/kdeconnect-kde/work/kdeconnect-kde-21.12.2`.

You are going to replace this directory with the latest source code. Delete the `kdeconnect-kde-21.12.2` folder, then clone the repository into the same location and rename the folder to `kdeconnect-kde-21.12.2` (or whatever the `kdeconnect-kde-version` folder was named before).

Then you can run the two commands below to compile and package KDE Connect. Drag the application into your Applications folder once the packaging finishes, and you can run it as usual!

```
craft --compile --install --qmerge kde/applications/kdeconnect-kde
craft --package kde/applications/kdeconnect-kde
```

## Making speedier changes

> ⚠️ Some functions that required shared libraries (like loading SMS messages) do not work with this method.

Craft takes a while to package changes which is not ideal, and I also don't like working in a folder 7 levels down from my home directory. So below is my super-hacky development setup:

1. Clone the KDE Connect repo anywhere.
2. Source your craft environment: `source ~/CraftRoot/craft/craftenv.sh`
3. `cd` to where you cloned your repo. Sourcing your craft env changes your current directory!
4. Make a build directory: `mkdir build`
5. Navigate to your build directory: `cd build`
6. Set up Makefiles: `cmake ..`
7. Compile and install KDE Connect: `make install`

At this point, you will have a KDE Connect app installed to your `/Applications/KDE` folder (this is different than where `craft` installs)! However, you won't be able to run it because this application folder is missing the necessary frameworks and resources.

To fix this, copy all the files from the Craft installation that do not exist in your new installation:

```bash
cp -n -r /Applications/kdeconnect-indicator.app/Contents/ /Applications/KDE/kdeconnect-indicator.app/Contents/
```

Now launch a new terminal tab, source your craft environment, and run:

```bash
/Applications/KDE/kdeconnect-indicator.app/Contents/MacOS/kdeconnect-indicator
```

You'll need to be inside your craft environment to launch the app because of how the libraries have been linked. Use this new terminal tab to launch KDE Connect and the original to re-run `make install` whenever you make a change to the source code.

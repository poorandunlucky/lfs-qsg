# Lua Flash Store (LFS) Quick Start Guide

## Overview

The Lua Flash Store is a part of the NodeMCU firmware that allows it to run Lua applications directly from ROM, instead of having your modules take-up space in RAM.  The advantage of this should already be obvious, as RAM on ESP SOCs can quickly become a precious, and rare commodity.

## Basic Process

First, you have to use a LFS-enabled firmware.  You can either build this yourself, or use the [nodemcu-builder service](https://www.nodemcu-build.com/) but a fixed, and finite amount of flash storage has to be set aside for the LFS by the firmware.  Second, ensuring that your firmware is LFS-enabled, you have to compile your Lua code into a binary LFS image, again: either locally, or using a [contributed cloud service](https://blog.ellisons.org.uk/article/nodemcu/a-lua-cross-compile-web-service/).  Third, and maybe last, you need to upload this binary to flash using your preferred method, probably nodemcu-uploader, or ESPlorer, and import it to LFS with the `node.flashreload(‘file’)` command.  If you want, you can, finally, delete this file from flash, as it’s no longer needed.

Your code is now persistently stored in LFS, from where it can also be ran, so the space you lost from SPIFFS to create the LFS is re-gained, and since NodeMCU can run your compiled Lua code directly from LFS, you can also load all your modules stored there – up to 128 KB in size – without taking any RAM.  If you have a locally-installed toolchain (which you should, in my opinion... it’s really very simple), this process can easily be scripted, and seamlessly incorporated to your development cycle.

## What’s different?

Once the LFS image has been imported, the only thing you will need to do is run a command at every boot to re-initialize a package loader that will look in LFS before or after the default loader has tried SPIFFS, so that you can use `require()` as usual.  That’s the only thing.

## Loading modules from LFS

While it’s possible and not very difficult to run code that’s not part of a module from LFS, this example focuses on loading modules so as to keep things simple.

When loading a module, one runs the `require ( ‘name’ )` function, which calls a package loader that, in turn, will look for `name.lua`, or `name.lc` from SPIFFS, and load the module of the same name if it’s found in that file.  To load modules from LFS, we need to add a package loader that will look there, before, or after the default package loader has looked through SPIFFS.  This can be done in a `_init.lua` file that’s compiled in with the rest.

## _init

Optionally, you can put an initialization file in LFS that will be parsed at every boot before `init.lua` if it’s present, and instead of `init.lua` (but you can call it from `_init` with `dofile()`).
There is an [`_init.lua`](https://github.com/nodemcu/nodemcu-firmware/blob/master/lua_examples/lfs/_init.lua) file included with the LFS sample code provided in the source tree, but it’s somewhat obfuscating to me, so I decided to include another [example _init file](_init.lua.example).

Essentially: it creates a package loader that will look for modules in LFS after the default loader has searched the SPIFFS partition, that will allow users to call `require()` as usual, and it will try to run `init.lua` (the one from SPIFFS) if it’s there.  Up to you to customize it to suit your needs.

This is it for the things you have to do differently.  Once the package loader has been initialized by the `_init` file in LFS (or the `init.lua` file in SPIFFS if you don’t want to use `_init`), you can load your modules using `require()` as you’ve always done, and there’s nothing else to do.

## Dummy Strings

Another use that was found for the LFS was to preload common string constants from the firmware so they remain stored in LFS rather than get loaded into RAM.  This can save a few kilobytes of RAM, and only entails running a command, and pasting the output in a file that gets compiled with the LFS image (as a variable assignment in the `_init` file, maybe).  Please see the [dummy strings](https://github.com/nodemcu/nodemcu-firmware/blob/master/lua_examples/lfs/dummy_strings.lua) example file from the sample code for more information.

## LFS Step-by-Step

- The first thing is to compile the firmware with LFS enabled, and its size specified.  This can be done by uncommenting //DEFINE LFS_ENABLE from [user_config.h]( ), or selecting the appropriate options from the drop-down menus from the cloud build service.

- Second, you need to compile your code into an LFS image.  You can do this locally using `luac.cross -f` (`luac.cross.int` for integer builds) in the root directory of the nodemcu-firmware source tree (built by default), or using the [contributed service]().  For instance, if you put your modules in the “module” folder of your project directory, you could run `luac.cross -f -o lfs.img modules/*.lua` to compile your Lua code into a LFS binary image; see `luac.cross --help` for usage information.

- You then need to upload your binary image to your ESP’s flash storage, and import it into firmware.  This can be done with your favourite tool, probably nodemcu-uploader, or ESPlorer.  If using nodemcu-uploader, the command would be `nodemcu-uploader --port /dev/usb1.1 upload lfs.img`.  Afterwards, using a terminal, simply call `node.flashreload ( ‘lfs.img’ )`, and the ESP will reboot if successful.

- Assuming you created your LFS image with an [`_init`](_init.lua.example) file that contains the function, and calls from the included example, there is nothing else to do.  The package loader will be created, and your `init.lua` file from SPIFFS will be parsed as usual.  On the matter, if you need to work on your modules, unless you changed the package loader’s index in `_init`, SPIFFS will have priority over LFS, so those modules will be loaded first by `require()` if they exist.

## Final Words

Despite obviously being a powerful, and important implementation in the NodeMCU firmware, I hope that I have made the LFS seem simple, and easy to use, and that your creativity will continue to expand, along the possibilities the ESP SOC and NodeMCU firmware offer you.

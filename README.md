# firmwared manifest - workspace for building firmwared
![Current travis build status](https://travis-ci.org/ncarrier/firmwared-manifest.svg?branch=master)
## Overview

*firmwared* is a Daemon responsible of spawning drone firmware instances in
containers.  
This document is in markdown format and is best read with a markdown viewer. In
command line, one can use `pandoc README.md | lynx -stdin`.

## Quick start

### Needed packages

For debian-based distributions (tested on debian 8):

        $ sudo apt-get install curl git build-essential pkg-config libssl-dev \
                libapparmor-dev libblkid-dev liblua5.2-dev

### The *firmwared* workspace uses google repo and parrot alchemy

        $ mkdir -p ~/bin
        $ export PATH=~/bin:$PATH
        $ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
        $ chmod a+x ~/bin/repo

### Create a firmwared workspace

        $ mkdir firmwared
        $ cd firmwared
        $ repo init -u https://github.com/ncarrier/firmwared-manifest.git
        $ repo sync
        $ . scripts/setenv
        $ bb all final -j5

### Test firmwared

        $ cp packages/firmwared/examples/firmwared.conf .
        $ # adapt the config to your needs, for example :
        $ sed -i "s#base_dir = .*#base_dir = \"$(pwd)/\"#g" firmwared.conf
        $ mkdir -p firmwares mount
        $ sudo addgroup --system firmwared
        $ sudo usermod -a -G firmwared $USER

You need to unlog/relog for the user to belong to firmwared's group.  

Then as root:

        # . Alchemy-out/linux-native-x64/final/native-wrapper.sh
        # firmwared firmwared.conf

Firmwared should start and stay in the foreground, if it fails, fix the problems
it complains about.  
Then in another terminal:

        $ . scripts/setenv
        $ fdc ping
        PONG

Firmwared just said it was alive.
*fdc* is a lightweight client for firmwared.
For that reason, using *fdc* requires that firmwared is running.
We need now to prepare a firmware (a rootfs), which we will instanciate later.
An example minimal rootfs, with an adb server, is provided for testing:

        $ tar xf packages/firmwared/examples/example_firmware.tar.bz2
        $ fdc prepare firmwares $(pwd)/example_firmware.ext2

Now we need to prepare an instance of the firmware just registered.
We will then play a little with it:

        $ fdc prepare instances <tab>
        $ fdc start <tab>
        $ adb connect $(fdc get_config net_first_two_bytes)$(fdc get_property instances <instance_name> id).1:9050
        $ adb shell
        BusyBox v1.20.2 (2015-06-12 11:26:38 CEST) built-in shell (ash)
        Enter 'help' for a list of built-in commands.

        root@:/ # <CTRL+d>

Now we will cleanup all:

        $ fdc kill <tab>
        $ fdc drop instances <tab>
        $ fdc drop firmwares <tab>
        $ fdc quit
        firmwared said: BYEBYE

You can obtain more help with:

        $ fdc commands # be sure to have restarted firmwared before :)
        $ fdc help <command>
        $ man ./packages/firmwared/man/{fdc.1,firmwared.1,firmwared.conf.5}

## automated tests

Some components of the workspace come with automated tests.
This section documents how to run them.

### fusion

At the root of the firmwared workspace, create a file named
**Alchemy-debug-setup.mk**, containing the following line:

        TARGET_TEST := 1

Cleanup and rebuild the whole workspace:

        $ rm -rf Alchemy-out
        $ bb all final -j5
        $ . scripts/setenv

The tests are built-in the resulting libraries, you can run the with:

        $ ./Alchemy-out/linux-native-x64/final/usr/lib/libutils.so
        $ ./Alchemy-out/linux-native-x64/final/usr/lib/librs.so

Tests for libioutils and libpidwatch must be ran as root:

        # . ./Alchemy-out/linux-native-x64/final/native-wrapper.sh
        # ./Alchemy-out/linux-native-x64/final/usr/lib/libpidwatch.so
        # PROCESS_TEST_SCRIPT=./packages/fusion/libioutils/tests/test.process \
                ./Alchemy-out/linux-native-x64/final/usr/lib/libioutils.so

### firmwared

The automated tests need:

1. that no firmware is registered in firmwared, drop them with
`fdc drop firmwares`
2. that no instance is registered in firmwared, kill them with `fdc kill`, then
drop them with `fdc drop instances`
3. that **scripts/setenv** file has been sourced in each of the terminals you
will use, or more precisely,
**./Alchemy-out/linux-native-x64/final/native-wrapper.sh**.

In one terminal, run firmwared via the script provided, which makes it be
restarted until the end of the tests, if it quits:

        # ./packages/firmwared/tests/run_firmwared.sh

In another window, as a normal user, member of the **firmwared** group:

        $ ./packages/firmwared/tests/run_all.sh

Invidual tests in the firmwared/tests directory can be ran, and will return 0
on success and 1 on error.

# firmwared manifest - workspace for building firmwared

## Overview

*firmwared* is a Daemon responsible of spawning drone firmware instances in
containers.  
This document is in markdown format and is best read with a markdown viewer. In
command line, one can use `pandoc README.md | lynx -stdin`.

## Quick start

### The *firmwared* workspace uses google repo and parrot alchemy

        $ mkdir -p ~/bin
        $ export PATH=~/bin:$PATH
        $ curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
        $ chmod a+x ~/bin/repo

### Create a firmwared workspace

        $ mkdir firmwared
        $ cd firmwared
        $ repo init -u git@github.com:ncarrier/firmwared-manifest.git
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
We need now to prepare a firmware (a rootfs), which we will instanciate later.
An example minimal rootfs, with an adb server, is provided for testing:

        $ tar xf packages/firmwared/examples/example_firmware.tar.bz2
        $ fdc prepare firmwares $(pwd)/example_firmware.ext2

Now we need to prepare an instance of the firmware just registered.
We will then play a little with it:

        $ fdc prepare instances <tab>
        $ fdc start <tab>
        $ adb connect $(fdc get_config net_first_two_bytes).$(fdc get_property instances <instance_name> id).1:9050
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

        $ fdc commands
        $ fdc help <command>
        $ man ./packages/firmwared/man/{fdc.1,firmwared.1,firmwared.conf.5}

+++
date = "2015-10-24"
title = "Upstart & Go"
slug = "upstart-go"
+++

These scripts are system jobs for upstart and are to be placed in /etc/init.

**app-worker.conf**

    description     "App-Worker"
    author          "Adnaan"

    start on (net-device-up
              and local-filesystems
              and runlevel [2345])

    stop on runlevel [06]
    respawn

    instance $PORT

    script
    	#initramfs provides early userspace
    	exec 2>>/dev/.initramfs/app-worker.log
    	set -x
    	export APP="/root/goworkspace/src/myappcode"
    	#change directory or go won't read the web app resources
    	chdir $APP
    	#execute
    	exec sudo $APP/app --port $PORT
    end script
<!--more-->
The above script can be directly used as such:

Start a worker

    service app-worker start PORT=8085

Stop a worker

    service app-worker stop PORT=8085

But what if need a bunch of workers. Can't go starting them individually now, can we?

**app-pool.conf**

    description "App Pool"
    author       "Adnaan"

    start on (local-filesystems and net-device-up and runlevel [2345])

    stop on shutdown


    pre-start script
    	#initramfs provides early userspace
    	exec 2>>/dev/.initramfs/app-pool.log
    	set -x

    	start app-worker PORT=8085
    	start app-worker PORT=8086
    	#add more jobs..
    end script

Start a Pool

    service app-pool start

Stop the Pool

    service app-pool stop

The above won't have any effect since it restarts the workers again. We need to individually stop the workers. Let's find our workers.

    initctl list | grep "app"

+++
date = '2025-11-18T18:40:29-06:00'
draft = false
title = 'Darwin_notifications'
+++

Just a quick little update, while reverse engineering `notifyd` to figure out how I could enumerate available notifications on macOS, I stumbled upon a dump path reachable by two places. The dump writes a lot of useful information about the notifyd state to a provided file descriptor (or a default one -- `/var/run/notifyd_<pid of notifyd>.status` -- if the provided one is invalid).

1. IPC via Mach + MIG
 - This path is reachable by sending some crafted messages to the notifyd service. It required the sender of the messages to have some private entitlements, be running as root, and pass a sandbox profile check.
2. Signal handler
 - This path is reachable by sending a `SIGUSR1` signal to the `notifyd` process. This does not require any special entitlements or privileges. However, it does require the sender to be running as the same user as `notifyd` (which is usually `root`). 

Through the dump you can actively find a lot of otherwise elusive things (names of notifications, registered clients, etc). This is especially useful for reverse engineering purposes.

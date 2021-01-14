---
layout: post
title:  "Test podman with inspec"
date:   2021-01-13 08:00:00 +0100
categories: podman inspec make
tag: podman
---
# The journey
I've been running windows 10 and WSL2 on my work laptop since WSL2 was available and I was allowed to install it by company policy. Previously I had a setup with WSL1, virtualbox and boot2docker for running and building docker images on my local machine. This served me well for quite some time, but I wasn't to pleased with the WSL1 integration and performance, so I decided to go for WSL2. 

One of my main goals with this setup is that I shouldn't loose any benefits, I already managed to achive with my WSL1. So I needed to find a way to run containers on this setup. After some thinkering, I managed to get podman and buildah running inside WSL2 (This is enough for another blogpost, perhaps later). With this setup I am capable of building and running containers on my local machine. This all works well, so it's time for the next step. Testing these images and make sure they fullfill all the requirements that were set out.

We already setup some preliminary testing with inspec, but this setup was designed while we were running docker and not podman. The first test, just giving it a go, resulted in catastrophic failure, as stated below.

```bash
export CT=example:240cf8b032
CID=$(podman run --rm -d -i --init --entrypoint=/bin/cat ${CT})
inspec exec https://github.com/dev-sec/linux-baseline -t docker://${CID}
Traceback (most recent call last):
        34: from /usr/bin/inspec:206:in `<main>'
        33: from /usr/bin/inspec:206:in `load'
        32: from /opt/inspec/embedded/lib/ruby/gems/2.6.0/gems/inspec-bin-4.23.15/bin/inspec:11:in `<top (required)>'
...
1: from /opt/inspec/embedded/lib/ruby/2.6.0/socket.rb:1213:in `connect_nonblock'
/opt/inspec/embedded/lib/ruby/2.6.0/socket.rb:1213:in `__connect_nonblock': No such file or directory - connect(2) for /var/run/docker.sock (Errno::ENOENT)
        34: from /usr/bin/inspec:206:in `<main>'
        33: from /usr/bin/inspec:206:in `load'
...
```

Most likely you'll end up google'ing stuff like '''podman inspec''' or '''podman rspec''' and finally at some point, you'll probably find [this github issue](https://github.com/inspec/train/issues/525) and continue your search in the train repository, where you'll search will soon become a dead end. 

Lucky for you there is a solution, but you just have to figure out how to use it.

# Podman vs Inspec

If you manage to figure this all out and didn't give up, at some point you might discover there is a podman system service [ref-link](https://github.com/containers/podman/blob/master/docs/source/markdown/podman-system-service.1.md) that can be run, to create a system daemon that can be used in the same way as you used to have when you were running docker on your machine/server/...

It might be as easy as running
```bash
podman system service -t 0 unix://var/run/docker.sock
```
but you'll find out, you can't write in the run folder as your own user.

You might be tempted to run
```bash
sudo podman system service -t 0 unix://var/run/docker.sock
```
But if you've been trying so hard to not run images as root and not installing docker, why would you run the daemon as root? This command should do the trick:

```bash
podman system service -t 0 unix://$XDG_RUNTIME_DIR/podman.sock & DOCKER_SOCK_PID="$!"
```

Now you can use this socket as a command interface to podman and try to run your test again.

```bash
export CT=alpine:latest
CID=$(podman run --rm -d -i --init --entrypoint=/bin/cat ${CT})
DOCKER_HOST=unix://$XDG_RUNTIME_DIR/podman.sock inspec exec https://github.com/dev-sec/linux-baseline -t docker://${CID}
[2021-01-13T08:19:48+01:00] WARN: URL target https://github.com/dev-sec/linux-baseline transformed to https://github.com/dev-sec/linux-baseline/archive/master.tar.gz. Consider using the git fetcher

Profile: DevSec Linux Security Baseline (linux-baseline)
Version: 2.6.1
Target:  docker://92193531e79df9e6f9a8a29a694718a71b4525a7c329de5615c7185b4b7a95ca

  ✔  os-01: Trusted hosts login
     ✔  File /etc/hosts.equiv is expected not to exist
  ×  os-02: Check owner and permissions for /etc/shadow (1 failed)
     ✔  File /etc/shadow is expected to exist
     ✔  File /etc/shadow is expected to be file
     ✔  File /etc/shadow is expected to be owned by "root"
     ✔  File /etc/shadow is expected not to be executable
     ✔  File /etc/shadow is expected not to be readable by other
     ✔  File /etc/shadow group is expected to eq "shadow"
     ✔  File /etc/shadow is expected to be writable by owner
     ✔  File /etc/shadow is expected to be readable by owner
     ×  File /etc/shadow is expected not to be readable by group
     expected File /etc/shadow not to be readable by group
...
  ✔  sysctl-33: CPU No execution Flag or Kernel ExecShield
     ✔  /proc/cpuinfo Flags should include NX


Profile Summary: 26 successful controls, 29 control failures, 1 control skipped
Test Summary: 69 successful, 51 failures, 3 skipped
```

Don't forget to cleanup after you run the test. Stop the listening socket and remove the file as podman doesn't seem to cleanup after itself.
```bash
kill -TERM DOCKER_SOCK_PID="$!"
rm $XDG_RUNTIME_DIR/podman.sock
```

Good luck experimenting and running tests on your containers with podman and inspec!

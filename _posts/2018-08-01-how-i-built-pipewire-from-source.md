---
layout: post
section-type: post
title: How I built pipewire from source code in Linux with systemd
category: tech
tags: [ 'multimedia', 'gnome', 'pipewire', 'linux' ]
---

Lately, I have been interested in contributing to
[Pipewire](https://pipewire.org/). One interesting thing about it is that it
allows you to use the same video device (for example your webcam) in
different applications at the same time.

If you have a webcam, try for example in your terminal:
```
gst-launch-1.0 v4l2src ! xvimagesink
```
It just will open a window with the capture of your webcam... don't close it yet!
If in other terminal, you run `gst-launch-1.0 v4l2src ! xvimagesink` again,
you will get the following error:

![Device busy](/img/posts/2018-08-01-how-i-built-pipewire-from-source/v4l2src-device-busy.png)

and you realize that your webcam can only be used by one application at the same
time, at least until you keep reading the rest of this post.

As I said, I was interested in contributing to pipewire, so you will see how to
build it form source code. But if you just want to use the one that is already
provided by your distro repositories, just go to the final part.

```
cd ~
git clone https://github.com/PipeWire/pipewire
cd pipewire
```

Then you can build pipewire, but before that, for the time I am trying it
(2018-08-01), I had the problem that pipewire wanted to override my local
pipewire installation (the one that I have already installed from the official
packages of Fedora) even when I set a prefix.

![systemd unit directory error because of missing permissions](/img/posts/2018-08-01-how-i-built-pipewire-from-source/systemduserunitdir-error.png)

This error, as you may say, is because my user does not have permissions to
write into `/usr/lib/systemd/user`. That's fine. Actually, I am not sure how bad
or good would be to override my system wide pipewire installation. Luckyly, a
guy from the #pipewire channel with the nickname _jadahl_ had a
[patch](/assets/posts/2018-08-01-how-i-built-pipewire-from-source/patch/systemduserunitdir.diff).

So if you have the same problem, use that patch. To use it just do,
```
git checkout -b systemduserunitdir
curl -O http://cfoch.github.io/assets/posts/2018-08-01-how-i-built-pipewire-from-source/patch/systemduserunitdir.diff
git apply systemduserunitdir.diff
```

Now we will build pipewire from source...
but before, I some folders a directory where pipewire files will be installed to:

```
mkdir -p ~/env/pipewire
```

and also I create the following directory because the `.socket` and `.service`
unit files should be installed there instead of `/usr/lib/systemd/user`:

```
mkdir ~/.local/share/systemd/user
```

Now, to build pipewire:

```
mkdir builddir
cd builddir
meson --prefix=$HOME/env/pipewire ..
meson configure -Dsystemd_user_unit_dir=$HOME/.local/share/systemd/user
ninja
ninja install
```

If you had all the required dependencies, you will have pipewire installed now.
In your `~/.bashrc` file, you may want to add the following lines:

```
export PIPEWIRE_PREFIX=$HOME/env/pipewire
export PREFIX_PATH=$PIPEWIRE_PREFIX/bin:$PATH
export LD_LIBRARY_PATH=$PIPEWIRE_PREFIX/lib64:$LD_LIBRARY_PATH
export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:$PIPEWIRE_PREFIX/lib64/pkgconfig
export GST_PLUGIN_PATH_1_0=$PIPEWIRE_PREFIX/lib64/gstreamer-1.0/:$GST_PLUGIN_PATH_1_0
```

In my case, I didn't add these lines to my `.bashrc`, because I have a script
called `env.sh` in `~/env/pipewire/env.sh`, so whenever I want to build an
application that uses pipewire from master I just run `source ~/env/pipewire/env.sh`
to enter in my environment. You can download my script from
[here](/assets/posts/2018-08-01-how-i-built-pipewire-from-source/script/env.sh)
but I warn that it may contain things that you don't need!

Now, you have pipewire installed from master. You have to start the daemon, but
you don't want to start the daemon from the pipewire that is installed from the
repositories of your Linux distribution.

The solution is to tell systemd to start the daemon for the current user session:
```
systemctl --user start pipewire
```

However, when I did that I had the following error:
![systemd failed](/img/posts/2018-08-01-how-i-built-pipewire-from-source/pipewire-systemd-failed.png)

I checked my log `journalctl --user -u pipewire` and the problem was because of an
"error while loading shared libraries: libpipewire-0.2.so":

![systemd failed log](/img/posts/2018-08-01-how-i-built-pipewire-from-source/pipewire-systemd-failed-log.png)

for some reason, _systemd_ wasn't recognizing my environment. Looking at the
manual page of it I was really happy when I found an option to set an
environment variable. Before executing the following command take care you
dont't have the LD_LIBRARY_PATH already set in _systemd_, you can check that with
`systemctl --user show-environment | grep LD_LIBRARY_PATH`. If the output is
empty, run this command:

```
systemctl --user set-environment LD_LIBRARY_PATH=$HOME/env/pipewire/lib64
```

Now you can start the daemon:

```
systemctl --user start pipewire
```

![systemd running](/img/posts/2018-08-01-how-i-built-pipewire-from-source/pipewire-systemd-running.png)

Also, you will note that the output says that pipewire is loaded from the
systemd unit user directory which in my case is
`/home/cfoch/.local/share/systemd/user/pipewire.service`. That is exactly what I
want.

To test your webcam being shared by multiple applications you can try the
following command in two or more terminals:

```
gst-launch-1.0 pipewiresrc ! videoconvert ! xvimagesink
```


![multiple pipewiresrc](/img/posts/2018-08-01-how-i-built-pipewire-from-source/screenshot-gstpipewiresrc.png)

You can also try one of the pipewire's examples:

```
cd ~/pipewire/builddir/src/examples/
./video-play
```

![video-play sample](/img/posts/2018-08-01-how-i-built-pipewire-from-source/screenshot-pipewire-video-play-sample.png)

Well... that's it!

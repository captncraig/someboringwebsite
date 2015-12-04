+++
date = "2015-12-03T10:11:21-07:00"
description = "Why and how I dropped nginx for caddy."
title = "Deploying Caddy"
draft = true
tags = [
    "caddy",
    "deployment",
]
+++

I am pretty infatuated with [mholt](https://github.com/mholt)'s new webserver, [caddy](http://caddyserver.com). I've been using it to host a variety of thing on my own server (including this new fancy blog), and found it all around easier to configure and use. It also has a ton of cool features built in:

- [Magical](https://www.youtube.com/watch?v=nk4EWHvvZtI), Automatic TLS certificates with Let's Encrypt.
- Simple config file format. No magic.
- [git plugin](http://caddyserver.com/docs/git) Allows for automatic deployment on git hooks (more about this in another post maybe).
- Good enough performance. Not battle hardened, but benchmarks are comperable to nginx for basic things.
- Static content serving or dynamic proxying.
- Tons of cool [plugins](https://caddyserver.com/docs).
- Super easy extensibility (in Go!).

In the slack room, one question that comes up pretty much every day is "how do I daemonize and run this thing?" To me this means two things:

1. It must start when the system starts.
2. It must restart if it crashes.

My go to solution for these goals has been supervisord because it is pretty easy to configure. I had caddy running under supervisor for some time, but ended up moving off because of the way caddy can restart itself gracefully. I couldn't get supervisor to recognize the new forked process. If you are running caddy with Let's Encrypt support, caddy will automatically restart itself when it renews certs, so this causes problems. This led me to systemd, which has always scared me, but it turns out it is not so bad.

I will be showing how I did this on my Digital Ocean droplet running Ubuntu 15.10. Your mileage may vary on other systems, but it should work for any linux system with systemd as the default init system.

# 1. Unit file

Every systemd unit needs a unit file. Here's the base of mine:

```
[Unit]
Description=Caddy

[Install]
WantedBy=multi-user.target

[Service]
Type=simple
ExecStart=/usr/local/bin/caddy -pidfile /tmp/caddy.pid -agree -email me@myemail.com -conf /home/caddy/Caddyfile -log stderr
PIDFile=/tmp/caddy.pid
ExecReload=/bin/kill -USR1 $MAINPID

LimitNOFILE=32768
Restart=always
```

This is pretty simple stuff. The biggest thing you need to worry about is the `ExectStart` line. This is the caddy command and the arguments to it. You must supply the full path to your caddy executable (`/usr/local/bin/caddy` for me), the full path to your caddy file (`/home/caddy/Caddyfile` for me), and your email. You shouldn't really need to edit anything else.

I drop this file as `/etc/systemd/system/caddy.service`

Tell the system to enable it by running `sudo systemctl enable caddy`, and start it with `sudo service caddy start`.

Caddy should now be running and you should be able to verify by visiting a site from your Caddyfile.

# 2. Security and Users

The above unit file will run caddy as root, which is certainly easier, but is not really a good security practice. Especially since my instance is pulling and executing code from the internet.

I run caddy as a special `caddy` user, who has almost no priveleges.

To do so, add the following to your unit file:

```
User=caddy
PermissionsStartOnly=true
ExecStartPre=/sbin/setcap cap_net_bind_service=+ep /usr/local/bin/caddy
Environment=HOME=/home/caddy
```

The pre start command there uses setcap so that the caddy executable can bind to 80 and 443 as a non-root user. Its a pretty cool trick if you ask me. I also make sure to set the `HOME` environment variable so caddy knows where to store it's certificates and other data.

# 3. Logging 

By default, all logs will go into `/var/log/syslog`. This is ok, but almost everything else on your system also goes there. I like to split caddy logs into their own file for easy access. I can so this by adding a simple rule to `rsyslog`. I put the following in `/etc/rsyslog.d/50-caddy.conf`:

```
:syslogtag, startswith, "caddy" /var/log/caddy.log

```

After rebooting all logs also go to `/var/log/caddy.log` so I can view them easier. 

To make log viewing super easy I also add the following to my Caddyfile:

```
logs.captncraig.io {
	basicauth / myuser mypass
	root /var/log
}
```

With that I have a nice graphical way to browse logs remotely in my browser. I wouldn't do that on a system where I care about security, but it is pretty nice for my toy stuff.

# 4. Custom build / plugins

One thing that kinda sucks about go programs is the lack of any ability to load code at runtime.

Caddy has a rather novel way of loading plugins, but it requires actually changing the caddy source and recompiling. There is a pretty good tool to automate this: [caddyext](https://github.com/caddyserver/caddyext). I run this little script to build caddy from tip with the git extension as well.

```
export GOPATH=/opt/gopath
export PATH=/opt/gopath/bin:$PATH
go get -v -u github.com/mholt/caddy
go get -v -u github.com/abiosoft/caddy-git
go get -v -u github.com/caddyserver/caddyext
caddyext install git github.com/abiosoft/caddy-git
go install github.com/mholt/caddy
```

You can of course add as many extensions as you like this way.

# that's it!

It seems like a lot, but it is really not so much. If there was a simple debian package it might be easier, but that doesn't exist (yet). This gives me a ton of power and flexibility, and is also a pretty good framework for running any app under systemd. 

Special thanks to [@luitdv](https://twitter.com/luitvd) for help with some of the trickier bits of systemd and caddy. Hope it helps!
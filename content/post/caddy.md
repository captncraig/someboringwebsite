+++
date = "2015-12-01T10:11:21-07:00"
description = "Why and how I dropped nginx for caddy."
title = "Deploying Caddy"
tags = [
    "caddy",
    "deployment",
]
+++

I am pretty infatuated with [mholt](https://github.com/mholt)'s new webserver, [caddy](http://caddyserver.com). I've been using it to host a variety of thing on my own server (including this new fancy blog), and found it all around easier to configure and use. It also has a ton of cool features built in:

- [Magical](https://www.youtube.com/watch?v=nk4EWHvvZtI), Automatic TLS certificates with Let's Encrypt.
- Simple config file format.
- [git plugin](http://caddyserver.com/docs/git) Allows for automatic deployment on git hooks (more about this in another post maybe).
- Good enough performance. Not battle hardened, but benchmarks are comperable to nginx for basic things.
- Static content serving or dynamic proxying.
- Tons of cool [plugins](https://caddyserver.com/docs).
- Super easy extensibility (in Go!).

In the slack room, one question that comes up pretty much every day is "how do I daemonize and run this thing?"

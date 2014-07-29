---
layout: post
title: "Solution for Google banning Linodes"
date: 2014-07-28 18:00:00
categories: tricks
comments: true
---

### Google banned Linodes

I usually use linode to visit Google services via ssh tunnel. but recently I always got captchas even `Sorry...` page.

Finally I knows that google banned ipv6 traffics from linode which they treated as robots.

### Solution

disable ipv6 of linode

#### for Ubuntu

append lines below to `/etc/sysctl.conf`

* net.ipv6.conf.all.disable_ipv6=1

* net.ipv6.conf.default.disable_ipv6=1

* net.ipv6.conf.lo.disable_ipv6=1

then restart network `/etc/init.d/networking restart` or `reboot`

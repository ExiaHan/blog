---
title: "Start To Use Nginx Hexo and Let's Encrypt"
date: 2017-01-30T03:51:33+08:00
categories: ['Blog']
tags: ['Nginx', 'Https', 'Hexo']
---

## TL;DR

It's just a record for myself. It may contain errors. But it still can help you if you don't know how to start the config.
For a Offical instruction, you may try this in [Digital Ocean](https://www.digitalocean.com/community/tags/let-s-encrypt?type=tutorials).
**And what I think is that: The tutorials in DigitalOcean is much better that what is written in some linux distribution's wikipage like ubuntu....**

## Abstract

It is time to embrace https since chrome begin to mark website which use http as unsafe. As we know hexo are just a static html server, so if we want to use https, we need a **real** web server, which should supports https. What I choose is Nginx, it small, simple, elegent and stable. Below will show you how to set up nginx to support https and how to let nginx work with hexo and git.

##
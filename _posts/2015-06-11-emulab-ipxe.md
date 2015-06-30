---
layout: post
title: Emulab iPXE Fork
---

Quick start guide:

```
cd src
make
```

For any more detailed instructions, see [http://ipxe.org](http://ipxe.org)

This fork is funded by [The New Mexico Consortium](http://newmexicoconsortium.org) as part of the NSF funded [PRObE Project](http://nmc-probe.org)

# New commands

These new commands are available at the iPXE prompt or can be part
of an iPXE script

## bootinfo

Usage:

```
bootinfo <server name>
```

Gets the Emulab bootinfo for this computer.


## event

Sends an event from this node to the Emulab event system

```
event <dest server> <event name>
```

---
layout: post
title: "Virtualisierung als Sicherheitsfeature?"
date: 2007-10-27 19:19
comments: true
categories: ramblings
---

Eine Diskussion zu [Xen](http://de.wikipedia.org/wiki/Xen "Wikipedia: Xen") und [OpenBSD](http://www.openbsd.org/ "OpenBSD: Free, functional and secure.") samt der scheu hervorgebrachten Meinung »Virtualization seems to have a lot of security benefits« provozierte einmal mehr einen weiteren Grundsatzerguss [Theo de Raadts](http://en.wikipedia.org/wiki/Theo_de_Raadt "Wikipedia: Theo de Raadt") auf der öffentlichen Mailingliste des selbsternannt proaktiven Betriebssystems - in bekanntem Tonfall:

>„You’ve been smoking something really mind altering, and I think you should share it. x86 virtualization is about basically placing another nearly full kernel, full of new bugs, on top of a nasty x86 architecture which barely has correct page protection. Then running your operating system on the other side of this brand new pile of shit. You are absolutely deluded, if not stupid, if you think that a worldwide collection of software engineers who can’t write operating systems or applications without security holes, can then turn around and suddenly write virtualization layers without security holes. You’ve seen something on the shelf, and it has all sorts of pretty colours, and you’ve bought it. That’s all x86 virtualization is.“

Diesem Kontext sei [“An Empirical Study into the Security Exposure to Hosts on Hostile Virtualized Environments”](http://taviso.decsystem.org/virtsec.pdf) zum Geleit gegeben. Das wenig überraschende Fazit: Auch virtuelle Maschinen möchten abgesichert und regelmäßig gepflegt sein. Unfehlbar sind sie nicht.

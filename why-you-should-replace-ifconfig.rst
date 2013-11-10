Why you should replace ifconfig (net-tools) with iproute2
#########################################################
:date: 2011-12-22 03:15
:author: Tim Heckman
:copyright: 2011
:license: CC BY-SA
:category: linux
:tags: linux, networking
:slug: why-you-should-replace-ifconfig

|Sample output of iproute2|

A Brief History of ifconfig (net-tools)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Most system administrators who are tasked with maintaining UNIX and
Linux systems have the syntax for the ifconfig burned in to their
skulls. The combination of ``ifconfig``, ``route``, ``arp``, and
``netstat`` have been the pillar used to configure networks and
troubleshoot issues on UNIX-like operating systems for over 25 years.

The ifconfig command was originally released as part of the BSD TCP/IP
toolkit in 4.2 BSD back in August of 1983. The use of ifconfig in BSD
paved the way for it to be considered one of the utilities in the
original toolkit of the internet.

The freely available BSD UNIX operating systems (e.g., FreeBSD, NetBSD,
etc.) still continue active development of the ifconfig utility. The
core features of the utility of been extended to configure WLAN
interfaces and controlling advanced features such as bridging and
tunneling. However, ifconfig on the Linux side has seen no major
updates in over 10 years. The latest release is 1.60 from April 15,
2001.

Enter iproute2!
^^^^^^^^^^^^^^^

This collection of utilities was first released in April of 1999 and
replaced the functions of net-tools. Iproute2 brings additional
features, some of which have been implemented in the BSD ifconfig, such
as multicast support, traffic control (shaping), tunnel management, as
well as others. Most modern distributions have this installed by
default and are working to fully replace ifconfig (and the net-tools
package) with iproute2. Arch Linux having done so back in `June of
2011`_.

While the toolkit does bring new features, it also requires some muscle
memory be broken when it comes to networking. The commands do provide
the same features, however the syntax is a bit different.

Why iproute2?
^^^^^^^^^^^^^

If you are reading an article online, an excerpt from a book, or some
tutorial you will see the net-tools package used in most cases. But why
is a package considered deprecated still being used? This is most
likely due to the majority of users being familiar, or at least having
been exposed to, the net-tools suite. Additionally, for some system
administrators the familiarity of the command if comforting.

The comfort, however, is no replacement for the power of the iproute2
toolset. The iproute2 package usages the ``ip`` command. Under this
command the functionality of ``route``, ``ifconfig``, ``ipmaddr``, and
``iptunnel`` are implemented. This provides you with one command to
handle your networking tasks. While it will take a bit of habit
breaking, learning the syntax will make network management an
easier endeavor.

Part of the iproute2 toolkit the ability for network traffic control.
 This is something that cannot be done with the net-tools suite. With
the ``tc`` command a knowledgable system administrator can implement
traffic shaping as well as a policy for queueing packets.

The next feature that is not implemented in net-tools is the ability to
have multiple routing tables. There are quite a few uses for having
multiple routing tables such as implementing a routing policy based on
the source IP address. You may be asking yourself where this is useful,
cases such as having different networks on a single system can benefit
from such a policy.

Still not sold?
^^^^^^^^^^^^^^^^

Iproute2 provides you with a modern suite of utilities that makes the
experience of maintaining the network on your Linux system easier and
more intuitive. The toolset is actively developed and maintained and is
the future of how you will be handling your network configurations on
Linux operating systems. The toolset also gives you the ability to
perform more advanced networking administration duties.

By making the permanent change to iproute2 in all of your scripts and
daily management tasks you will help move away from the net-tools
package.

The Syntax
^^^^^^^^^^^

In the coming days I will be writing a quick blog post "iproute2 by
example". I'll be going over some of the key pieces of the utility to
provide a quick-start guide.

More Information
^^^^^^^^^^^^^^^^

You can read more about the iproute2 suite on these following sites:

-  http://en.wikipedia.org/wiki/Iproute2
-  http://www.linuxfoundation.org/collaborate/workgroups/networking/

.. _June of 2011: http://www.archlinux.org/news/deprecation-of-net-tools/

.. |Sample output of iproute2| image:: /images/iproute2.png
   :target: /images/iproute2.png

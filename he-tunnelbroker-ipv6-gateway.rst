Hurricane Electric Tunnel Broker IPv6 Gateway
#############################################
:date: 2011-05-24 11:22
:author: Tim Heckman
:copyright: 2011
:license: CC BY-SA
:category: IPv6
:tags: 6in4, debian, he, henet, hurricane electric, ipv4, ipv6, linux, networking, radvd, ubuntu
:slug: he-tunnelbroker-ipv6-gateway

|Hurricane Electric IPv6 Tunnel Broker|

With the global regional internet registries' IPv4 pools starting to run
dry the interest in adopting IPv6 has come to the forefront. While it
may be some time until you are able to run native IPv6 at home, there's
an easy way to have all of your devices on the home network, including
your Android smartphone and Apple iPod, use your Hurricane Electric IPv6
tunnel. With the configuration explained in this article any device that
is ready for IPv6, and accepts stateless configuration, will be able to
communicate via IPv6 shortly after attaching to your network. The
networking configuration will be handled by the Debian "interfaces" file
as well as iproute. The stateless autoconfiguration will be handled by
radvd.

I recently purchased a newer netbook from my coworker, and was curious
about how I could use the old one. This idea to configure an IPv6
gateway, which had already been implemented in a similar fashion by
someone else I work with, instantly came to mind. If you want to use
your current PC (running Debian for example) to handle this, there
should be no issue with that. Just remember if that system goes down
for some reason all IPv6 traffic on the network goes with it until it's
back up. I'm often booting to different operating systems, so a
dedicated system was needed in my case.

Additionally, this article was written from the experience on freshly
installed Debian Squeeze (SSH Server + Core install) system. The
instructions should hold true to for any Debian-derived distribution
(the Ubuntu family, Linux Mint). If you are not using Debian this
article should at least point you in the right direction.

At this point you will want to have obtained an IPv6 tunnelbroker
account from Hurricane Electric. If you've yet to obtain an account
there head on over to http://tunnelbroker.net/. The sign-up process is
pretty straight forward. At the end they'll e-mail you a randomly
generated password and you can then log in and make the password a bit
more personal. Once logged in you'll want to create a new regular
tunnel from the menu on the left side. When on the tunnel creation page
simply enter the IPv4 address you'll be connecting from and nearest
tunnel server. Once the tunnel is created you'll be taken to a page
that has the tunnel information. You'll want to keep this handy as
we'll need this information throughout the whole process.

Now let's get started. The first thing we're going to accomplish is
bringing the IPv6 tunnel up on machine that's going to act as your
gateway. This process is pretty straight forward. There's two ways to
accomplish this. The first method will show you how to bring the IPv6
tunnel up manually using only iproute. The method I recommend is editing
your /etc/network/interfaces file to bring the tunnel up at boot. This
latter method is the one we'll be using for the end product. This
method I do use relies on iproute for some of the configuration after
the tunnel comes up.

The first step is going to be to ensure that your kernel is ready for
IPv6 networking. This should be compiled in to the kernel, and if not
it's fairly easy to enable the support. This command will tell us if
the support is compiled in to your kernel:

::

    grep -i "config_ipv6" /boot/config-$(uname -r)

    CONFIG_IPV6=y
    CONFIG_IPV6_PRIVACY=y
    CONFIG_IPV6_ROUTER_PREF=y
    CONFIG_IPV6_ROUTE_INFO=y
    CONFIG_IPV6_OPTIMISTIC_DAD=y
    CONFIG_IPV6_MIP6=y
    CONFIG_IPV6_SIT=m
    CONFIG_IPV6_NDISC_NODETYPE=y
    CONFIG_IPV6_TUNNEL=m
    CONFIG_IPV6_MULTIPLE_TABLES=y
    CONFIG_IPV6_SUBTREES=y
    CONFIG_IPV6_MROUTE=y
    CONFIG_IPV6_PIMSM_V2=y

If your output does not look similar to that it means your kernel was
not compiled with IPv6 baked in. However, your distribution maintainers
may have been mindful of that and already configured it to load as a
module. You can check that with this command:

::

    lsmod | grep ipv6

If the module has not been loaded we will need to load the module, and
then configure your system to bring that module up at boot. The first
step would be to enable the module with modprobe:

::

    modprobe ipv6

You would also want to add the following line to /etc/modules to ensure
that module is loaded on boot as well:

::

    ipv6

Now that you have IPv6 enabled in your kernel we are ready to bring the
tunnel. The first method is to bring up the route manually using
iproute. We will not be using this method in the final product, but
it's nice to know the commands so that you are able to play with
bringing up additional IP addresses in your block later on. So now
let's get the tunnel up and running to ensure your IPv6 connectivity is
working:

The first step would be to make sure iproute (sometimes called iproute2)
is installed on your system:

::

    apt-get update && apt-get upgrade
    apt-get install iproute

You would then want to proceed to bring the tunnel up manually:

::

    ip tunnel add he-ipv6 mode sit remote 209.51.161.14 local 10.0.0.10 ttl 255
    ip link set he-ipv6 up
    ip addr add 2001:470:1f06:1251::2/64 dev he-ipv6
    ip route add ::/0 dev he-ipv6
    ip -f inet6 addr

**Note**: I noticed the linewrapping above makes it look like the "ttl"
section is its own command. That actually is part of the command above
it and should be on the same line

Before running these commands let me explain what each one does, as well
as note any changes that will be made so it works on your system.

-  ip tunnel add he-ipv6 mode sit remote 209.51.161.14 local 10.0.0.10
   ttl 255

   -  The first command adds a tunnel named "he-ipv6" device to your
      system.
   -  The mode is sit because it's IPv6 over IPv4.
   -  The remote address is the IPv4 address of your Tunnel Broker
      server. You'll need to change this so it matches your Server IPv4
      address from the Tunnel Broker information page.
   -  The local IP address is going to be the eth0 IPv4 address of the
      system you are running the command on. My netbook was given
      10.0.0.10 by my router, you'll want to change this so it matches
      the IPv4 address of eth0 on your system.
   -  Sets the time to live (TTL) of the packets to 255, as recommended
      by HE.

-  ip link set he-ipv6 up

   -  This brings the he-ipv6 tunnel up.

-  ip addr add 2001:470:1f06:1251::2/64 dev he-ipv6

   -  This adds the client IPv6 address to your he-ipv6 tunnel. You'll
      want to obtain this IP address from the Tunnel Broker information
      page.

-  ip route add ::/0 dev he-ipv6

   -  Adds a route for ::/0 to use he-ipv6.

-  ip -f inet6 addr

   -  Prints the IPv6 addresses of all interfaces on the system

At this point you should be able to consider yourself a member of the
exclusive IPv6 club. You can verify that by running the following
command:

::

    ping6 -c4 ipv6.google.com

If your tunnel is not working at this point, or you made a mistake while
inputting your commands, you can take down the castle you just build
with these commands:

::

    ip route del ::/0 dev he-ipv6
    ip addr del 2001:470:1f06:1251::2/64 dev he-ipv6
    ip link set he-ipv6 down
    ip tunnel del he-ipv6

Now that you've gotten IPv6 working, I'm going to make you destroy the
castle anyhow. These commands show you how to bring up the IPv6 tunnel
manually. As mentioned previously, this will not be the method we'll
use to bring the tunnel up at boot. Once you've finished running those
commands we can move on to implementing the tunnel in a way that
provides a bit more sanity. We're going to create an interface in the
"/etc/network/interfaces" file to handle the creation and destruction of
the IPv6 tunnel, as well as the address needed to properly route other
devices on your network through that tunnel. If you are not using a
Debian-derived distribution you'll need to research how to accomplish
this on your distro. You may find that you actually need to create a
script to run the iproute commands, from above, at boot.

In my configuration I converted my gateway system to use static IPv4
addresses. There are two ways to do this. In most modern routers you
can establish static IP based on the device's MAC address. I, of
course, did this to ensure no other devices would be given the IP
address. However, to speed up the boot process (and for my own sanity)
I've modified the network configuration so IPv4 is completely static.
 If you do decide to edit your "/etc/network/interfaces" file to use a
static IP address you'll need to change these values to fit your
network:

::

    auto eth0
    iface eth0 inet static
        address 10.0.0.10
        netmask 255.255.255.0
        gateway 10.0.0.1

A few lines down after that I began to build my IPv6 tunnel interface.
 I'll just provide you with the entire block here, and then break down
each section and let you know what you'll need to change:

::

    auto he-ipv6
    iface he-ipv6 inet6 v4tunnel
        endpoint 209.51.161.14
        address 2001:470:1f06:1251::2
        netmask 64

        #bring up the networking needed for LAN dual-stack
        up /sbin/ip -6 route add default dev he-ipv6
        up /sbin/ip -6 addr add 2001:470:1f07:1251::1/64 dev eth0

        #take it down!
        pre-down /sbin/ip -6 addr del 2001:470:1f07:1251::1/64 dev eth0
        pre-down /sbin/ip -6 route del default dev he-ipv6

**Note**: Make sure that your "addr add" and "addr del" lines use your
*routed* /64 prefix, not the prefix for your IPv6 client IP

Now to break down this section, of course the lines beginning with '#'
is simply a commented that I added to keep track of what the lines do.

-  auto he-ipv6
-  iface he-ipv6 inet6 v4tunnel

   -  The "auto" line provides the instruction to bring this interface
      up when networking is started
   -  The second line creates the IPv6 v4tunnel interface called
      "he-ipv6"

-  endpoint 209.51.161.14

   -  This is the server IPv4 address for your tunnel. As when this was
      encountered way back during the iproute commands you would want to
      modify this to match your information.

-  address 2001:470:1f06:1251::2

   -  This is your client IPv6 address, make sure you replace this value
      with the one on your information page.

-  netmask 64

   -  The IPv6 address is part of a "/64". If you were using a "/48"
      allocation from HE you'd want to change this value to 48.

-  up /sbin/ip -6 route add default dev he-ipv6

   -  This has the default IPv6 route go through the he-ipv6 tunnel.

-  up /sbin/ip -6 addr add 2001:470:1f07:1251::1/64 dev eth0

   -  This adds an IPv6 address from your block of IPs to eth0. I
      simply added one to the last digit of the client IPv6 address.
      You could make this anything within your "/64" This will need to
      be replaced from something within your block.

-  pre-down /sbin/ip -6 addr del 2001:470:1f07:1251::1/64 dev eth0
-  pre-down /sbin/ip -6 route del default dev he-ipv6

   -  These two commands are ran before the he-ipv6 interface is brought
      down. Although I believe it would not cause any issues I'm taking
      the extra steps to bring this tunnel down cleanly. As with the
      previous "addr add" line, you'll need to make sure the two
      addresses match.

There may be a different way to accomplish this, as the interfaces file
allows you to do things a few ways depending on what you're working on.
 However, this allows me to easily control when the IPv6 address gets
brought up.

If you are not using a Debian-derived distribution here is a note for
you: Adding that IPv6 address to eth0 is important and you need to make
sure this step is done. I spent quite a bit of time trying to figure
out why my routes looked good, yet only my gateway could ping6 outside
of my LAN. It was because I neglected to bring an IPv6 address up on
eth0.

At this point your networking configuration should be complete. To keep
the Debian ifupdown system happy I'd say reboot your system at this
time. After it comes back up make sure you can run these two commands
without fail:

::

    ping -c4 www.google.com
    ping6 -c4 ipv6.google.com

Once you are certain your networking came up we're going to configure
radvd. Radvd (Router Advertisement Daemon) is an easy to set up
application that advertises your IPv6 address space to the local
network. This will allow for any device, that connects to your network
and allows stateless autoconfiguration (SLAAC or also known as
autoconf), to be given an IPv6 address based on its MAC address. In
essence you'll be running a dual-stack configuration on all devices,
that support it, on your network. Yes, even your friend's laptop will
hop on the IPv6 partyvan. Let's get started by installing radvd.

::

    apt-get update && apt-get upgrade
    apt-get install radvd

At this point you will need to make some configuration changes to your
operating system to ensure IPv6 forwarding is enabled. This allows your
system to route the IPv6 traffic to and from your network and the
subsequent devices. To enable IPv6 forwarding for your running system
you'll need to issue this command:

::

    echo 1 > /proc/sys/net/ipv6/conf/all/forwarding

You will also need to uncomment this line from "/etc/sysctl.conf" (or
add it if it doesn't exist). It will enable IPv6 forwarding at boot:

::

    net.ipv6.conf.all.forwarding=1

Once that's in place you are ready to begin configuring radvd. There
was not a configuration installed by default on my system. You'll want
to create the following file with your preferred text editor:
/etc/radvd.conf

Here is the configuration I am using for radvd. You'd want to replace
the prefix with yours from the tunnel information page:

::

    interface eth0 {
        IgnoreIfMissing on;
        AdvSendAdvert on;
        MinRtrAdvInterval 30;
        MaxRtrAdvInterval 60;
        prefix 2001:470:1f07:1251::/64 {
            AdvOnLink on;
            AdvAutonomous on;
            AdvRouterAddr on;
        };
    };

**Note**: Be sure to use your routed /64 prefix from the tunnel broker
information page. Also, someone mentioned they had issues and had to
drop the Min/MaxRtrAdvInterval to 3 and 10, respectively.

This configuration will announce router advertisements so that even an
Android smartphone will be able to run a dual-stack IPv4/v6 network
configuration. Now to break down the configuration above:

-  interface eth0 { };

   -  This is the parent configuration block for the interface. In this
      situation the local network (where the IPv4 machines will be
      connecting from) will be on eth0.

-  IgnoreIfMissing on;

   -  After writing this article I found that eth0 wasn't always ready
      when radvd was started. As such, it would simply exit at boot
      leaving my IPv6 gateway not doing what it's supposed to. This
      option tells radvd that if eth0 is not ready yet, it should be in
      the near future and to keep an eye out for it. Once radvd sees
      that eth0 is ready to rock, it then begins normal operation.

-  AdvSendAdvert on;

   -  Enables the periodic sending of router advertisements.

-  MinRtrAdvInterval 30;

   -  The minimum amount of time allowed between sending unsolicited
      advertisements.

-  MaxRtrAdvInterval 60;

   -  The maximum amoutn of time allowed between sending unsolicited
      advertisements.

-  prefix 2001:470:1f07:1251::/64 { };

   -  Your IPv6 prefix that you'll be announcing RAdvs for.

-  AdvOnLink on;

   -  Announces that this prefix can be used for on-link determination.

-  AdvAutonomous on;

   -  This allows for autonomous address configuration.

-  AdvRouterAddr on;

   -  The address of the interface is announced rather than the network
      prefix.

Once this configuration is in place you should be all set to start
radvd, and IPv6 should be brought up on all of your devices shortly
thereafter:

::

    /etc/init.d/radvd start

In the event of power loss, your IPv6 connectivity should be restored
once the gateway machine is brought back up afterwards. If your ISP
provides dynamic IP addresses you may have to reset the IPv4 endpoint of
your tunnel. I'll be working on a BASH script in the coming days that
will automate the updating of the IPv4 address via the Tunnel Broker
API.

Here are some outputs from my gateway system and my desktop to help you
find any possible errors. Gateway:

::

    gateway ~# ifconfig
    eth0      Link encap:Ethernet  HWaddr 00:24:e8:ed:24:21
              inet addr:10.0.0.10  Bcast:10.0.0.255  Mask:255.255.255.0
              inet6 addr: 2001:470:1f07:1251::1/64 Scope:Global
              inet6 addr: fe80::224:e8ff:feed:2421/64 Scope:Link
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:9882 errors:0 dropped:0 overruns:0 frame:0
              TX packets:7893 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:1000
              RX bytes:3272489 (3.1 MiB)  TX bytes:1321119 (1.2 MiB)
              Interrupt:43 Base address:0x6000 

    he-ipv6   Link encap:IPv6-in-IPv4
              inet6 addr: fe80::a00:a/64 Scope:Link
              inet6 addr: 2001:470:1f06:1251::2/64 Scope:Global
              UP POINTOPOINT RUNNING NOARP  MTU:1480  Metric:1
              RX packets:1293 errors:0 dropped:0 overruns:0 frame:0
              TX packets:1870 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:0
              RX bytes:333050 (325.2 KiB)  TX bytes:179585 (175.3 KiB)

    lo        Link encap:Local Loopback
              inet addr:127.0.0.1  Mask:255.0.0.0
              inet6 addr: ::1/128 Scope:Host
              UP LOOPBACK RUNNING  MTU:16436  Metric:1
              RX packets:51 errors:0 dropped:0 overruns:0 frame:0
              TX packets:51 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:0
              RX bytes:4384 (4.2 KiB)  TX bytes:4384 (4.2 KiB)

    gateway ~# ip -6 route show
    2001:470:1f06:1251::/64 via :: dev he-ipv6  proto kernel  metric 256
    2001:470:1f07:1251::/64 dev eth0  proto kernel  metric 256
    fe80::/64 via :: dev he-ipv6  proto kernel  metric 256
    fe80::/64 dev eth0  proto kernel  metric 256
    default dev he-ipv6  metric 1024

Desktop:

::

    desktop ~# ifconfig
    eth0      Link encap:Ethernet  HWaddr 00:1f:bc:01:1c:34
              inet addr:10.0.0.2  Bcast:10.0.0.255  Mask:255.255.255.0
              inet6 addr: 2001:470:1f07:1251:21f:bcff:fe01:1c34/64 Scope:Global
              inet6 addr: fe80::21f:bcff:fe01:1c34/64 Scope:Link
              UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
              RX packets:968793 errors:0 dropped:0 overruns:0 frame:0
              TX packets:735364 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:1000
              RX bytes:473842221 (473.8 MB)  TX bytes:76510896 (76.5 MB)
              Interrupt:48 Base address:0xa000 

    eth1      Link encap:Ethernet  HWaddr 00:1f:bc:01:1c:35
              UP BROADCAST MULTICAST  MTU:1500  Metric:1
              RX packets:0 errors:0 dropped:0 overruns:0 frame:0
              TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:1000
              RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
              Interrupt:49 Base address:0xc000 

    lo        Link encap:Local Loopback
              inet addr:127.0.0.1  Mask:255.0.0.0
              inet6 addr: ::1/128 Scope:Host
              UP LOOPBACK RUNNING  MTU:16436  Metric:1
              RX packets:2274 errors:0 dropped:0 overruns:0 frame:0
              TX packets:2274 errors:0 dropped:0 overruns:0 carrier:0
              collisions:0 txqueuelen:0
              RX bytes:303532 (303.5 KB)  TX bytes:303532 (303.5 KB)

    desktop ~# ip -6 route show
    2001:470:1f07:1251::/64 dev eth0  proto kernel  metric 256  expires 86366sec mtu 1500 advmss 1440 hoplimit 4294967295
    fe80::/64 dev eth0  proto kernel  metric 256  mtu 1500 advmss 1440 hoplimit 4294967295
    default via fe80::224:e8ff:feed:2421 dev eth0  proto kernel  metric 1024  expires 146sec mtu 1500 advmss 1440 hoplimit 64

I welcome any feedback that you may have and if you have any questions
regarding the process don't hesitate to comment and I'll do my best to
reply back and offer any assistance.

UPDATE: Keeping The Tunnel Alive

I've found that with no activity my IPv6 tunnel would enter a bit of a
sleep state. As soon as my home activity picked up the tunnel would
wake back up. The only way I could thing to fix this would be
to periodically ping a known working IPv6 address. You can accomplish
this fairly easily. As root simply follow these steps:

::

    crontab -e

Add this line to the end of that file:

::

    */15 * * * * /bin/ping6 -c2 -i5 ipv6.google.com >/dev/null 2>&1

This will ping Google every 15 minutes. If you find this still causes
issues try changing the 15 to 10, or maybe even 5.

UPDATE[2]: Update IPv4 Address via API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

I've written a BASH script to be ran as a cronjob to keep tabs on your
local IPv4 address. HE needs to know your IPv4 address for the tunnel
to work. If your external IPv4 address changes it recognizes this and
updates Hurricane Electric's API accordingly. I've provided a full blog
post explaining the script `right here`_.

.. _right here: http://blog.timheckman.net/2011/05/31/ipv6-gateway-bash-script/

.. |Hurricane Electric IPv6 Tunnel Broker| image:: /images/he-tunnelbroker.jpg
   :target: /images/he-tunnelbroker.jpg

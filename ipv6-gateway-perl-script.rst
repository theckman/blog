IPv6 Gateway Maintenance Perl Script
####################################
:date: 2011-12-26 12:23
:author: Tim Heckman
:copyright: 2011
:license: CC BY-SA
:category: development, IPv6, linux, networking, perl
:tags: debian, development, hurricane electric, ipv6, networking, perl
:slug: ipv6-gateway-perl-script

|image0|

Earlier in 2011 I wrote an article providing instructions for `setting
up your own IPv6 dual-stack setup at home using a Hurricane Electric
Tunnel Broker account`_. The process works great if you are running the
tunnel on a network where your IPv4 address will not change. However,
for most people who would want this configuration (including myself),
your external IP address isn't always static. This left the door open
for your IPv6 networking to not work properly if your external address
changed. Shortly thereafter I wrote a shell script to help keep the
tunnel alive using the Tunnel Broker API (`link`_).

The shell script had a few limitations I had to work around so I
planned, awhile back, to rewrite the script. It took longer than I had
wanted but I finished it just in time for the New Year. On an off-topic
note I hope you all had a great Holiday and a prosperous New Year.

I'll be providing a quick history of the script as well as an overview
of the functionality after the break. If you just want to skip that and
get the script head on over to the \ `GitHub page`_. All the
information you need is provided in the README.

Hurricane Electric IPv4 BASH Script
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The first tool I used to keep my address up to date was a shell script
(`GitHub`_) that only updated Hurricane Electric if the external address
changed. I also implemented some URL rotation so I wasn't hitting
someone's IP script too often. There were some limitations I had to
overcome so the script is fairly hacked together to do what I want. You
can also see my explanation of the script in this `blog post`_.

The Reimplementation
~~~~~~~~~~~~~~~~~~~~

Even while writing the above script I knew that at some point I should
rewrite the script using a language that was more versatile. For awhile
I've been toiling with the decision of what language I wanted to work on
next. Perl or Python? I was a bit more familiar with Perl syntax and
would need to spend extra time learning the semantic of Python (which is
still on my todo list).

The Goal
~~~~~~~~

The goal of the script is to be ran on a regular schedule by cron with
root permissions. It was also to require no interaction from the
sysadmin as well as to provide plenty of information if the sysadmin
wishes to fiddle with it. The final goal was to only update the
external IPv4 address when needed and avoid any repetitive unnecessary
calls to the Hurricane Electric Tunnel Broker API as well as the
different sites that are used to pull the IP address.

How It Works
~~~~~~~~~~~~

The script uses a few Perl modules to log to Syslog, save the external
IP and last URL for external IP used to a YAML file to ensure smooth
operation, and to pull the IP and update the IP via HTTP/HTTPS. The
script is designed to be resilient if one, two, or even three of the
external IP address sites are down. I'll provide some detailed
information about each process.

Logging
^^^^^^^

I wanted to implement a logging system with varying verbosity to let you
know what's going on. In the configurations section there is a
commented out $debug variable that allows you to set the verbosity. The
script has a subroutine for logging called slog. The slog subroutine
then uses `Logger::Syslog`_ to spit the information to syslog. It may
also print depending on how verbose the debugging is.

YAML!
^^^^^

I needed a lightweight and easy solution for saving/retrieving the
previous external IP address as well as which URL was used last.
 Immediately Yet Another Markup Language (YAML) came to mind. By
default this information is saved in \`\`\`/var/cache/he-ipv4.yml\`\`\`
using `YAML::Tiny`_. This is configurable by changing the $configFile
variable to an absolute location. The data is also validated to make
sure it looks right. Is the IP and IP, and is the last used URL index
an integer?

Obtaining External IP
^^^^^^^^^^^^^^^^^^^^^

The process by which the external IP address is obtained leaves room for
some unexpected results/data. Garbage in, garbage out. Interactions
with websites for obtaining the IP address, as well as interacting with
Hurricane Electric Tunnel Broker API, use `LWP::Protocol::https`_
and \ `WWW::Mechanize`_. The subroutine that obtains the external IP
address is the most complex in the script as it has to make sure the
data it received is valid.

The code that pulls the external IP address is looped until either a
valid IP is obtained, or all URLs are unable to produce a valid IPv4
address. The data received from the server is mathed via Regex as well
as verifying that the HTTP status code is 200. If the script is unable
to obtain valid data it exits with an error.

Finalizing The Update
^^^^^^^^^^^^^^^^^^^^^

Once the script obtains the external IP address it does a quick check to
see if it has changed. If not the script writes the last external IP
address and URL used to the YAML file and exits.

If the IP address has changed it submits a call to the subroutine that
updates the external IP address. It uses the UserID, UserPass, and
TunnelID configuration options to set the new IPv4 address. Once the
API has been updated the tunnel interface is brought down and back up.
 Also, RAdvD is restarted to make sure the native IPv6 on the LAN is
fully woking.

Closing Notes
~~~~~~~~~~~~~

This gives you an overview of how the process actually works. The
project with full code, commit history, comments, and README are
available `on GitHub`_. I do encourage you to read, fork, and submit
patches for my code as I still have plenty to learn with Perl and do
welcome any enhancements to the script. I look forward to seeing what
you come up with.

As usual leave any comments or feedback you have about the article!

.. _setting up your own IPv6 dual-stack setup at home using a Hurricane Electric Tunnel Broker account: http://blog.timheckman.net/2011/05/24/he-tunnelbroker-ipv6-gateway/
.. _link: https://ipv4.tunnelbroker.net/ipv4_end.php
.. _GitHub page: https://github.com/theckman/he-ipv4-sh
.. _GitHub: https://github.com/theckman/he-ipv4-sh
.. _blog post: http://blog.timheckman.net/2011/05/31/ipv6-gateway-bash-script/
.. _`Logger::Syslog`: http://search.cpan.org/~sukria/Logger-Syslog-1.1/lib/Logger/Syslog.pm
.. _`YAML::Tiny`: http://search.cpan.org/~adamk/YAML-Tiny-1.50/lib/YAML/Tiny.pm
.. _`LWP::Protocol::https`: http://search.cpan.org/search?query=LWP%3A%3AProtocol%3A%3Ahttps&mode=all
.. _`WWW::Mechanize`: http://search.cpan.org/~jesse/WWW-Mechanize-1.71/lib/WWW/Mechanize.pm
.. _on GitHub: https://github.com/theckman/he-ipv4-perl

.. |image0| image:: /images/he-ipv4-perl.png
   :target: /images/he-ipv4-perl.png

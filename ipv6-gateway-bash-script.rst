IPv6 Gateway Maintenance BASH Script
####################################
:date: 2011-05-31 03:55
:author: Tim Heckman
:copyright: 2011
:license: CC BY-SA
:category: development
:tags: development, IPv6, linux, networking
:slug: ipv6-gateway-bash-script

|image0|

**Edit (12/17/2011)**: The script is now on `GitHub`_!

In my previous `blog post`_ I gave instructions as to how I configured
my own Hurricane Electric IPv6 gateway at home. I used my older netbook
and a fresh install of Debian Squeeze. The IPv6 tunnel relies on
knowing your current IPv4 address to validate your connection.
 Unfortunately, like many others, my IPv4 address is dynamic. I knew I
would have to either handle these situations manually, or write a script
to update the API with my new IPv4 address.

I found a few scripts online that did exactly that. However, most were
simple and just updated the API without checking to see if that was even
required. To minimize my API calls to the tunnel broker I wrote my own
moderately overkill BASH script to handle the updating of my external
IPv4 address. This script will only work on Debian-derived systems, and
is tailored for those who followed my previous blog post. Let me provide
you with a quick run-down of how the script works.

The big thing is the script needs to be ran as root. The reason for
this is that I restart the tunnel, as well radvd, are restarted if the
external IPv4 address changes. The script takes a few configuration
options: your hashed UserID (provided by HE on their Main Page), a
hashed version of your password (using md5sum), your tunnel interface
name, as well as options regarding the logging of the script and what
URLs to use to obtain the external IPv4 address. The script also
creates two cache files in /var/cache. These files store the most
recent IPv4 address, as well as the number used to alternate external
"what is my IP" URLs.

Here is how the script processes:

-  The script verifies this is being ran as root. If not it exits.
-  The script then checks to see if the two cache files exist.

   -  If the two files do not exist they are created. If the file
      creation fails the script exits with an error.

-  The script decides in which order the two URLs will be used by the
   information stored in the cache file.
-  The script then uses curl to pull your IPv4 address from an external
   source.

   -  It checks to see if the data obtained is an IPv4 address. If not,
      it tries the secondary external source. If that fails to obtain
      one it tries the primary source again. If that fails also, the
      script exits in error.

-  The script then checks to see if the cache file was created when this
   script ran.

   -  If the cache file was not created it checks to see if the cache
      file contains a valid IPv4 address.

      -  If it does, it then matches it against the IPv4 address
         obtained earlier. If the address is the same the script exits.
      -  If the IPv4 address has changed it's updated via HE's API, the
         tunnel interface is restarted as well as radvd. The new IPv4
         address is saved in the cache file for future use.

   -  If the cache file does not contain a valid IPv4 address the IP is
      updated via HE's API and an IPv4 address is saved in the cache
      file for future use.
   -  If the cache file is brand new the IPv4 address is updated and
      valid data is placed in the cache file.

To install the script simply download this file and rename it to
he-ipv4.sh. You'll then want to move the file to /usr/sbin/. Then
allow the file to be executable with chmod +x /usr/sbin/he-ipv4.sh.
 You'll also need to edit the file's configuration section to match your
Tunnel Broker and local network configuration. Then you want to edit
the crontab, as root, and enter these two lines at the bottom.

::

    crontab -e

::

    @reboot /usr/sbin/he-ipv4.sh >/dev/null 2>&1
    */15 * * * * /usr/sbin/he-ipv4.sh >/dev/null 2>&1

Your script will now run at every reboot as well as every quarter hour.
This should keep your IPv6 gateway functional with minimal amounts of
intervention.

Unfortunately, I've only been able to simulate my IPv4 address changing.
 I've not been able to test the script in a situation where the address
actually changed. However, in the simulated instance everything worked
perfectly!

**Edit (12/17/2011)**: I've finally created a github repo for this
script. The script is a bit over-complex as I had to work around some
shell scripting limitations. I welcome any forks as I am interested in
how you might be able to improve the script:

- https://github.com/theckman/he-ipv4-sh

While they are really no longer needed, here are the original two
pastebin URLs so you can view the source:

| - http://pastebin.com/T1tgSS8F
|  - http://pastie.org/1996914

If you have any feedback on this script, or any issues, don't hesitate
to leave a comment or open an issue on GitHub.

.. _GitHub: https://github.com/theckman/he-ipv4-sh
.. _blog post: http://blog.timheckman.net/2011/05/24/he-tunnelbroker-ipv6-gateway/

.. |image0| image:: /images/marv-office.jpg
   :target: /images/marv-office.jpg

Lighting Your Life with Go: The Go LIFX Client
##############################################
:date: 2016-01-07 22:17
:author: Tim Heckman
:copyright: 2016
:license: CC BY-SA
:category: coding
:tags: golang, go, LIFX, socket programming
:slug: lighting-your-life-with-go

Awhile back, I picked up a full set of LIFX light bulbs to install
around my apartment. The setup and operation is straightforward and
mostly works as expected. I've finally gotten around to wanting to
write some code to interface with them, and yet the ideal `Go package`_
has `some issues`_ with the latest LAN protocol.

There appear to be other concerns about how easily the code above can
be maintained, so I've decided to try and write my own implementation
of the LIFX V2 protocol in Go using the `published protocol documentation`_.
In addition to the published documentation, LIFX provides the
`Github repository`_ used to generate that site so you can track and
contribute fixes to the document.

The LAN Protocol
------------
The LIFX LAN protocol is meant for controlling your devices on the
local network. This requires you have some permanent device present if
you want persistent control of your bulbs. There is an `HTTP-based API`_
available if you don't have a device on the local network. I don't plan
on diving in to that API here.

The LAN protocol itself is fairly straightforward. Message payloads are
packed in to UDP packets and send over the LAN to the bulbs. The
protocol docs specify that the byte order for numerical data types is
`little-endian`_.

Each message consists of a header and a payload. The header itself
consists of three different sections which I'll go more in to later:

* `Frame section`_
* `Frame Address section`_
* `Protocol Header section`_

When interacting with bulbs, it is expected that multiple messages will
be transmitted. The protocol documentation also recommends limiting the
number of messages transmitted per second:

	Maximum recommended message transmit rate to a device: 20 per second

This gives you a general idea of how the protocol works. So let's dive
in to the actual message header sections. In this post I'll only be
providing an overview of each section. In later posts, while discussion
implementation, I will dive deeper in to each header. For now if you'd
like more details on the headers I'd suggest reading the protocol docs
directly.

Frame
_____
The first section of the message header is the `Frame section`_. This
section is used to define things like which LIFX protocol version the
message is, the size of the entire message, as well as how to use
certain fields in later header sections.

The Frame section is 32 bits (4 bytes) in total size, with some of the
fields being smaller than a ``uint8``. This means we'll need to pack
multiple values within a single ``uint16``.

Frame Address
_____________
The `Frame Address section`_ is actually used for a few different
purposes, the first of which is specifying the device we are targeting
for the message. In addition, this section is where you specify whether
the device should acknowledge and/or respond to your message. Lastly,
the message sequence number is provided in this header.

This header section is 128 bits (16 bytes) in total size, with one field
being smaller than a ``uint8``. In addition, this section and the next
both contain sizable blocks of reserved space. This section has 54 bits
of reserved space.

Protocol Header
_______________
The `Protocol Header section`_ is the simplest of the header sections.
The section itself consists mostly of reserved space, with one ``uint16``
value being used to specify the type of message payload.

This section is 96 bits (12 bytes) in size, with 80 bits (10 bytes) of
reserved space. None of the fields are different sizes than standard Go
``uint`` types, so no value packing is needed.

Payload
_______
This is the actual part of the message that informs the device what
to do. The content of this part of the message is entirely dependent on
the type of payload.

There are two message types available for the LIFX protocol at the time
of writing:

* `Device Messages`_
* `Light Messages`_

`Device Messages`_ are used to acquire the state of a device, without
knowing or caring about its type. It may seem strange that these weren't
just thrown under the `Light Messages`_. It was done this way because
the LIFX protocol engineers were trying to future-proof the protocol
for a future where LIFX may do more than bulbs.

`Light Messages`_ are for getting state of a specific device type, in
this case a light bulb. There are Light Messages that seem to duplicate
`Device Messages`_. However, they are extended to add support for a
duration field.

Writing The Client
----------
The Go client will need to accomplish a few tasks, the first of which is
to build structs that can be serialized and deserialized in a way that
meets the specs of the protocol. I plan on doing the work `on Github`_
to ensure its availability to others. The source code itself will be
released under the `BSD 3-Clause License`_.

Once the structs are built, the client will need to be written to
consume those structs and to communicate with LIFX bulbs. Once the bulb
communication code is working, we can begin implementing each of the
individual message types.

One additional goal of this client will be to ensure we can obtain
as many device-related metrics as possible.

I hope to try and document a nice amount of work that goes in to this
project. It will make an interesting exercise in writing blog posts
as well as help my own understanding of the protocol.

Until next time!

.. _Go package: https://github.com/wolfeidau/lifx
.. _some issues: https://github.com/wolfeidau/lifx/issues/13
.. _published protocol documentation: https://goo.gl/V497hL
.. _Github repository: https://github.com/LIFX/lifx-protocol-docs
.. _HTTP-based API: https://goo.gl/OTTCEA
.. _little-endian: https://en.wikipedia.org/wiki/Endianness#Little-endian
.. _Frame section: https://goo.gl/ZrmDGa
.. _Frame Address section: https://goo.gl/3X9YGs
.. _Protocol Header section: https://goo.gl/InumRv
.. _Device Messages: https://goo.gl/sZku3o
.. _Light Messages: https://goo.gl/MqVVlb
.. _on Github: https://github.com/theckman/go-lifx
.. _BSD 3-Clause License: https://opensource.org/licenses/BSD-3-Clause

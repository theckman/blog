The Go LIFX Client Part 2
##########################
:date: 2016-01-12 00:05
:author: Tim Heckman
:copyright: 2016
:license: CC BY-SA
:category: coding
:tags: golang, go, LIFX, socket programming
:slug: go-lifx-client-part-2

The benefits of writing a client that requires some socket work, is that
you have a clear starting point: build code that can communicate in a
a way that is fully compliant to the `protocol spec`_. For the LIFX client,
the first step towards that is generating valid UDP packets.

Let's go in the order things are sent in a single message packet. First
up: the header. The header itself is 32 bytes (256 bits) in size and has
three sections:

* Frame
* Frame Address
* Protocol Header

In this post, we'll focus primarily on marshaling and unmarshaling the
``Frame`` section. The rest of the sections will get the same treatment,
so it makes sense to focus primarily on one.

Designing The Header
--------------------
Each header section should be its own struct, which will allow you to
marshal individual portions of the header. Following the order of the
header sections, this is the simplest header definition:

.. code-block:: go

	type Header struct {
		Frame          *Frame
		FrameAddress   *FrameAddress
		ProtocolHeader *ProtocolHeader
	}

As there are three sections that make up a message header, we will need to
implement three structs that can be marshaled and unmarshaled to and from
the packet format respectively. The marshaling should be done by each
individual type, with the marshal function of ``Header``  invoking them.

Frame Struct
------------
The first struct to implement is the one for the Frame_. It consists of a
few fields:

============= ====== ======== =============
    Field      Bits    Type    Description
============= ====== ======== =============
  Size         16     uint16   Size of entire message in bytes including this field
  Origin       2      uint8    Message origin indicator: must be zero (0)
  Tagged       1      bool     Determines usage of the Frame Address target field
  Addressable  1      bool     Message includes a target address: must be one (1)
  Protocol     12     uint16   Protocol number: must be 1024 (decimal)
  Source       32     uint32   Source identifier: unique value set by the client, used by responses
============= ====== ======== =============

As you can see, some of these fields are smaller than the data type we
can use to represent them. The total size of the ``Origin``, ``Tagged``,
``Addressable``, and ``Protocol`` fields is 16 bits which means we will
need to pack them in to a ``uint16``.

Let's transcribe the field table above in to a Go struct:

.. code-block:: go

	type Frame struct {
		Size        uint16
		Origin      uint8
		Tagged      bool
		Addressable bool
		Protocol    uint16
		Source      uint32
	}

To pack this in a UDP packet, we'll need to write a function that marshals
the struct and returns a byte slice. While we know the protocol spec
defines the byte order as being little-endian, we'll take the byte order
as a function parameter:

.. code-block:: go

	func (f *Frame) MarshalPacket(order binary.ByteOrder) ([]byte, error) {}

Frame: Marshaling
_________________
With our marshal function signature defined, let's start with doing some
input validation on the struct fields that will need to have some bits
truncated off. The ``Origin`` field is only 2 bits in size, and the
``Protocol`` field is only 12 bits in size. Within our marshaling function,
we'll need to check the value of these fields to ensure they don't overflow
their specified size.

The first step is to define some variables for the errors we will be
returning. By defining them as exported variables on the package, consumers
can assert whether or not it's an error indicating that a specific field has
overflowed:

.. code-block:: go

	// ErrFrameProtocolOverflow is the error returned when the Frame.Protocol value is too large
	var ErrFrameProtocolOverflow = fmt.Errorf("The Protocol field cannot be larger than %d, please choose another value (suggested: 1024)", ^uint16(0)>>4)

	// ErrFrameOriginOverflow is the error returned when the Frame.Origin value is too large
	var ErrFrameOriginOverflow = fmt.Errorf("The Origin field cannot be larger than %d; should be set to 0", ^uint8(0)>>6)

Now that the errors are defined at the top of the package, let's begin writing
our marshal function with the field value validation:

.. code-block:: go

	func (f *Frame) MarshalPacket(order binary.ByteOrder) ([]byte, error) {
		if frame.Origin > ^uint8(0) >> 6 { // 3
			return nil, ErrFrameOriginOverflow
		}

		if frame.Protocol > ^uint16(0) >> 4 { // 4095
			return nil, ErrFrameProtocolOverflow
		}

If you're not familiar with bit shifting, the above lines of code may not
make much sense. For the first if-statement, we create a new uint8
(``uint8(0)``) and then XOR (``^``) it. After doing that, we shift the
value to the right 6 times (``>> 6``) thus giving us our maximum value for
the Origin field (``3``). Here's how it works:

.. code-block:: go

	// this shows how each position in a binary number defines its value
	// pos:  1    2   3   4  5  6  7  8
	// val: 128  64  32  16  8  4  2  1
	// So:
	// 00000011 = 3
	// 10000000 = 128

	x := uint8(0)  // 00000000 (0)
	x = ^x         // 11111111 (255)
	x >> 2         // 00111111 (63)
	x >> 6         // 00000011 (3)
	x << 2         // 11111100 (252)

	x = 1          // 00000001 (1)
	x = x << 2     // 00000100 (4)
	x = x << 3     // 00100000 (32)
	x = x >> 2     // 00001000 (8)

That means that the value of the ``Origin`` field can be, at most, ``3``.
It also means that the value of the ``Protocol`` field can only be, at
most, ``4095``.

Now that we've confirmed our values won't overflow the sizes defined in
the spec, we can write the binary values. For this we'll use the Write_
function available within the binary_ package. The ``Write`` function
writes to an ``io.Writer``, so we'll use a `bytes.Buffer`_ for writing.
Because the ``Write`` method of a ``bytes.Buffer`` is defined as a pointer
method, we need to be sure to use a ``*bytes.Buffer``.

According to the spec the first field to be of the Frame section is the
``Size`` field, so let's write that as well:

.. code-block:: go

	buf := &bytes.Buffer{}

	// write the Size field using the byte order in order to buf
	if err := binary.Write(buf, order, frame.Size); err != nil {
		return nil, err
	}

If we were to return ``buf.Bytes()`` at this point, it would be 2 bytes
long and would be the value of the ``Size`` field.

For the next values within the Frame section, we need to do some bit
shifting as the protocol only requires 2 bits worth of information from
the ``Origin`` field and only 12 from the ``Protocol`` field. Those fields
also need to be combined with the ``Tagged`` and ``Addressable`` bool
fields to form a single ``uint16``:

.. code-block:: go

	// the next 16 bit value is multiple fields packed together:
	// Origin: 2
	// Tagged: 1
	// Addressable: 1
	// Protocol: 12
	mid := uint16(frame.Origin)<<14 | frame.Protocol<<4>>4

	if frame.Tagged {
		mid = mid | (1 << 13)
	}

	if frame.Addressable {
		mid = mid | (1 << 12)
	}

	if err := binary.Write(buf, order, mid); err != nil {
		return nil, err
	}

Looking at this code for the first time can be a bit confusing, so
we will break each part down. Before doing that, let's go over some
bitwise math if you are aren't very familiar with it.

The bitwise-and (``&``) operator compares the individual bits and only
evaluates the result as ``1`` if both values are ``1`` as shown here:

.. code::

	01101010 &
	01011010
	--------
	01001010

The bitwise-or (``|``) operator is similar to the bitwise-and, except it
evaluates to ``1`` if either value is ``1``.

.. code::

	01101010 |
	01011010
	--------
	01111010

Let's now explain how each part of the above code snippet works.

.. code-block:: go

		mid := uint16(frame.Origin)<<14 | frame.Protocol<<4>>4

The ``Origin`` field is defined in the struct as a ``uint8``, so we first
need to convert this value to a ``uint16``. The protocol states that the
``Origin`` field is only two (``2``) bits in size, so we'll need to set
it to the high two bits of the ``uint16``. We shift the value to the left
by 14 bits (``16 - 2 = 14``).

The spec defines the ``Protocol`` field value as only being ``12`` bits
in length, meaning we need to only use the low 12 bits of the value. For
this, we shift the value 4 bits to the left, and then 4 bits back to the
right, effectively nulling out the high 4 bits of the value. Because the
high 4 bits becomes zero, it won't impact the end result of the value as
evaluated by the bitwise-or.

We then bitwise-or the two values together to pack both the ``Origin``
and ``Protocol`` fields in to the ``uint16`` value.

The next line is us adding the packed value of the ``Tagged`` and
``Addressable`` fields to the ``uint16`` value:

.. code-block:: go

	if frame.Tagged {
		mid = mid | (1 << 13)
	}

	if frame.Addressable {
		mid = mid | (1 << 12)
	}

Because both the ``Tagged`` and ``Addressable`` fields are ``bool``, they
are effectively ``1`` bit worth of data. So in this section of code, we
set the lowest most bit (``1``) if the field is true.

According to the protocol spec, the ``Tagged`` value is at the third-highest
bit, and the ``Addressable`` value is at the fourth-highest bit. So for the
``Tagged`` field we will need to shift by ``13`` bits (``16 - 3 = 13``) and
for the ``Addressable`` field we will need to shift by ``12`` bits
(``16 - 4 = 12``).

So we shift the number ``1`` by the number of bits based on the field, and
then bitwise-or the value with the current value of the ``mid`` variable.
This means we've now packed both the ``Tagged`` and ``Addressable`` field,
so all we need to do is write the data to the ``*bytes.Buffer``:

.. code-block:: go

	if err := binary.Write(buf, order, mid); err != nil {
		return nil, err
	}

We then only need to write the ``Source`` field to the buffer, and we are
finished marshaling:

.. code-block:: go

	if err := binary.Write(buf, order, frame.Source); err != nil {
		return nil, err
	}

	return buf.Bytes(), nil

At this point a byte slice, containing the fully marshaled ``Frame``,
is returned from the function and can be written to a UDP connection.

Frame: Unmarshaling
___________________
Unmarshaling the packet actually requires less code than marshaling. We
only need to read the values in, and assign them to the struct fields.

The first step is to build the signature of the method used to unmarshal
the packet. In the marshaling function, we used the ``binary.Write()``
function for writing. The binary_ package has a Read_ function that takes
an ``io.Reader`` as its source to read binary data. As we are using this
function to read the data, we should accept an ``io.Reader`` as a parameter
to our unmarshaling function. In addition to that parameter, we will include
an order parameter to specify the byte order (little- or big-endian):

.. code-block:: go

	func (frame *Frame) UnmarshalPacket(data io.Reader, order binary.ByteOrder) error {}

We only need to return an error from this function as the results will be
set on the struct fields themselves.

The ``Read`` function does not return the data read, but instead sets it
to a pointer provided to the function. So, to read the ``Size`` field we
would have the following code:

.. code-block:: go

	func (frame *Frame) UnmarshalPacket(data io.Reader, order binary.ByteOrder) error {
		if err := binary.Read(data, order, &frame.Size); err != nil {
			return err
		}

We provide the function a pointer to the value of ``frame.Size``, which
it then uses to set the value. For fields that required no packing,
unmarshaling is straightforward and easy.

For the value we packed, we first need to read the full value from the
``io.Reader`` and then unpack the individual values. In our case we need
to read the ``uint16`` out of the ``io.Reader`` and then unpack the
values from it.

.. code-block:: go

	var u16 uint16
	if err := binary.Read(data, order, &u16); err != nil {
		return err
	}

Once the data has been read, we have the ``u16`` variable which can be
used to access the individual fields:

.. code-block:: go

	frame.Origin = uint8(u16 >> 14)    // get top 2 bits

Remember that or the ``Origin`` field, we only care about 2 bits worth
of information, so we can just shift the value over and convert it back
to a ``uint8`` (``16 - 2 = 14``).

In the marshal function, we had to explicitly convert the bool fields to
``uint`` values. When Unmarshaling, we can just bitshift and see if the
value is equivalent to ``1`` (``true``):

.. code-block:: go

	frame.Tagged = u16>>13&1 == 1      // get 3rd bit and eval if it's true
	frame.Addressable = u16>>12&1 == 1 // get 4th bit and eval if it's true

Here, we shift the values over so the field we care about is the right-most
bit, and then we bitwise-and it with ``1``. This ensures that only the
right-most bit is used in the evaluation. We then evaluate as to whether
the value is equal to ``1``; if so the field becomes ``true``.

The last field we need to unpack is the ``Protocol`` field:

.. code-block:: go

	frame.Protocol = u16 << 4 >> 4     // get bottom 12 bits

We end up taking the same action on the ``u16`` variable that we took
on the ``Protocol`` field when marshaling to get its value. We use a left
and right bitshift to trim off the top 4 bits.

The final step is to read the ``Source`` field from the ``io.Reader``:

.. code-block:: go

	return binary.Read(data, order, &frame.Source)

Because the value gets set on the ``frame.Source`` field, we can just
return the error from the last ``Read`` function.

Conclusion
----------
In the end, we've ended up with a ``struct`` type that can be marshaled
to and from a format as specified by the `protocol spec`_. There are
individual functions for each action, and there is some field validation
in the marshaling function to ensure its values can be packed without
losing data.

We've also briefly explained some bitwise math operators, shown
how bitshifting works, and displayed how you can combine binary values
together efficiently by using both.

In the next post of the series we'll continue by implementing unit tests
for the ``Frame`` marshal and unmarshal functions. Following that we'll
continue building the remaining header sections and move on to thinking
about how we'll handle reading and writing payloads.

.. _protocol spec: https://goo.gl/s2utlU
.. _Frame: https://goo.gl/USjWMp
.. _Write: https://godoc.org/encoding/binary#Write
.. _binary: https://godoc.org/encoding/binary
.. _bytes.Buffer: https://golang.org/pkg/bytes/#Buffer
.. _Read: https://golang.org/pkg/encoding/binary/#Read

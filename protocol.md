# Specification of a peer-to-peer secure communication protocol

## 0. Purpose

The purpose of this protocol is to replace the use of SSL with something
similar but not identical. The similarity is so that security guarantees are
retained. The difference is such that should the protocol's details remain
proprietary (initiation of the shared secret, establishing of nonce and they
key rotations) or include some level of per-use obfuscation, it cannot be
easily crypto-analyzed. While the motto is "in cryptography, everything should
be open but the keys", proprietary protocols hold the advantage of obscurity.
In this particular case, because the underlying cryptography is standardize,
this is not "security via obscurity", but obscurity added to existing security
guarantees.

The protocol can operate over standard (unencrypted) Unix/BSD TCP sockets,
or over encrypted channels, such as multiplexed SSH connections and other TCP
style protocols (protocols with certain guarantees like TCP provides) such
as proper SSL itself.

Because the protocol deals with encrypted data, requisite data integrity,
and sequence ordering, each transmission of a FRAME (a self-contained packet
of data) involves both parties keeping track of the sequence of messages
that the other party sends in addition to what they send. The protocol dictates
that a 32-bit unsigned integer is used for the sequence numbering, starting
from 1, and wrapping when the 32-bit maximum number is reached.


## 1. Parties

There are two parties communicating via a socket with no distinction between
them. Each party can be involed in sending and receiving simultanuously, but
it is up to the implementation to handle frame reception and transmission.
A party that is sending is in a SEND state. They can also be in a RECV state
when in the process of receiving a frame, or in a WAIT stage where they are
listening for messages. This distinctions are made so that it is made explicit
that the implementation should not be multi-threaded (one process sending data
while the other one is receiving over the same socket).

One distinction is made between parties, determined by the higher level
protocol using this protocol for communication, and its purpose is to dictate
which part of the shared secret each party will use (the "top" or "bottom" of
the shared secret block. The idea is that each party encrypts using a separate
part of the shared secret, nonce, and AES-CTR counter sequence (see below).


## 2. Frame

A FRAME can arrive at any time to either party; there can be monitoring of the
socket for "wants to write" activity. Each frame is entirely encrypted; the
HEADER, PAYLOAD, HMAC are concatanated and subsequently encrypted. Each frame
is encrypted with AES-CTR encryption based on an agreed upon setup ahead of
time, which includes nonce ("iv"), counter, and key rotation using a shared
secret (see AES in CTR mode in an alternate source for more information).

The frame (including header) is broken into 16 byte chunks, which are the
finest granularity that the AES 256-bit encryption can support. Because each
frame's chunk can be deciphered by AES-CTR using the nonce "iv" and counter,
the header can be entirely deciphered before the payload is received. This
allows for the frame's length to be read when deciphering only the header
(the header must contain the size of the payload -- see below).

The header is 16 bytes at the start of the frame. It contains the length
in the first 4 bytes, a protocol-specific code in the next 4 bytes, and the
number (in sequence) of the frame in the next 4 bytes. Padding of 4 bytes
contains a checksum for the header only, which is formed by bitwise XORing
the three prior 4-byte fields.

The payload follows the header, and is *LENGTH* bytes long. The message
authentication code (HMAC) follows the payload as the last part of a frame.
The HMAC is 8 bytes in length; these are the 8 most significant bytes of the
SHA-256 hash of the concatanation of the header and payload, while also
preserving the SHA-256 internal state from the prior frame. Hashing each
frame requires reseting the starting point of the next buffer to be processed
as if it starts at byte zero, but the internal state of the hashing function
is left intact from the previous use.

A frame may not fall on the 16-byte boundary of the AES-CTR stream cipher.
The protocol does not require the full AES-CTR cipher pad to be used for
bitwise XOR ciphering, and how this is handled is up to the implementation.
(Sufficient padding bytes past the boundary can be added to the frame for
the stream cipher to work properly if the specific AES-CTR implementation
needs them, without affecting the protocol.)

Here is a diagram of the frame and its parts:

```
 |------ FRAME ------------------------------------------------------------ |
 |  HEADER      |  PAYLOAD                                    |  HMAC       |
 |--------------------------------------------------------------------------|
 |<- 16 bytes ->|<- LENGTH bytes ---------------------------->|<- 8 bytes ->|
 |--------------------------------------------------------------------------|
```

Here is a diagram of the frame's header and its parts:
```
 |------ HEADER -----------------------------------------|
 | LENGTH      | CODE        | SEQ. NUMBER | CHEKSUN     | 
 |-------------------------------------------------------|
 |<- 4 bytes ->|<- 4 bytes ->|<- 4 bytes ->|<- 4 bytes ->|
 |-------------------------------------------------------|
```

The *LENGTH*, *CODE*, and *SEQUENCE NUMBER* are given in "network byte order"
(big endian).


## 3. Transmission and reception

The protocol must allow flexibility for how the sender and receiver structure
the process of sending and receiving large payloads that exceed the frame size.
For example, the sender may know in advance that a "large" payload is going to
be sent. The implementation should be able to break the large payload into
several frames, while the process is hidden behind a single API call to send.
The same holds true for the receiver of large payloads, where a single API
call may receive several frames into a large buffer.

This protocol allows for a frame to exceed the size of TCP packets, but it
cannot exceed the hard limit of the 32-bit unsigned integer for its length.

The application must receive assurances for the integrity of the data that
it receives in its buffer. If data integrity or other violation occurs, the
sanctified data found in the buffer when the API call returns are considered
to be valid, but the API will return an error. Similarly, the application
can request for a number of bytes to be read that are larger than the number
that the API call receives before reaching a pre-defined timeout. In this
case, the API will notify the caller with an "error" that the peer is slow,
but all data returned in the buffer will be sanctified. Another case can be
encountered where the application requests a number of bytes that does not
fall on a frame boundary. In this case. the API will return the number of
bytes that were contained up to the end of the last frame for which the HMAC
could be checked; any bytes remaining will stay in the kernel's buffers until
a subsequent API read operation is performed, and no error will be returned.


## 4. Initiation of frame

A sender has to have completed sending or receiving a full frame before they
can begin sending a new frame. The same holds true for the receiver. And
therefore, it is implied that a higher-level protocol that coordinates their
behaviour and adheres to these guidelines is in operation. The higher level
operations decide how to handle errors.

IN 2025/06/05-06/19


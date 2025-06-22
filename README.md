# iznoSSL
Experimenting with an organic secure socket layer protocol 


SUMMARY

Experimenting with an organic secure socket layer transport protocol that runs
on top of other protocols (such as plain TCP/IP or SSH tunnels). This is _not_
SSL or TLS, but it is very similar, and built for a very specific purpose.

You are not supposed to use this code. There is nothing to see here. You are
supposed to be using the code that people build for production. You have been
warned.

With this project, I'm also trying to test the power of artificial intelligence
in the flavour of large and small langauge models (LLM/SLM) in a strange but
hopeful manner. Persons with authority in modern artificial intelligence (most
notably Andrej Karpathy) have proclaimed that "we will not be coding in a
programming langauge, but in English", hinting at the idea that we should be
able to dictate to a computer what we want to have programmed, and it would
program it for us. When that becomes a reality (not "if"), an LLM should be
able to take the specification of a communication protocol such as the one
that is of interest to me, and produce the software that is its implementation.
It has also been said that a "markdown" specification document --instead of a
PDF-- should be easier for an LLM's tokenizer to absorb. This repository wants
to be that, the specification that an LLM will absorb to produce code, in
addition to some other objectives.

You are looking at a communication protocol that employs encryption in order
to keep data from being evesdropped or tampered with. A good protocol, and
one that is as close to SSL (and TLS) as possible, has several attributes:
(1) authenticates the identity of the parties involved (at least one, the
server, if used in a client-server context, but not necessarily just one),
(2) establish a common secret that is ephemeral to a given session,
(3) encrypt all data communicated after the common secret is exchanged, and
(4) provide a mechanism to authenticate data such that the receiver can
verify data integrity.
I am not including (1) or (2) here for now, but (3) and (4) are in here.

I articulated the protocol in a textual description that is supposed to be
self-contained, but does require some knowledge of the jargon and of some
specific algorithms (the SHA-256 hashing function and the AES-CTR). You
can look at online information about both. I have my own implementations,
which I have built for use in unrelated projects (SHA-256 shows up in many
places, like cryptocurrencies, as does AES in CTR and CBC modes).

For those in-the-know, I am using AES in CTR mode to form an encryption pad
sequence that is used to XOR the plaintext. The AES uses a nonce that has a
scheme behind its rotation that depends on entropy derived from the shared
secret. Its CTR (counter mode) operation is standard. I use a SHA-256 hashing
function as the HMAC, and the SHA-256 state includes all prior history of the
session. The shared secret is negotiated based on a standard Diffie-Hellman
exchange of a 8192-bit entropy pad. In this exchange, I use an ECDSA scheme
for signing during the exchange, and also for peer authentication. Neither
of those parts are included in this repository.

You may have heard that "there are two types of people who build their own
encryption: the very smart ones and the very dumb ones." I do not consider
myself to be very smart, so there is that...

IN 2025/06/21


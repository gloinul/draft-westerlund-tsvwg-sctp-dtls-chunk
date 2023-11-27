---
docname: draft-westerlund-tsvwg-sctp-dtls-chunk-latest
title: Stream Control Transmission Protocol (SCTP) DTLS Chunk
abbrev: SCTP DTLS Chunk
obsoletes:
cat: std
ipr: trust200902
wg: TSVWG
area: Transport
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"

venue:
  group: Transport Area Working Group (tsvwg)
  mail: tsvwg@ietf.org
  github: gloinul/draft-westerlund-tsvwg-sctp-DTLS-chunk

author:
-
   ins:  M. Westerlund
   name: Magnus Westerlund
   org: Ericsson
   email: magnus.westerlund@ericsson.com
-
   ins: J. Preuß Mattsson
   name: John Preuß Mattsson
   org: Ericsson
   email: john.mattsson@ericsson.com
-
   ins: C. Porfiri
   name: Claudio Porfiri
   org: Ericsson
   email: claudio.porfiri@ericsson.com

informative:
  RFC8446:

  I-D.westerlund-tsvwg-sctp-DTLS-handshake:
    target: "https://datatracker.ietf.org/doc/draft-westerlund-tsvwg-sctp-dtls-handshake/"
    title: "Datagram Transport Layer Security (DTLS) in the Stream Control Transmission Protocol (SCTP) DTLS Chunk"
    author:
      -
       ins:  M. Westerlund
       name: Magnus Westerlund
       org: Ericsson
       email: magnus.westerlund@ericsson.com
      -
       ins: J. Preuß Mattsson
       name: John Preuß Mattsson
       org: Ericsson
       email: john.mattsson@ericsson.com
      -
       ins: C. Porfiri
       name: Claudio Porfiri
       org: Ericsson
       email: claudio.porfiri@ericsson.com
    date: June 2023


normative:
  RFC2119:
  RFC4895:
  RFC5061:
  RFC8126:
  RFC8174:
  RFC9147:
  RFC9260:

  TLS-CIPHER-SUITS:
    target: "https://www.iana.org/assignments/tls-parameters/tls-parameters.xhtml#tls-parameters-4"
    title: "TLS Cipher Suites"
    date: November 2023

--- abstract

This document describes a method for adding Cryptographic protection
to the Stream Control Transmission Protocol (SCTP). The SCTP DTLS
chunk defined in this document is intended to enable communications
privacy for applications that use SCTP as their transport protocol and
allows applications to communicate in a way that is designed to
prevent eavesdropping and detect tampering or message forgery.

Applications using SCTP DTLS chunk can use all transport
features provided by SCTP and its extensions but with some limitations.

--- middle

# Introduction {#introduction}

   This document defines a DTLS chunk for the Stream Control
   Transmission Protocol (SCTP), as defined in {{RFC9260}}.

   This specification defines the actual DTLS chunk, how to enable
   it usage, how it interacts with the SCTP association establishment
   to enable endpoint authentication, key-establishment, and key
   updates.

   The DTLS chunk is designed to enable mutual
   authentication of endpoints, data confidentiality, data origin
   authentication, data integrity protection, and data replay
   protection for SCTP packets after the SCTP association has been
   established. It is dependent on a Key Management function that is
   defined seperately to achieve all these capabilities. The
   keymanagement function uses an API to provision the SCTP
   association's DTLS chunk protection with key-material to enable and
   rekey the protection operations.

   Applications using SCTP DTLS chunk can use most transport features
   provided by SCTP and its extensions. However, there can be some
   limitations or additional requirements for them to function such as
   those noted for SCTP restart and use of Dynamic Address
   Reconfiguration, see {{sec-asconf}} and {{sec-restart}}. Due to its
   level of integration as discussed in next section it will provide
   its security functions on all content of the SCTP packet, and will
   thus not impact the potential to utilize any SCTP functionalities
   or extensions that are possible to use between two SCTP peers with
   full security and SCTP association state.

   DTLS is considered version 1.3 as specified in {{RFC9147}} whereas
   other versions are explicitely not part of this document.

# Conventions

{::boilerplate bcp14}

# Overview

## Protocol Overview {#protocol-overview}

The DTLS chunk is defined as a method for secure and confidential
transfer for SCTP packets.  This is implemented inside the SCTP
protocol, in a sublayer between the SCTP common header handling and
the SCTP chunk handling.  Once an SCTP packet has been received and
the SCTP common header has been used to identify the SCTP association,
the DTLS chunk is sent to the DTLS protection operator that will
return the SCTP payload containing the unprotected SCTP chunks, those
chunks will then be handled according to their SCTP protocol
specifications. {{sctp-DTLS-chunk-layering}} illustrates the DTLS
chunk layering in regard to SCTP and the Upper Layer Protocol (ULP).

~~~~~~~~~~~ aasvg
+---------------+ +--------------------+
|               | |       DTLS 1.3     |  Keys
|      ULP      | |                    +-------------.
|               | |   Key Management   |              |
+---------------+-+---+----------------+            --+-- API
|                     |                 \    User     |
|                     |                  +-- Level    |
| SCTP Chunks Handler |                      Messages |
|                     |                               |
|                     | +-- SCTP Unprotected Payload  |
|                     |/                              |
+---------------------+    +---------------------+    |
|        DTLS         |    |       DTLS 1.3      |    |
|        Chunk        |<-->|                     |<--'
|       Handler       |    | Protection Operator |
+---------------------+    +---------------------+
|                     |\
| SCTP Header Handler | +-- SCTP Protected Payload
|                     |
+---------------------+
~~~~~~~~~~~
{: #sctp-DTLS-chunk-layering title="DTLS Chunk Layering
in Regard to SCTP and ULP" artwork-align="center"}

Use of the DTLS chunk is defined per SCTP association.

On the outgoing direction, once the SCTP stack has created the
unprotected SCTP packet payload containing control and/or DATA chunks,
that payload will be sent to the DTLS protection Operator to be
protected. The format of the protected payload is a DTLS 1.3 record
encapsulated in a DTLS chunk.

The SCTP protection operator performs protection operations on the
whole unprotected SCTP packet payload, i.e., all chunks after the SCTP
common header. Information protection is kept during the lifetime of
the association and no information is sent unprotected except than the
initial SCTP handshake, initial DTLS handshake, the SCTP common
header, the SCTP DTLS chunk header and the SHUTDOWN-COMPLETE chunk.

SCTP DTLS chunk capability is agreed by the peers at the
initialization of the SCTP association. Until the DTLS protection has
been keyed only plain text key-management traffic using a special PPID
may be flow, no ULP traffic. The key management function uses an API
to key the DTLS protection operation function. Usage of the DTLS 1.3
handshake for initial mutual authentication and key establishment as
well a periodic re-authentication and rekeying with Diffe-Hellman of
the DTLS chunk protection is defied in a sepearte document
{{I-D.westerlund-tsvwg-sctp-DTLS-handshake}}.

When the endpoint authentication and key establishment has been
completed, the association is considered to be secured and the ULP is
informed about that. From this time on it's possible for the ULPs to
exchange data securely.

DTLS chunks will never be retransmitted, retransmission is implemented
by SCTP endpoint at chunk level as in the legacy. DTLS replay
protection will be used to supress duplicated DTLS chunks, however a
failure to prevent replay will only result in duplicated SCTP chunks and
will be handled as duplicated chunks by SCTP endpoint in the same way
a duplicated SCTP packet with those SCTP chunks would have been.


## DTLS Considerations {#DTLS-engines}

DTLS 1.3 is assumed to be implemented by a key handler and a
protection operator. The key handler implements the key management by
means of handshake, it's properly configured using secrets. The way
DTLS 1.3 is configured with secrets is part of the another document
{{I-D.westerlund-tsvwg-sctp-DTLS-handshake}}. The DTLS protection
operator is the encryption engine of DTLS 1.3, it's configured with
the needed keys by the key handler.

SCTP DTLS chunk directly uses DTLS 1.3 protection operator by
requesting protection and unprotection of a buffer, in particular the
protection buffer should never exceed the possible SCTP packet size
thus DTLS protection operator needs to be aware of the PMTU (see {{pmtu}}).

The key management part of the DTLS 1.3 is the set of data
and procedures that take care of key distribution, verification, and
update. SCTP DTLS provides support for in-band key management, on
those cases the Protection Engines uses SCTP DATA chunks identified
with a dedicated Payload Protocol Identifier.

During protection engine initialization, that is after the SCTP
association reaches the ESTABLISHED state (see {{RFC9260}} Section 4),
but before DTLS 1.3 key-management has completed and the
Protected Assocation Parameter Validation is completed, any in-band
Key Management MAY use SCTP user message that SHALL use the SCTP-DTLS
PPID (see {{iana-payload-protection-id}}). These DATA chunks
SHALL be sent unprotected by the protection engine as no keys have
been established yet. As soon as the protection engine has been
intialized and the validation has occured, further DTLS 1.3 handshakes
being sent using SCTP use messages with the
SCTP-DTLS PPID, will have their message protected inside SCTP
DTLS chunk protected with the currently established key.
SCTP DTLS chunk state evolution is described in {{init-state-machine}}.

DTLS related procedures MAY use the Flags byte provided by the
DTLS chunk header (see {{sctp-DTLS-chunk-newchunk-crypt-struct}})
for their needs. Details of the use of Flags are specified within
this document in the relevant sections.

The SCTP common header is assumed to be implicitly protected by the
protection engine. This protection is based on the assumption that
there will be a one-to-one mapping between SCTP association and
individually established security contexts.

## SCTP DTLS Chunk Buffering and Flow Control {#buffering}

DTLS 1.3 operations and SCTP are asynchronous, meaning that the
protection operator may deliver the decrypted SCTP Payload to the SCTP
endpoint without respecting the reception order.  It's up to SCTP
endpoint to reorder the chunks in the reception buffer and to take
care of the flow control according to what specified in
{{RFC9260}}. From SCTP perspective the DTLS chunk processing is part
of the transport network.

Even though the above allows the implementors to adopt a
multithreading design of the protection engines, the actual
implementation should consider that out-of-order handling of SCTP
chunks is not desired and may cause false congestion signals and
trigger retransmissions.

## PMTU Considerations {#pmtu}

The addition of the DTLS chunk to SCTP reduces the room for payload,
due to the size of the DTLS chunk header, padding, and authentication tag.  Thus, the SCTP
layer creating the plain text payload needs to know about the overhead
to adjust its target payload size appropriately.

On the other hand, the protection operator needs to be informed about
the PMTU by removing from the value the sum of the common SCTP header
and the DTLS chunk header. This implies that SCTP can propagate
the computed PMTU at run time specifically.

From SCTP perspective, if there is a maximum size of plain text data
that can be protected by the protection engine that must be
communicated to SCTP. As such a limit will limit the PMTU for SCTP to
the maximum plain text plus DTLS chunk and algorithm overhead plus
the SCTP common header.

## Congestion Control Considerations {#congestion}

The SCTP mechanism for handling congestion control does depend on
successful data transfer for enlarging or reducing the congestion
window CWND (see {{RFC9260}} Section 7.2).

It may happen that protection engine discards packets due to internal
checks or because it has detected a malicious attempt. As those
packets do not represent what the peer sent, it is acceptable to
ignore them, although in-situ modification on the path of a packet
resulting in discarding due to integrity failure will leave a gap, but
has to be accepted as part of the path behavior.

The protection operator shall not interfere with the SCTP congestion
control mechanism, this basically means that from SCTP perspective
the congestion control is exactly the same as how specified
in {{RFC9260}}.

## ICMP Considerations {#icmp}

The SCTP implementation will be responsible for handling ICMP messages
and their validation as specified in {{RFC9260}} Section 10. This
means that the ICMP validation needs to be done in relation to the
actual sent SCTP packets with the DTLS chunk and not the unprotected
payload. However, valid ICMP errors or information may indirectly be
provided to the protection operator, such as an update to PMTU values
based on packet to big ICMP messages.

## Path Selection Considerations {#multipath}

When an Association is multihomed there are multiple paths between Endpoints.
The selection of the specific path to be used at a certain time belongs
to SCTP protocol that will decide according to {{RFC9260}}.
The Protection Operator shall not influence the path selection algorithm,
actually the Protection Operator will not even know what path is being used.

## Dynamic Address Reconfiguration Considerations  {#sec-asconf}

When using Dynamic Address Reconfiguration {{RFC5061}} in an SCTP
association using DTLS Chunk the ASCONF chunk is protected, thus it
needs to be unprotected first, furthermore it MAY come from an unknown
IP Address.  In order to properly address the ASCONF chunk to the
relevant Association for being unprotected, Destination Address,
Source, Destination ports and VTag shall be used. If the
combination of those parameters is not unique the implementor MAY
choose to send the DTLS Chunk to all Associations that fit with the
parameters in order to find the right one. The association will
attempt de-protection operations on the DTLS chunk, and if that is
successful the ASCONF chunk can be processed.

The section 4.1.1 of {{RFC5061}} specifies that ASCONF message are
required to be sent authenticated with SCTP-AUTH {{RFC4895}}.
For SCTP associations using DTLS Chunks this
results in the use of redundant mechanism
for Authentication with both SCTP-AUTH and the DTLS Chunk. We
recommend to amend {{RFC5061}} for including DTLS Chunks as
Authentication mechanism for ASCONF chunks.

## SCTP Restart Considerations  {#sec-restart}

This section deals with the handling of an unexpected INIT chunk during an
Association lifetime as described in {{RFC9260}} section 5.2 The introduction of
DTLS CHUNK opens for two alternatives depending on if the protection engine
preserves its key material state or not.

When the encryption engine can preserve the key material, meaning that
encrypted data belonging to the current Association can be encrypted and
decrypted, the request for SCTP Restart SHOULD use INIT chunk in DTLS chunk.

When the DTLS context is not preserved, the SCTP Restart can only be
accomplished by means of plain text INIT.  This opens to a
man-in-the-middle attack where a malicious attacker may theoretically
generate an INIT chunk with proper parameters and hijack the SCTP
association. This should only be allowed under explictly configured
policy.

Editors note: The whole section related to SCTP Restart requires
further work, though.

### INIT chunk in DTLS chunk

This procedure as currently defined updates {{RFC9260}}, thus this
part requires agreements and possibly a new approach.

If the key material associated with the SCTP association has been
preserved the peer aiming for a SCTP Restart can still send DTLS
chunks that can be processed by the remote peer.  In such case the
peer willing to restart the Association SHOULD send the INIT chunk in
a DTLS chunk and encrypt it.  At reception of the DTLS chunk
containing INIT, the receiver will follow the procedure described in
{{RFC9260}} section 5.2.2 with the exception that all the chunks will
be sent in DTLS chunks.

An endpoint supporting SCTP Association Restart and implementing DTLS
Chunk MUST accept receiving SCTP packets with a verification tag with
value 0.  The endpoint will attempt to map the packet to an
association based on source IP address, destination address and
port. If the combination of those parameters is not unique the
implementor MAY choose to send the DTLS Chunk to all Associations that
fit with the parameters in order to find the right one. Note that type
of trial decrypting of the SCTP packets will increase the resource
consumption per packet with the number of matching SCTP associations.

Note that the Association Restart will update the verification tags
for both endpoints.  At the end of the unexpected INIT handshaking the
receiver of INIT chunk SHALL perform a rekeying as soon
as possible to verify the peer identity.

### INIT chunk as plain text

If the key material isn't preserved the peer aiming for a SCTP Restart
can only perform an INIT in plain text. Supporting this option opens
up the SCTP association to an availability attack, where an capable
attacker may be able to hijack the SCTP association. Therefore an
implementation should only support and enable this option if restart
is crucial and only when a policy is explicit set to enable the
function.

To mount the attack the attacker needs to be able to process copies of
packets sent from the target endpoint towards its peer for the
targeted SCTP association. In addition the attacker needs to be able
to send IP packets with a source address of the target's peer. If the
attacker can send an SCTP INIT that appear to be from the peer, if the
target is allowing this option it will generate an INIT ACK back, and
assuming the attacker succesfully completes the restart handshake
process the attack has managed to change the VTAG for the association
and the peer will no longer respond, leading to a SCTP associatons
failure.

As mitigation an SCTP endpoint supporting Association Restart by means
of plain text INIT SHOULD support is the following. The endpoint
receiving an INIT should send HEARTBEATs protected by DTLS CHUNK to
its peer to validate that the peer is unreachable. If the endpoint
receive an HEARTBEAT ACK within a reasonable time (at least a couple
of RTTs) the restart INIT SHOULD be discarded as the peer obviously
can respond, and thus have no need for a restart. A capable attacker
can still succeed in its attack supressing the HEARTBEAT(s) through
packet filtering, congestion overload or any other method preventing
the HEARTBEATS or there ACKs to reach their destination. If it has
been validated that the peer is unreachable, the INIT chunk will
trigger the procedure described in {{RFC9260}} section 5.2.2

Note that the Association Restart will update the verification tags
for both endpoints.  At the end of the unexpected INIT handshaking the
receiver of INIT chunk SHALL trigger the creation of a new DTLS
connection to be executed as soon as possible.  Also note that failure
in handshaking of a new DTLS connection is considered a protocol
violation and will lead to Association Abort (see {{ekeyhandshake}}).


# New Parameter Type {#new-parameter-type}

This section defines the new parameter type that will be used to
negotiate the use of the DTLS 1.3 chunk during
association setup. {{sctp-DTLS-chunk-init-parameter}} illustrates
the new parameter type.

| Parameter Type | Parameter Name |
| 0x80xx | DTLS 1.3 Chunk Protected Association |
{: #sctp-DTLS-chunk-init-parameter title="New INIT/INIT-ACK Parameter" cols="r l"}

Note that the parameter format requires the receiver to ignore the
parameter and continue processing if the parameter is not understood.
This is accomplished (as described in {{RFC9260}}, Section 3.2.1.)  by
the use of the upper bits of the parameter type.

## DTLS 1.3 Chunk Protected Association {#protectedassoc-parameter}

This parameter is only used to indicate the request and acknowledge of
support of DTLS 1.3 Chunk during INIT/INIT-ACK handshake.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Parameter Type = 0x80XX    |       Parameter Length        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Options                    |       Padding                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-DTLS-chunk-init-options title="Protected Association Parameter" artwork-align="center"}

{: vspace="0"}
Parameter Type: 16 bits (unsigned integer)
: This value MUST be set to 0x80XX.

Parameter Length: 16 bits (unsigned integer)
: This value holds the length of the Options field in
  bytes plus 4.

Options: 16 bits (unsigned integer)
: This value is set by default to zero. It contains indication of
  optional feature support.

Padding: 16 bits
: The sender MUST pad the chunk with two all zero bytes
  to make the chunk 32-bit aligned. The Padding MUST NOT be longer
  than 2 bytes and it MUST be ignored by the receiver.

RFC-Editor Note: Please replace 0x08XX with the actual parameter type
value assigned by IANA and then remove this note.

# New Chunk Types {#new-chunk-types}

##  DTLS Chunk (DTLS) {#DTLS-chunk}

This section defines the new chunk type that will be used to
transport DTLS 1.3 Records containing SCTP payload.
{{sctp-DTLS-chunk-newchunk-crypt}} illustrates the new chunk type.

| Chunk Type | Chunk Name |
| 0x4X | DTLS Chunk (DTLS) |
{: #sctp-DTLS-chunk-newchunk-crypt title="DTLS Chunk Type" cols="r l"}

RFC-Editor Note: Please replace 0x4x with the actual chunk type value
assigned by IANA and then remove this note.

It should be noted that the DTLS chunk format requires the receiver
stop processing this SCTP packet, discard the unrecognized chunk and
all further chunks, and report the unrecognized chunk in an ERROR
chunk using the 'Unrecognized Chunk Type' error cause.  This is
accomplished (as described in {{RFC9260}} Section 3.2.) by the use of
the upper bits of the chunk type.

The DTLS chunk is used to hold the DTLS 1.3 record with the protected
payload of a plain SCTP packet.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Type = 0x4X  | DCI           |         Chunk Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                            Payload                            |
|                                                               |
|                               +-------------------------------+
|                               |           Padding             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-DTLS-chunk-newchunk-crypt-struct title="DTLS Chunk Structure" artwork-align="center"}

{: vspace="0"}
Chunk Type: 8 bits (unsigned integer)
: This value MUST be set to 0x4X for all DTLS chunks.

DTLS Connection Index (DCI): 8 bits : This is used to indicate the set of Keys and other
parameters used in the protection operation to form the DTLS record
present in the Payload.

Chunk Length: 16 bits (unsigned integer)
: This value holds the length of the Payload in bytes plus 4.

Payload: variable length
: This holds the encrypted data in one or more DTLS 1.3 Records {{RFC9147}}.

Padding: 0, 8, 16, or 24 bits
: If the length of the Payload is not a multiple of 4 bytes, the sender
  MUST pad the chunk with all zero bytes to make the chunk 32-bit
  aligned.  The Padding MUST NOT be longer than 3 bytes and it MUST
  be ignored by the receiver.

##  Protection Solution Validation Chunk (PVALID) {#pvalid-chunk}

This section defines the new chunk types that will be used to validate
the Init/Init-ACK negotiation that selected the DTLS 1.3 chunk.  This
to prevent down grade attacks on the negotiation if other protection
solutions where offered. {{sctp-DTLS-chunk-newchunk-pvalid-chunk}}
illustrates the new chunk type.

| Chunk Type | Chunk Name |
| 0x4X | Protection Solution Validation (PVALID) |
{: #sctp-DTLS-chunk-newchunk-pvalid-chunk title="PVALID Chunk Type" cols="r l"}

It should be noted that the PVALID chunk format requires the receiver
stop processing this SCTP packet, discard the unrecognized chunk and
all further chunks, and report the unrecognized chunk in an ERROR
chunk using the 'Unrecognized Chunk Type' error cause.  This is
accomplished (as described in {{RFC9260}} Section 3.2.) by the use of
the upper bits of the chunk type.

The PVALID chunk is used to hold a 32-bit vector of offered protection
solutions in the INIT.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Type = 0x4X  |   Flags = 0   |         Chunk Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Protection Solutions Indicator                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-DTLS-chunk-newchunk-PVALID -struct title="PVALID Chunk Structure" artwork-align="center"}

{: vspace="0"}
Chunk Type: 8 bits (unsigned integer)
: This value MUST be set to 0x4X.

Chunk Flags: 8 bits
: MUST be set to zero on transmit and MUST be ignored on receipt.

Chunk Length: 16 bits (unsigned integer)
: This value holds the length of the Protection Engines field in bytes plus 4.

Protection Solutions Indicator: 32 bits (unsigned integer)
: This value is set by default to zero. It uses the different
  bit-values to indicate that the INIT contained an offer of the
  indiacted protection solutions. Value 0x1 is used to indiacte that
  one offered DTLS 1.3 Chunk.

RFC-Editor Note: Please replace 0x4X with the actual chunk type value
assigned by IANA and then remove this note.

# Error Handling {#error_handling}

This specification introduces a new set of error causes that are to be
used when SCTP endpoint detects a faulty condition. The special case is
when the error is detected by the DTLS 1.3 Protection that may provide
additional information.

## Mandatory Protected Association Parameter Missing {#enoprotected}

When an initiator SCTP endpoint sends an INIT chunk that doesn't
contain the DTLS 1.3 Chunk Protected Association or other protection
solutions towards an SCTP endpoint that only accepts protected
associations, the responder endpoint SHALL raise a Missing Mandatory
Parameter error. The ERROR chunk will contain the cause code 'Missing
Mandatory Parameter' (2) (see {{RFC9260}} Section 3.3.10.7) and the
protected association parameter identifier
{{protectedassoc-parameter}} in the missing param Information field.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Cause Code = 2         |         Cause Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Number of missing params = N                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Protected Association ID    |     Missing Param Type #2     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Missing Param Type #N-1    |     Missing Param Type #N     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-DTLS-init-chunk-missing-protected title="ERROR Missing Protected Association Paramater" artwork-align="center"}

Note: Cause Length is equal to the number of missing parameters 8 + N
* 2 according to {{RFC9260}}, section 3.3.10.2. Also the Protection
Association ID may be present in any of the N missing params, no order
implied by the example in
{{sctp-DTLS-init-chunk-missing-protected}}.

## Error in Protection {#eprotect}

A new Error Type is defined for DTLS Chunk, it's used for any
error related to the Protection mechanism described in this
document and has a structure that allows detailed information
to be added as extra causes.

This specification describes some of the causes whilst the
key establishment specification MAY add further causes.

When detecting an error, SCTP will send an ABORT chunk containing
the relevant Error Type and Causes.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Cause Code = TBA9         |         Cause Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Extra Cause #1        |         Extra Cause #2        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Extra Cause #N-1       |         Extra Cause #N        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-eprotect-error-structure title="Error in Protection Cause Format" artwork-align="center"}

{: vspace="0"}
Casuse Code: 16 bits (unsigned integer)
: The SCTP Error Chunk Cause Code indicating "Error in Protection" is TBA9.

Cause Length: 16 bits (unsigned integer)
: Is for N extra Causes equal to  4 + N * 2

Extra Cause: 16 bits (unsigned integer)
: Each Extra Cause indicate an additional piece of information as part
  of the error. There MAY be zero to as many as can fit in the extra
  cause field in the ERROR Chunk (A maximum of 32764).

Editor's Note: Please replace TBA9 above with what is assigned by IANA.

Below a number of defined Error Causes are defined, additional causes
can be registered with IANA following the rules in {{IANA-Extra-Cause}}.

### Error During Protection Handshake {#ekeyhandshake}

If the protection specifies a handshake for example for
authentication, and key management is implemented in-band, it may
happen that the procedure has errors. In such case an ABORT chunk will
be sent with error in protection cause code (specified in
{{eprotect}}) and extra cause "Error During Protection Handshake"
identifier 0x01.

### Failure in Protection Solution Validation {#evalidate}

A Failure may occur during protection solution validation, i.e. when
the PVALID chunks {{pvalid-chunk}} are exchanged to validate the
initialization.  In such case an ABORT chunk will be sent with error
in protection cause code (specified in {{eprotect}}) and extra cause
"Failure in Validation" identifier 0x02 to indicate this failure.

### Timeout During Protection Handshake or Validation {#etmout}

Whenever a T-valid timeout occurs, the SCTP endpoint will send an
ABORT chunk with "Error in Protection" cause (specified in
{{eprotect}}) and extra cause "Timeout During Protection Handshake or
Validation" identifier 0x03 to indicate this failure.  To indicate in
which phase the timeout occurred an additional extra cause code is
added. If the protection solution specifies that key management is
implemented in-band and the T-valid timeout occurs during the
handshake the Cause-Specific code to add is "Error During Protection
Handshake" identifier 0x01.  If the T-valid timeout occurs during the
protection association parameter validation, the extra cause code to
use is "Failure in Validation" identifier 0x02.

## Critical Error from DTLS {#eengine}

DTLS 1.3 MAY inform local SCTP endpoint about errors.  When an Error
in the DTLS 1.3 compromises the protection mechanism, the protection
operator may stop processing data altogether, thus the local SCTP
endpoint will not be able to send or receive any chunk for the
specified Association.  This will cause the Association to be closed
by legacy timer-based mechanism. Since the Association protection is
compromised no further data will be sent and the remote peer will also
experience timeout on the Association.

## Non-critical Error in the Protection {#non-critical-errors}

A non-critical error in DTLS 1.3 means that the
protection operator is capable of recovering without the need
of the whole Association to be restarted.

From SCTP perspective, a non-critical error will be perceived
as a temporary problem in the transport and will be handled
with retransmissions and SACKS according to {{RFC9260}}.

When the protection operator will experience a non-critical error,
an ABORT chunk SHALL NOT be sent.

# Procedures {#procedures}

## Establishment of a Protected Association {#establishment-procedure}

An SCTP Endpoint acting as initiator willing to create a DTLS 1.3
chunk protected association shall send to the remote peer an INIT
chunk containing the DTLS 1.3 Chunk Protected Association parameter
(see {{protectedassoc-parameter}}) whith the optional information, if
any (see {{sctp-DTLS-chunk-init-options}}).

An SCTP Endpoint acting as responder, when receiving an INIT chunk
with DTLS 1.3 Chunk Protected Association parameter, will reply with
INIT-ACK with its own DTLS 1.3 Chunk Protected Association parameter
and any optional information.

Additionally, an SCTP Endpoint acting as responder willing to support
only protected associations shall consider INIT chunk not containing
the DTLS 1.3 Chunk Protected Association parameter as an error, thus
it will reply with an ABORT chunk according to what specified in
{{enoprotected}} indicating that for this endpoint mandatory DTLS 1.3
Chunk Protected Association parameter is missing.

When initiator and responder have agreed on a protected association by
means of handshaking INIT/INIT-ACK the SCTP association establishment
continues until it has reached the ESTABLISHED state. However, before
the SCTP assocation is protected by the DTLS 1.3 Chunk some additional
states needs to be passed. First DTLS 1.3 Chunk needs be initializied
in the PROTECTION INTILIZATION state. This MAY be accomplished by the
procedure defined in {{I-D.westerlund-tsvwg-sctp-DTLS-handshake}}, or
through other methods that results in at least one DCI has
initialized state using the API. When that has been accomplished one
enters the VALIDATION state where one validates the exchange of the
DTLS 1.3 Chunk Protected Association Parameter and any alternative
protection solutions. If that is successful one enters the PROTECTED
state. This state sequence is depicted in {{init-state-machine}}.

Until the procedure has reached the PROTECTED state the only usage of
DATA Chunks that is accepted is DATA Chunks with the SCTP-DTLS PPID
used to exchange in-band key establishment messages for DTLS. Any
other DATA chunk being sent on a Protected association SHALL be
silently discarded.

DTLS 1.3 initializes itself by transferring its own handshake messages
as payload of the DATA chunk necessary
{{I-D.westerlund-tsvwg-sctp-DTLS-handshake}}. The DTLS 1.3 Chunk
initialization SHOULD be supervised by a T-valid timer that fits DTLS
1.3 handshake and may also be further adjusted based on whether
expected RTT values are outside of the ones commonly occurring on the
general Internet, see {{t-valid-considerations}}. At completion of
DTLS 1.3 Chunk initialization the setup of the Protected association is
complete and one enters the VALIDATION state, and from that time on
only DTLS 1.3 chunks will be exchanged. Any plain text chunk will be
silently discarded.

In case of T-valid timeout, the endpoint will generate an ABORT chunk.
The ERROR handling follows what specified in {{ekeyhandshake}}.

When entering the VALIDATION state, the initiator MUST send to the
responder a PVALID chunk (see
{{sctp-DTLS-chunk-newchunk-pvalid-chunk}}) containing indication of
all offered protection solutions previously sent in the INIT chunk,
including the 0x1 value indicating that DTLS 1.3 Chunk Protected
Association parameter was included. The transmission of the PVALID
chunk MUST be done reliably. The responder receiving the PVALID chunk
will compare the indicated solutions with the ones previously received
as parameters in the INIT chunk, if they are exactly the same, it will
reply to the initiator with a PVALID chunk containing the chose
proteciton solution, otherwise it will reply with an ABORT
chunk. ERROR CAUSE will indicate "Failure in Validation" and the SCTP
association will be terminated. If the association was not aborted the
protected association is considered successfully established and the
PROTECTED state is entered.

When the initiator receives the PVALID chunk, it will compare with the
previous chosen Options and in case of mismatch with the one received
previously in the protected association parameter in the INIT-ACK
chunk, it will reply with ABORT with the ERROR CAUSE "Failure in
Validation", otherwise the protected association is successfully
established and the initiator enters the PROTECTED state.

If T-valid timer expires either at initiator or responder, it will
generate an ABORT chunk.  The ERROR handling follows what specified in
{{etmout}}.

In the PROTECTED state any ULP SCTP messages for any PPID MAY be
exchanged in the protected SCTP association.

## Termination of a Protected Association {#termination-procedure}

Besides the procedures for terminating an association explained in
{{RFC9260}}, DTLS 1.3 SHALL ask SCTP endpoint for
terminating an association when having an internal error or by
detecting a security violation.
During the termination procedure all Control Chunks SHALL be protected
except SHUTDOWN-COMPLETE. The internal design of Protection
Engines and their capability is out of the scope of the current
document.

## Protection Initialization State Machine {#init-state-machine}


~~~~~~~~~~~ aasvg
     +---------------+
     |  ESTABLISHED  |
     +-------+-------+
             |
             | If INIT/INIT-ACK has Protected
             | Association Parameter
             v
+--------------------------+
| PROTECTION INITILIZATION |
+------------+-------------+
             |
             | start T-valid timer.
             |
             | [DTLS SETUP]
             |-----------------
             | send and receive
             | protection engine handshake
             v
 +----------------------+
 |      VALIDATION      |
 +-----------+----------+
             |
             | [ENDPOINT VALIDATION]
             |------------------------
             | send and receive
             | PVALID by means of
             | DTLS chunk.
             v
     +---------------+
     |   PROTECTED   |
     +---------------+
~~~~~~~~~~~
{: #sctp-DTLS-state-diagram title="DTLS Chunk State Diagram" artwork-align="center"}

## Considerations on Key Management {#key-management-considerations}

When the Association is in PROTECTION INITILIZATION state, in-band key
management MAY use SCTP user messages with the SCTP-DTLS PPID (see
{{iana-payload-protection-id}}) for message transfer that will be sent
unencrypted.

When the Association is in DTLS chunk PROTECTED state and the SCTP
assocation is in ESTABLISHED or any of the states that can be reached
after ESTABLISHED state, in-band key management are RECOMMENDED to
use SCTP user messages for message transmission that will be
protected by the DTLS 1.3 protected and encapsulated in DTLS chunks.

## Consideration on T-valid {#t-valid-considerations}

The timer T-Valid supervises initializations that depend on how the
handshake is specified for the key establishment used for the DTLS 1.3
chunk and also on the characteristics of the transport network.

This specification recommends a default value of 30 seconds for
T-valid.

# Protected Data Chunk Handling {#protected-data-handling}

With reference to the DTLS Chunk states and the state Diagram as
shown in Figure 3 of {{RFC9260}}, the handling of Control chunks, Data
chunks and DTLS chunks follows the rules defined below:

- When the association is in states CLOSED, COOKIE-WAIT, and
COOKIE-ECHOED, any Control chunk is sent unprotected (i.e. plain
text). No DATA chunks shall be sent in these states and DATA chunks
received shall be silently discarded.

- When the DTLS Chunk is in state PROTECTED and the SCTP association
is in states ESTABLISHED or in the states for association shutdown,
i.e. SHUTDOWN-PENDING, SHUTDOWN-SENT, SHUTDOWN-RECEIVED,
SHUTDOWN-ACK-SENT as defined by {{RFC9260}}, any SCTP chunk including
DATA chunks, but excluding DTLS chunk, will be used to create an SCTP
payload that will be encrypted by the DTLS 1.3 protection operation
and the resulting DTLS record from that encryption will be the used as
payload for a DTLS chunk that will be the only chunk in the SCTP
packet to be sent. DATA chunks are accepted and handled according to
section 4 of {{RFC9260}}.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Common Header                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Chunk #1                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            . . .                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Chunk #n                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-DTLS-encrypt-chunk-states-1 title="Plain Text SCTP Packet" artwork-align="center"}

The diagram shown in {{sctp-DTLS-encrypt-chunk-states-1}} describes
the structure of any plain text SCTP packet being sent or received
when the DTLS Chunk is not in VALIDATION or PROTECTED state.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Common Header                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         DTLS Chunk                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-DTLS-encrypt-chunk-states-2 title="Protected SCTP Packets" artwork-align="center"}

The diagram shown in {{sctp-DTLS-encrypt-chunk-states-2}} describes
the structure of an SCTP packet being sent after the DTLS Chunk
VALIDATION or PROTECTED state has been reached. Such packets are built
with the SCTP common header. Only one DTLS chunk can be sent in
a SCTP packet.

## Protected Data Chunk Transmission {#data-sending}

When the DTLS Chunk state machine hasn't reached the VALIDATION state,
DTLS 1.3 MAY perform key management in-band, thus the DTLS chunk
Handler will receive plain control and DATA chunks from the SCTP chunk
handler.

When DTLS Chunk has reached the VALIDATION and PROTECTED state,
the DTLS chunk handler will receive control chunks and DATA chunks
from the SCTP chunk handler as a complete SCTP payload with maximum
size limited by PMTU reduced by the size of the SCTP common header and
the DTLS chunk overhead.

That plain payload will be sent to the protection operator in use for
that specific association, the protection operator will return an
encrypted DTLS 1.3 record.

An SCTP packet containing an SCTP DTLS chunk SHALL be delivered
without delay and SCTP bundling SHALL NOT be performed.

## Protected Data Chunk Reception {#data-receiving}

When the DTLS Chunk state machine hasn't reached the VALIDATION state
it MAY perform key management in-band. In such case, the DTLS chunk
handler will receive plain control chunks and DATA chunks with
SCTP-DTLS PPID from the SCTP Header Handler. Those plain text control
chunks will be forwarded to SCTP chunk handler as well as the DATA
chunk with the SCTP-DTLS PPID.

When the DTLS Chunk state machine has reached the VALIDATION or
PROTECTED state, the DTLS chunk handler will receive DTLS chunks
from the SCTP Header Handler.  Payload from DTLS chunks will be
forwarded to the protection operator which will return a plain
SCTP Payload.  The plain SCTP payload will be forwarded to SCTP Chunk
Handler that will split it in separated chunks and will handle them
according to {{RFC9260}}.

Meta data, such as ECN, source and destination address or path ID,
belonging to the received SCTP packet SHALL be tied to the relevant
set chunks and forwarded transparently to the SCTP endpoint.

### SCTP Header Handler

The SCTP Header Handler is responsible for correctness of the SCTP
common header, it receives the SCTP packet from the lower transport
layer, discriminates among associations and forwards the payload and
relevant data to the SCTP protection engine for handling.

In the opposite direction it creates the SCTP common header and fills
it with the relevant information for the specific association and
delivers it towards the lower transport layer.

# Abstract API

This section describes and abstract API that is needed between a key
establishment part and the DTLS 1.3 protection chunk.

## Cipher Suit Capabilities

The key-management function needs to know which cipher suits defined
for usage with DTLS 1.3 that are supported by the DTLS chunk and its
protection operations block. All TLS cipher suit that are defined are
listed in the TLS cipher suit registry {{TLS-CIPHER-SUITS}} at IANA
and are identified by a 2-byte value. Thus this needs to return a list
of all supported cipher suits to the higher layer.

Request : Get Cipher Suit

Parameters : none

Reply   : Cipher Suit

Parameters : list of cipher suits

## Establish Keying Material

The DTLS Chunk can use one of out of multiple sets of cipher suit and
corresponding key materials. Which has been used are explicitly
indicated in the DCI field.

The following information needs to be provided when setting Keying material:

Request : Establish Key

Paramters :

* SCTP Assocation:
: Reference to the relevant SCTP assocation to set the keying material for.

* DCI:
: The DTLS connection index value to establish (or overwrite)

* DTLS Epoch:
: The DTLS epoch these keys are valid for. Note that Epoch lower than
  3 are note expected as they are used during DTLS handshake.

* Cipher Suit:
: 2 bytes cipher suit identification for the DTLS 1.3 Cipher suit used
  to identify the operators to perform the DTLS record protection.

* client_application_traffic_secret:

: The cipher suit specific binary object containing all necessary
information for protection operations. The secret will used by the DTLS 1.3 client to
encrypt the record. Binary arbitrary long object depending on the
cipher suit used.

* server_application_traffic_secret:'

: The cipher suit specific binary object containing all necessary
information for protection operations. The secret that will be used by
the DTLS 1.3 server to encrypt the record. Binary arbitrary long
object depending on the cipher suit used.

Reply : Established

Parameters : true or false

## Destroy Read Keying Material

A function to destroy the read (recieve) keying material for a given epoch for a given
DCI for a given SCTP Association.

Request : Destroy read key and IV

Paramters :

* SCTP Association

* DCI

* DTLS Epoch

Reply: Destroyed

Parameters : true or false

## Destroy Write Keying Material

A function to destroy the write (transmit) keying material for a given epoch for a given
DCI for a given SCTP Association.

Request : Destroy write key and IV

Paramters :

* SCTP Association

* DCI

* DTLS Epoch

Reply: Destroyed

Parameters : true or false

## Set DCI to Use

Set which key context (DCI) to use to protect the future SCTP packets sent by the
SCTP Association.

Request : Set DCI used

Paramters :

* SCTP Association

* DCI

Reply: DCI set

Parameters : true or false

## Get q

Get q, the number of protected messages (AEAD encryption invocations) for
a given epoch.

Request : Get q

Paramters :

* SCTP Association

* DCI

* DTLS Epoch

Reply: q

Parameters : non-negative integer

## Get v

Get v, the number of attacker forgery attempts
(failed AEAD decryption invocations) for a given epoch.

Request : Get v

Paramters :

* SCTP Association

* DCI

* DTLS Epoch

Reply: v

Parameters : non-negative integer


## Per Packet Information



## Configure Replay Protection

The DTLS replay protection in this usage is expected to be fairly
robust. Its depth of handling is related to maximum network path
reordering that the receiver expects to see during the SCTP
association. However as the actual reordering in number of packets are
a combination of how delayed one packet may be compared to another
times the actual packet rate this can grow for some applications and
may need to be tuned. Thus, having the potential for setting this a
more suitable value depending on the use case should be considered.

Request : Configure Replay Protection

Paramters :

* DCI

* SCTP Association

* Configuration parameters

Reply: Replay Protection Configured

Parameters : true or false


# IANA Considerations {#IANA-Consideration}

This document defines two new registries in the Stream Control
Transmission Protocol (SCTP) Parameters group that IANA
maintains. Theses registries are for the extra cause codes for
protection related errors and the Options. It also adds
registry entries into several other registries in the Stream Control
Transmission Protocol (SCTP) Parameters group:

*  Two new SCTP Chunk Types

*  One new SCTP Chunk Parameter Type

*  One new SCTP Error Cause Codes

*  One new SCTP Payload Protocol Identifier

## DTLS Options Identifier Registry {#iana-dtls-options}

IANA is requested to create a new registry called "DTLS Chunk
Options Identifiers". This registry is part of the Stream
Control Transmission Protocol (SCTP) Parameters grouping.

The purpose of this registry is to enable optional behaviors of
DTLS Chunk. Values will be assigned by IANA
a unique 16-bit unsigned integer is used.
Values 0-65534 are available for assignment. Value 65535 is
reserved for future extension. The proposed general form of the
registry is depicted below in {{iana-protection-options-identifier}}.

| ID Value | Name | Reference | Contact |
| 0-65534 | Available for Assignment | RFC-To-Be | |
| 65535 | Reserved | RFC-To-Be | Authors |
{: #iana-protection-options-identifier title="Protection Engine Identifier Registry" cols="r l l l"}

New entries are registered following the Specification Required policy
as defined by {{RFC8126}}.

## Protection Error Cause Codes Registry {#IANA-Extra-Cause}

IANA is requested to create a new registry called "Protection Error
Cause Codes". This registry is part of the Stream Control Transmission
Protocol (SCTP) Parameters grouping.

The purpose of this registry is to enable identification of different
protection related errors when using DTLS chunk and a protection
engine.  Entries in the registry requires a Meaning, a reference to
the specification defining the error, and a contact. Each entry will
be assigned by IANA a unique 16-bit unsigned integer identifier for
their protection engine. Values 0-65534 are available for
assignment. Value 65535 is reserved for future extension. The proposed
general form of the registry is depicted below in
{{iana-protection-error-cause}}.

| Cause Code | Meaning | Reference | Contact |
| 0 | Error in the Protection Engine List | RFC-To-Be | Authors |
| 1 | Error During Protection Handshake | RFC-To-Be | Authors|
| 2 | Failure in Protection Engines Validation | RFC-To-Be | Authors |
| 3 | Timeout During KEY Handshake or Validation | RFC-To-Be | Authors |
| 4-65534 | Available for Assignment | RFC-To-Be | Authors |
| 65535 | Reserved | RFC-To-Be | Authors |
{: #iana-protection-error-cause title="Protection Error Cause Code" cols="r l l l"}

New entries are registered following the Specification Required policy
as defined by {{RFC8126}}.

## SCTP Chunk Types

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Chunk Types" registry, IANA is requested to add the two new entries
depicted below in in {{iana-chunk-types}} with a reference to this
document. The registry at time of writing was available at:
https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml#sctp-parameters-1

| ID Value | Chunk Type | Reference |
| TBA6 | DTLS Chunk (DTLS) | RFC-To-Be |
| TBA7 | Protected Association Parameter Validation (PVALID) | RFC-To-Be |
{: #iana-chunk-types title="New Chunk Types Registered" cols="r l l"}


## SCTP Chunk Parameter Types

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Chunk Parameter Types" registry, IANA is requested to add the new
entry depicted below in in {{iana-chunk-parameter-types}} with a
reference to this document. The registry at time of writing was
available at:
https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml#sctp-parameters-2

| ID Value | Chunk Parameter Type | Reference |
| TBA8 | Protected Association | RFC-To-Be |
{: #iana-chunk-parameter-types title="New Chunk Type Parameters Registered" cols="r l l"}


## SCTP Error Cause Codes

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Error Cause Codes" registry, IANA is requested to add the new
entry depicted below in in {{iana-error-cause-codes}} with a
reference to this document. The registry at time of writing was
available at:
https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml#sctp-parameters-24

| ID Value | Error Cause Codes | Reference |
| TBA9 | Protection Engine Error | RFC-To-Be |
{: #iana-error-cause-codes title="Error Cause Codes Parameters Registered" cols="r l l"}


## SCTP Payload Protocol Identifier

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Payload Protocol Identifiers" registry, IANA is requested to add the new
entry depicted below in in {{iana-payload-protection-id}} with a
reference to this document. The registry at time of writing was
available at:
https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml#sctp-parameters-25

| ID Value | SCTP Payload Protocol Identifier | Reference |
| TBA10 | Protection Engine Protocol Identifier | RFC-To-Be |
{: #iana-payload-protection-id title="Protection Engine Protocol Identifier Registered" cols="r l l"}


# Security Considerations {#Security-Considerations}

All the security and privacy considerations of the security protocol
used as the protection engine applies.

## Privacy Considerations

Using a security protocol in the SCTP DTLS chunk might lower the
privacy properties of the security protocol as the SCTP Verification
Tag is an unique identifier for the association.

## Downgrade Attacks {#Downgrade-Attacks}

The pvalid chunk provides a mechanism for preventing downgrade attacks
that detects downgrading attempts between protection solutions and
terminates the association. The chosen protection solution is the same
as if the peers had been communicating in the absence of an attacker.

The initial handshake is verified before the
DTLS Chunk is considered protected, thus no user data are sent before
validation.

The downgrade protection is only as strong as the weakest of the
supported protection solutions as an active attacker can trick the
endpoints to negotiate the weakest protection solution and then
modify the weakly protected pvalid chunks to deceive the endpoints
that the negotiation of the protection engines is validated. This
is similar to the downgrade protection in TLS 1.3 specified in
Section 4.1.3. of {{RFC8446}} where downgrade protection is not
provided when TLS 1.2 with static RSA is used. It is RECOMMENDED
to only support a limited set of strongly profiled protection
solutions.

# Acknowledgments

   The authors thank Michael Tüxen for his invaluable comments
   helping to cope with Association Restart, ASCONF handling and
   restructuring the Protection Engine architecture.

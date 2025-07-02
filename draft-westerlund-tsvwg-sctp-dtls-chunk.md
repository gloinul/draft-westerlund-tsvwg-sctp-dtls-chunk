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
  I-D.ietf-tsvwg-rfc4895-bis:

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
    date: March 2025


normative:
  RFC4895:
  RFC5061:
  RFC6083:
  RFC8126:
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
   established. It is dependent on a key management function that is
   defined seperately to achieve all these capabilities. The
   key management function uses an API to provision the SCTP
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
the DTLS chunk is sent to the DTLS Protection Operator that will
perform replay protection, decrypt, verify authenticity, and if the
DTLS chunk is successfully processed return the plain text SCTP
payload containing the SCTP chunks, those chunks will then be handled
according to their SCTP protocol
specifications. {{sctp-DTLS-chunk-layering}} illustrates the DTLS
chunk layering in regard to SCTP and the Upper Layer Protocol (ULP).

~~~~~~~~~~~ aasvg
+---------------+ +--------------------+
|               | |       DTLS 1.3     |  Keys
|      ULP      | |                    +<------------.
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

In the outgoing direction, once the SCTP stack has created the
unprotected SCTP packet payload containing control and/or DATA chunks,
that payload will be sent to the DTLS Protection Operator to be
protected. The format of the protected payload is a DTLS 1.3 record
encapsulated in a SCTP chunk which is named the DTLS chunk.

The SCTP Protection Operator performs protection operations on the
whole unprotected SCTP packet payload, i.e., all chunks after the SCTP
common header. Information protection is kept during the lifetime of
the association and no information is sent unprotected except the
initial SCTP handshake, any initial key-management traffic, the SCTP
common header, the SCTP DTLS chunk header, and the SHUTDOWN-COMPLETE
chunk.

SCTP DTLS chunk capability is agreed by the peers at the
initialization of the SCTP association. Until the DTLS protection has
been keyed only plain text key-management traffic using a dedicated
PPID (4242) may flow, no ULP traffic. The key management function uses
an API to key the DTLS protection operation function. Usage of the
DTLS 1.3 handshake for initial mutual authentication and key
establishment as well as periodic re-authentication and rekeying with
Diffe-Hellman of the DTLS chunk protection is defied in a seperate
document {{I-D.westerlund-tsvwg-sctp-DTLS-handshake}}.

When the endpoint authentication and key establishment has been
completed, the association is considered to be secured and the ULP is
informed about that. From this time on it's possible for the ULPs to
exchange data securely with its peer.

A DTLS chunk will never be retransmitted, retransmission is implemented
by SCTP endpoint at chunk level as specified in {{RFC9260}}. DTLS replay
protection will be used to supress duplicated DTLS chunks, however a
failure to prevent replay will only result in duplicated SCTP chunks and
will be handled as duplicated chunks by SCTP endpoint in the same way
a duplicated SCTP packet with those SCTP chunks would have been.


## DTLS Considerations {#DTLS-engines}

The DTLS Chunk architecture splits DTLS 1.3 as shown in
{{sctp-DTLS-chunk-layering}}, where there's a Key Management functionality
on top of SCTP Chunks Handler and a Protection Operator functionality
interfacing DTLS Chunk Handler.

Key Management is the set of data and procedures that take care of key
distribution, verification, and update, DTLS connection setup, update and
maintenance.

Protection Operator functionality is the set of data and procedures
taking care of User Data encryption into DTLS Record and DTLS record
decryption into User Data.

DTLS 1.3 operations requires to directly handshake messages with the
remote peer for connection setup and other features, this kind of
handshake is part of the Key Management functionality.  Key Management
function achieves these features behaving as a user of the SCTP
association.  Key Management sends and receives its own data via the
SCTP User Level interface.  Key Management's own data are
distinguished from any other data by means of a dedicated PPID using
the value 4242 (see {{iana-payload-protection-id}}).

A Key Management using DTLS when it has established a DTLS 1.3
connection, it can derive traffic and restart keys and set the
Protection Operator for User Data encryption/decription via the API
shown in {{sctp-DTLS-chunk-layering}} to create the necessary DTLS key
contexts. Both a DTLS Key context for traffic and a DTLS Key contect
for restart should be created.

DTLS 1.3 handshake messages, that are transported as SCTP User Data
with dedicated PPID = 4242, SHALL be sent and received as plain DATA
chunks until the Association has reached the VALIDATION state
({{init-state-machine}}).  From that time on, DTLS 1.3 handshake
messages SHALL be transported as SCTP User Data with dedicated PPID =
4242 within DTLS chunks, same as any ULP data traffic.

In this document we use the terms DTLS Key context for indicating a
Key, derived from a DTLS connection, and all relevant data that needs
to be provided to the Protection Operator for DTLS encryption and
decryption.  DTLS Key context includes Keys for sending and receiving,
replay window, last used sequence number. Each DTLS key context are
associated with a four value tuple identifying the context, consisting
of SCTP Association, the restart indicator, the DTLS Connection ID (if
used), an the DTLS epoch.

Support of DTLS Connection ID in the DTLS Record layer used in the
DTLS Chunk is OPTIONAL, and negotiated using the key-management
function.

The first established key context for any SCTP association and DTLS
connection ID (if used) SHALL use epoch=3. This ensures that the
epoch of the key context will normally match the epoch of the
DTLS key-management connection.

## SCTP DTLS Chunk Buffering and Flow Control {#buffering}

DTLS 1.3 operations and SCTP are asynchronous, meaning that the
Protection Operator may deliver the decrypted SCTP Payload to the SCTP
endpoint without respecting the reception order.  It's up to SCTP
endpoint to reorder the chunks in the reception buffer and to take
care of the flow control according to what specified in
{{RFC9260}}. From SCTP perspective the DTLS chunk processing is part
of the transport network.

Even though the above allows the implementors to adopt a
multithreading design of the Protection Operators, the actual
implementation should consider that out-of-order handling of SCTP
chunks is not desired and may cause false congestion signals and
trigger retransmissions.

## PMTU Considerations {#pmtu}

The addition of the DTLS chunk to SCTP reduces the room for payload,
due to the size of the DTLS chunk header, padding, and the AEAD
authentication tag.  Thus, the SCTP layer creating the plain text
payload needs to know about the overhead to adjust its target payload
size appropriately.

A path MTU discovery function in SCTP will need to know the actual
sent and received size of packets for the SCTP packets. This to
correctly handle PMTUD probe packets.

From SCTP perspective, if there is a maximum size of plain text data
that can be protected by the Protection Operator that must be
communicated to SCTP. As such a limit will limit the PMTU for SCTP to
the maximum plain text plus DTLS chunk and algorithm overhead plus
the SCTP common header.

## Congestion Control Considerations {#congestion}

The SCTP mechanism for handling congestion control does depend on
successful data transfer for enlarging or reducing the congestion
window CWND (see {{RFC9260}} Section 7.2).

It may happen that Protection Operator discards packets due to replay
protection, or integrity errors depending on network induced bit
errors or malicous modifications. As those packets do not represent
what the peer sent, it is acceptable to ignore them, although in-situ
modification on the path of a packet resulting in discarding due to
integrity failure will leave a gap, but has to be accepted as part of
the path behavior.

The Protection Operator will not interfere with the SCTP congestion
control mechanism, this basically means that from SCTP perspective
the congestion control is exactly the same as how specified
in {{RFC9260}}.

## ICMP Considerations {#icmp}

The SCTP implementation will be responsible for handling ICMP messages
and their validation as specified in {{RFC9260}} Section 10. This
means that the ICMP validation needs to be done in relation to the
actual sent SCTP packets with the DTLS chunk and not the unprotected
payload.

## Path Selection Considerations {#multipath}

When an Association is multihomed there are multiple paths between
Endpoints.  The selection of the specific path to be used at a certain
time belongs to SCTP protocol that will decide according to
{{RFC9260}}.  The Protection Operator shall not influence the path
selection algorithm, actually the Protection Operator will not even
know what path is being used.

The Replay window for the DTLS Sequence Number will need to take into
account that heartbeat (HB) chunks are sent concurrently over all
paths in multihomed Associations, thus it needs to be large enough to
accomodate latency differencies.

## Dynamic Address Reconfiguration Considerations  {#sec-asconf}

When using Dynamic Address Reconfiguration {{RFC5061}} in an SCTP
association using DTLS Chunk the ASCONF chunk is protected, thus it
needs to be unprotected first, furthermore it MAY come from an unknown
IP Address.  In order to properly address the ASCONF chunk to the
relevant Association for being unprotected, Destination Address,
Source, Destination ports and VTag shall be used. If the combination
of those parameters is not unique the implementor MAY choose to send
the DTLS Chunk to all Associations that fit with the parameters in
order to find the right one. The association will attempt
de-protection operations on the DTLS chunk, and if that is successful
the ASCONF chunk can be processed. Note that trial decoding should
have an limit in number of tried contexts to prevent denial of service
attacks on the endpoint.

The section 4.1.1 of {{RFC5061}} specifies that ASCONF message are
required to be sent authenticated with SCTP-AUTH {{RFC4895}}.  For
SCTP associations using DTLS Chunk this results in the use of
redundant mechanism for Authentication with both SCTP-AUTH and the
DTLS Chunk. We recommend to amend {{RFC5061}} for including DTLS
Chunks as Authentication mechanism for ASCONF chunks.

## SCTP Restart Considerations  {#sec-restart}

This section deals with the handling of an unexpected INIT chunk
during an Association lifetime as described in Section 5.2 of {{RFC9260}}
with the purpose of achieving a Restart of the current Association.

The SCTP Restart procedure is defined to maintain the security
characteristics of an SCTP Association using DTLS Chunk, this requires
that SCTP Restart procedure is modified in regards to how it is
described in {{RFC9260}}.

In order to support SCTP Restart, the SCTP Endpoints shall allocate
and maintain dedicated Restart DTLS Key contexts, SCTP packets
protected by these contexts will be identified in the DTLS chunk with
the R (Restart) bit set (see {{DTLS-chunk}}).  Both SCTP Endpoints
shall ensure that Restart DTLS key contexts is preserved for
supporting the SCTP Restart use case.

In order for the protected SCTP endpoint to be available for SCTP
Restart purposes, the DTLS chunk needs acess to a DTLS Key context for
this SCTP association that needs to be kept in a well-known state so
that both SCTP Endpoints are aware of the DTLS sequence numbers and
replay window, i.e. initialized but never used. An SCTP Endpoint SHALL
NEVER use the SCTP Restart DTLS Key for any other use case than SCTP
association restart.

An SCTP endpoint that want to enable itself initiating a SCTP restart
needs to store the restart Keys, DTLS conenction ID (if used) and
related DTLS epoch, indexed so that when performing a restart with the
peer node it had an protected SCTP association with can identify the
right restart Key and DTLS epoch and initialize the restart DTLS Key
Context for when restarting the SCTP assocation. The keys, DTLS
connection ID, and epoch needs to be stored safely so that they
survive the events that are causing SCTP Restart procedure to be used,
for instance a crash of the SCTP stack.

The SCTP Restart handshakes INIT, INIT-ACK, COOCKIE-ECHO, COOKIE-ACK
exactly as in legacy SCTP Restart case; these Chunks SHALL be
sent as DTLS chunk protected using the restart DTLS key context.

A DTLS Chunk using the restart DTLS key context is identified by
having the R bit (Restart Indicator) set in the DTLS Chunk (see
{{sctp-DTLS-chunk-newchunk-crypt-struct}}).  There's exactly one
active Restart DTLS Context at a time, the newest. However, a crash at the
point having completed the key-management exchange but failing to
commit the DTLS Key Context to secure storage could result in lost of the
latest DTLS Key Context . Therefore, the endpoints SHOULD retain the old restart
DTLS key context for at least 30 seconds after having the next
installed. However, the old restart DTLS Key Context SHOULD NOT be
maintained for more than 5 minutes.


~~~~~~~~~~~ aasvg

Initiator                                     Responder
    |                                             | -.
    +------------[DTLS CHUNK(INIT)]-------------->|   |
    |<---------[DTLS CHUNK(INIT-ACK)]-------------+   +-------
    |                                             |   | Using
    |                                             |   | SCTP
    +---------[DTLS CHUNK(COOKIE ECHO)]---------->|   | Chunks
    |<--------[DTLS CHUNK(COOKIE ACK)]------------+   +-------
    |                                             | -'

~~~~~~~~~~~
{: #DTLS-chunk-restart title="Handshake of SCTP Restart for DTLS in SCTP" artwork-align="center"}

The {{DTLS-chunk-restart}} shows how the control chunks being
used for SCTP Association Restart are transported within DTLS in SCTP.

The transport of INIT, INIT-ACK COOCKIE-ECHO, COOCKIE-ACK by means of
DTLS chunk ensures that the peer requesting the restart has been
previously validated and the SCTP statemachine after having reached
ESTABLISHED state moves automatically to PROTECTED state.

A restarted SCTP Association SHALL continue to use the Restart DTLS Key Context,
for User Traffic until a new traffic DTLS Key Context will be available. The
implementors SHOULD initiate a new DTLS keying as soon as possible,
and derive the traffic and restart keys so that the time when no
Restart DTLS Key Context is available is kept to a minimum. Note that another
restart attempt prior to having created new restart DTLS Key context
for the new SCTP association will result in the endpoints being unable
to restart the SCTP assocation.

After restart the next traffic key SHALL use epoch=3, i.e. the epoch
value is reseted. Note that if the restart epoch used also was 3 when
not using any DTLS connection ID, then the installation of the new
restart key context needs to be done with some care to avoid dropping
valid packets. After having derived new traffic keys the endpoint
installs the Traffic DTLS Key Context first, and start using it. The new restart
DTLS Key Context is only installed after any old in-flight restart packets have had
a chance to be received.

# New Parameter Type {#new-parameter-type}

This section defines the new parameter type that will be used to
negotiate the use of the DTLS 1.3 chunk during association setup, its
keying method and indicate preference in relation to different keying
and other security solutions. {{sctp-DTLS-chunk-init-parameter}}
illustrates the new parameter type.

| Parameter Type | Parameter Name |
| 0x80xx | DTLS 1.3 Chunk Protected Association |
{: #sctp-DTLS-chunk-init-parameter title="New INIT/INIT-ACK Parameter" cols="r l"}

Note that the parameter format requires the receiver to ignore the
parameter and continue processing if the parameter is not understood.
This is accomplished (as described in {{RFC9260}}, Section 3.2.1.)  by
the use of the upper bits of the parameter type.

## DTLS 1.3 Chunk Protected Association {#protectedassoc-parameter}

This parameter is used to the request and acknowledge of support of
DTLS 1.3 Chunk during INIT/INIT-ACK handshake and indicate preference
for keying and the preference order between multiple security
solutions (if supported).

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Parameter Type = 0x80XX    |       Parameter Length        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Protection Solution #1       |  Protection Solution #2       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
: Protection Solutions                                          :
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Protection Solution #N        | Padding                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-DTLS-chunk-init-options title="Protected Association Parameter" artwork-align="center"}

{: vspace="0"}
Parameter Type: 16 bits (unsigned integer)
: This value MUST be set to 0x80XX.

Parameter Length: 16 bits (unsigned integer)
: This value holds the length of the parameter, which will be the
  number of Protection Solution fields (N) times two plus 4 and, if N
  is odd, plus 2 bytes of padding.

Protection Solution fields: zero or more 16-bit SCTP Protection Solution Identifers:
: Each Protection Solution Identifer ({{IANA-Protection-Solution-ID}})
  is a 16-bit unsigned integer value indicting a Protection
  Solution. Protection solutions include both DTLS Chunk based, where
  a solution combines the DTLS chunk with a key-management solution,
  or non DTLS Chunk based security solution. The Protection Solutions
  are listed in descending order of preference, i.e. the first listed
  in the parameter is the most preferred and the last the least
  preferred by the sender in the INIT chunk. In the INIT-ACK chunk the
  endpoint includes all of the offered solutions which it supports and
  lists the selected one first. Including its decreasing preference on
  the additional Protection Solutions.

Padding: If the number of included Protection solutions is odd the
parameter MUST be padded with two zero (0) bytes of padding to make
the parameter 32-bit aligned.

RFC-Editor Note: Please replace 0x08XX with the actual parameter type
value assigned by IANA and then remove this note.

# New Chunk Types {#new-chunk-types}

##  DTLS Chunk (DTLS) {#DTLS-chunk}

This section defines the new chunk type that will be used to
transport the DTLS 1.3 record containing protected SCTP payload.
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
payload of a plain text SCTP packet without the SCTP common header.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Type = 0x4x   | reserved    |R|         Chunk Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                            Payload                            |
|                                                               |
|                               +-------------------------------+
|                               |           Padding             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-DTLS-chunk-newchunk-crypt-struct title="DTLS Chunk Structure"}

reserved: 7 bits
: Reserved bits for future use. Sender MUST set these bits to 0 and
  MUST be ignored on reception.

R: 1 bit (boolean)

: Restart indicator. If this bit is set this DTLS chunk is protected
  with by an Restart DTLS Key context.

Chunk Length: 16 bits (unsigned integer)
: This value holds the length of the Payload in bytes plus 4.

Payload: variable length
: This holds the encrypted data as one DTLS 1.3 Record {{RFC9147}}.

Padding: 0, 8, 16, or 24 bits
: If the length of the Payload is not a multiple of 4 bytes, the sender
  MUST pad the chunk with all zero bytes to make the chunk 32-bit
  aligned.  The Padding MUST NOT be longer than 3 bytes and it MUST
  be ignored by the receiver.

#  Downgrade Protection Validation Message {#pvalid}

This section defines a standardized message that will be exchanged
between the endpoints to validate the Init/Init-ACK negotiation that
selected the DTLS 1.3 chunk and the select key-management solution.
This to prevent down grade attacks on the negotiation if other
protection solutions where offered. {{sctp-DTLS-PVALID-message}}
illustrates the message structure. This message is exchanged after the
SCTP assocation reached VALIDATION state on stream 0 as reliable sent
DATA message, identified using the dedicated PVALID PPID=4243,
({{sec-iana-ppid}}) to enable the key-exchange mechanism for being the
receiver of the message and thus validating the negotiation.


~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Type = 0x01  |   Flags = 0   |         Chunk Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Protection Solution #1       |  Protection Solution #2       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Protection Solutions ...                                      |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Protection Solution #N        | Padding                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-DTLS-PVALID-message -struct title="PVALID Message Structure" artwork-align="center"}

{: vspace="0"}
Type: 8 bits (unsigned integer)
: This value MUST be set to 0x01 to indicate that it is a PVALID Message

Flags: 8 bits
: MUST be set to zero on transmit and MUST be ignored on receipt.

Protection Solution fields: zero or more 16-bit SCTP Protection Solution Identifiers:

: Each Protection Solution Identifier
  ({{IANA-Protection-Solution-ID}}) is a 16-bit unsigned integer
  indicating a Protection Solution. Protection solutions include both
  DTLS Chunk based, where a solution combines the DTLS chunk with a
  key-management solution, or non DTLS Chunk based ones. The included
  Protection solutions and their order MUST match what was sent in the
  DTLS 1.3 Chunk Protected Association parameter
  {{protectedassoc-parameter}} from the sending endpoint.

Padding: If the number of included Protection solutions is odd the
parameter MUST be padded with two zero (0) bytes of padding to make
the parameter 32-bit aligned.


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
DTLS 1.3 chunk protected association parameter identifier
{{protectedassoc-parameter}} in the missing param Information
field. It may also include additional parameters representing other
supported protection mechanisms that are acceptable per endpoint
security policy.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Cause Code = 2         |         Cause Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Number of missing params = N                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  DTLS 1.3 Chunk Protected Asc |     Missing Param Type #2     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Missing Param Type #N-1    |     Missing Param Type #N     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-DTLS-init-chunk-missing-protected title="ERROR Missing Protected Association Paramater" artwork-align="center"}

Note: Cause Length in bytes is equal to following with the number of
missing parameters as N: 8 + N * 2 according to {{RFC9260}}, section
3.3.10.2. Also the Protection Association ID may be present in any of
the N missing params, no order implied by the example in
{{sctp-DTLS-init-chunk-missing-protected}}.

## Error in DTLS Chunk  {#eprotect}

A new Error Type is defined for the DTLS Chunk, it's used for any
error related to the DTLS chunk's protection mechanism described in
this document and has a structure that allows detailed information to
be added as extra causes.

This specification describes some of the causes whilst the key
establishment specification MAY add further causes.

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
{: #sctp-eprotect-error-structure title="Error in DTLS Chunk Cause Format" artwork-align="center"}

{: vspace="0"}
Cause Code: 16 bits (unsigned integer)
: The SCTP Error Chunk Cause Code indicating "Error in Protection" is TBA9.

Cause Length: 16 bits (unsigned integer)
: Is for N extra Causes equal to  4 + N * 2

Extra Cause: 16 bits (unsigned integer)
: Each Extra Cause indicate an additional piece of information as part
  of the error. There MAY be zero to as many as can fit in the extra
  cause field in the ERROR Chunk (A maximum of 32764).

Editor's Note: Please replace TBA9 above with what is assigned by IANA.

Below a number of defined Error Causes (Extra Cause above) are
defined, additional causes can be registered with IANA following the
rules in {{IANA-Extra-Cause}}.

### Error During Protection Handshake {#ekeyhandshake}

The usage of the DTLS Chunk can specify a handshake, for example
{{I-D.westerlund-tsvwg-sctp-DTLS-handshake}}, in which case that
procedure may encounter an error. In such case an ABORT chunk will be
sent with error in protection cause code (specified in {{eprotect}})
and extra cause "Error During Protection Handshake" identifier
0x01. DTLS may provide a more granular information detailing the
reason that drove the protection to fail.  Such granular information
can be added to the Error List.

### Failure in Protection Solution Validation {#evalidate}

A Failure may occur during protection solution validation, i.e. when
the PVALID messages {{sctp-DTLS-PVALID-message}} are exchanged to validate the
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

DTLS Protection Operator MAY inform local SCTP endpoint about errors.
When an Error in the DTLS 1.3 compromises the protection mechanism,
the protection operator may stop processing data altogether, thus the
local SCTP endpoint will not be able to send or receive any chunk for
the specified Association.  This will cause the SCTP Association to be
closed by legacy timer-based mechanism. Since the Association
protection is compromised no further data will be sent and the remote
peer will also experience timeout on the Association.

## Non-critical Error in the Protection {#non-critical-errors}

A non-critical error in DTLS Protection Operator means that the
Protection Operator is capable of recovering without the need
of the whole SCTP Association to be restarted.

From SCTP perspective, a non-critical error will be perceived
as a temporary problem in the transport and will be handled
with retransmissions and SACKS according to {{RFC9260}}.

When the Protection Operator will experience a non-critical error,
an ABORT chunk SHALL NOT be sent.

# Procedures {#procedures}

## Establishment of a Protected Association {#establishment-procedure}

An SCTP Endpoint acting as initiator willing to create a DTLS 1.3
chunk protected association shall send to the remote peer an INIT
chunk containing the DTLS 1.3 Chunk Protected Association parameter
(see {{protectedassoc-parameter}}) indicating supported and preferred
key-management solutions (see
{{sctp-DTLS-chunk-init-options}}).

An SCTP Endpoint acting as responder, when receiving an INIT chunk
with DTLS 1.3 Chunk Protected Association parameter, will reply with
INIT-ACK with its own DTLS 1.3 Chunk Protected Association parameter
and any its set of supported key-management solutions, and the selected
to be used among the common set listed first.

Additionally, an SCTP Endpoint acting as responder willing to support
only protected associations shall consider an INIT chunk not containing
the DTLS 1.3 Chunk Protected Association parameter or another by
security policy accepted security solution as an error, thus it will
reply with an ABORT chunk according to what specified in
{{enoprotected}} indicating that for this endpoint mandatory DTLS 1.3
Chunk Protected Association parameter is missing.

When initiator and responder have agreed on a DTLS Chunk protected
association by means of handshaking INIT/INIT-ACK the SCTP association
establishment continues until it has reached the ESTABLISHED
state. However, before the SCTP assocation is protected by the DTLS
1.3 Chunk some additional states needs to be passed. First the DTLS
Chunk needs be initializied in the PROTECTION INTILIZATION state. This
MAY be accomplished by the procedure defined in
{{I-D.westerlund-tsvwg-sctp-DTLS-handshake}}.  When that has been
accomplished one enters the VALIDATION state where one validates the
exchange of the DTLS 1.3 Chunk Protected Association Parameter and any
alternative protection solutions. If that is successful one enters the
PROTECTED state. This state sequence is depicted in
{{init-state-machine}}.

Until the procedure has reached the PROTECTED state the only usage of
DATA Chunks that is accepted is DATA Chunks with the SCTP-DTLS PPID
value 4242 used to exchange in-band key establishment messages. Any
other DATA chunk being received in a Protected association SHALL be
silently discarded.

If in-band DTLS handshake {{I-D.westerlund-tsvwg-sctp-DTLS-handshake}}
is used to establish the security parameters for the DTLS Chunks, DTLS
1.3 initializes itself by transferring its own handshake messages as
payload of the DATA chunk. The DTLS Chunk initialization SHOULD be
supervised by a T-valid timer that accomodates DTLS 1.3 handshake and
may also be further adjusted based on whether expected RTT values are
outside of the ones commonly occurring on the general Internet, see
{{t-valid-considerations}}. The Association initiator and responder
will independently enter VALIDATION state when the security parameters
are locally installed for the DTLS chunk. During VALIDATION state
both initiator and responder SHALL handle plain text chunks as well as
DTLS chunks.

In case of T-valid timeout, the endpoint will generate an ABORT chunk.
The ERROR handling follows what specified in {{ekeyhandshake}}.

When DTLS Key Context have been installed, the initiator key-management
function MUST send to the responder a PVALID message (see
{{sctp-DTLS-PVALID-message}}) containing indication of all offered
protection solutions previously sent in the INIT chunk in the DTLS 1.3
Chunk Protected Association parameter. The transmission of the PVALID
message is done using PPID=XX on stream=0 using a reliable SCTP user
message.  The responder receiving the PVALID message will compare the
indicated solutions with the ones previously received in the parameter
in the INIT chunk. The responder will ignore unknown parameters and
security solutions. For the supported solutions if the parameters in
the INIT matches what is listed in the PVALID and there are no
additional by the endpoint supported solution in the PVALID, it will
reply to the initiator with a PVALID message containing the content of
parameter sent in the INIT-ACK, otherwise it will reply with an
ABORT chunk. ERROR CAUSE will indicate "Failure in Validation" and the
SCTP association will be terminated. If the association was not
aborted the protected association is considered successfully
established and the PROTECTED state is entered.

When the initiator receives the PVALID message, it will compare with the
previous chosen option and in case of mismatch with the one received
previously in the protected association parameter in the INIT-ACK
chunk, it will reply with ABORT with the ERROR CAUSE "Failure in
Validation", otherwise the protected association is successfully
established and the initiator enters the PROTECTED state.

If T-valid timer expires either at initiator or responder, the
endpoint will generate an ABORT chunk.  The ERROR handling follows
what specified in {{etmout}}.

In the PROTECTED state any ULP SCTP messages for any PPID SHALL be
exchanged in the protected SCTP association.

When entering the PROTECTED state, a Restart DTLS Key for the current
epoch SHOULD be provided into the DTLS key context store for the SCTP
association.

### Offering Multiple Security Solutions

An initiator of an SCTP association may want to offer multiple
different key-management solutions for DTLS Chunk or in combination
with other security solutions in addition to DTLS 1.3 chunks for the
SCTP association. This can be done but need to consider the downgrade
attack risks (see {{Downgrade-Attacks}}).

The initiator MAY include in its INIT additional security solutions
that are compatible to offer in parallel with DTLS 1.3 Chunks. This
may include SCTP-AUTH {{I-D.ietf-tsvwg-rfc4895-bis}}. This will result
in that a number of different SCTP parameters may be included that are
not possible to use simultaneously. Instead the responder needs to parse
these parameters to figure out which sets of solutions that are
offered that the implementation support, and apply its security
policies to select the most approriate. For example an offer of DTLS
1.3 Chunks and SCTP-AUTH, could be interpreted as three different
solutions with different properties, namely DTLS 1.3 Chunks,
DTLS/SCTP {{RFC6083}}, and SCTP-AUTH {{I-D.ietf-tsvwg-rfc4895-bis}} only.
However, here the DTLS 1.3 Chunk Protected Assocation Parameter can
indicate both preference and which of the solutions that are desired.

The responder selects one or possibly more of compatible security
solutions that can be used simultaneously and include them in the
response (INIT-ACK). If DTLS 1.3 chunks was selected the initiator
will later send the PVALID message indicating all the offered
solutions. This to prevent downgrade attacks where sent solution have
been removed on-path.


## Termination of a Protected Association {#termination-procedure}

Besides the procedures for terminating an association explained in
{{RFC9260}}, DTLS 1.3 SHALL ask SCTP endpoint for terminating an
association when having an internal error or by detecting a security
violation.  During the termination procedure all Control Chunks SHALL
be protected except SHUTDOWN-COMPLETE. The internal design of
Protection Engines and their capability is out of the scope of the
current document.

## Protection Initialization State Machine {#init-state-machine}


~~~~~~~~~~~ aasvg
     +---------------+
     |  ESTABLISHED  |
     +-------+-------+
             |
             | If INIT/INIT-ACK has DTLS 1.3 Chunk
             | Protected Association Parameter
             v
+---------------------------+
| PROTECTION INITIALIZATION |
+------------+--------------+
             |
             | start T-valid timer.
             |
             | [DTLS SETUP]
             |-----------------
             | send and receive
             | DTLS handshake
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

When the Association is in PROTECTION INITIALIZATION state, in-band
DTLS key management, e.g. {{I-D.westerlund-tsvwg-sctp-DTLS-handshake}}, SHALL
use SCTP user messages with the SCTP-DTLS PPID value = 4242 (see
{{iana-payload-protection-id}}) for message transfer that will be sent
and received unencrypted.

When the Association is in DTLS chunk PROTECTED state and the SCTP
assocation is in ESTABLISHED or any of the states that can be reached
after ESTABLISHED state, in-band key management are RECOMMENDED to
use SCTP Data chunk with dedicated PPID value = 4242, those chunks SHALL be
sent and received using DTLS Chunks with the current DTLS Key context.

The use of plain DATA chunk with PPID value = 4242 before the
association reaches the PROTECTED state cannot be avoided as
no valid DTLS key context exist until that state.
Further in-band key management SHALL NOT use plain DATA chunks
as this would allow attackers to inject overlapping DATA chunks
with protected and impact the content of the SACK block.
Based on that, as soon as the initiator or responder
independently enter PROTECTED state, they will silently discard
any plain chunks. Plain chunks that were sent but not
received yet will also be discarded as the SCTP protocol
does guarantee the needed retransmissions.

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
  text). No DATA chunks are sent in these states and DATA chunks
  received are silently discarded, see {{RFC9260}}.

- When the DTLS Chunk is in state PROTECTED and the SCTP association
  is in states ESTABLISHED or in the states for association shutdown,
  i.e. SHUTDOWN-PENDING, SHUTDOWN-SENT, SHUTDOWN-RECEIVED,
  SHUTDOWN-ACK-SENT as defined by {{RFC9260}}, any SCTP chunk
  including DATA chunks, but excluding DTLS chunk, will be used to
  create an SCTP payload that will be encrypted by the DTLS 1.3
  protection operation and the resulting DTLS record from that
  encryption will be the used as payload for a DTLS chunk that will be
  the only chunk in the SCTP packet to be sent. DATA chunks are
  accepted and handled according to section 4 of {{RFC9260}}.

- If an SCTP restart is occurring there are exception rules to the
  above. The INIT, INIT-ACK, COOKIE-ECHO and COOKIE-ACK SHALL be
  sent protected by DTLS chunk using a Restart DTLS key context.
  The DTLS chunk with restart Key is
  continuning to protect any SCTP chunks sent while being in SCTP
  state ESTABLISHED, VALIDATION and PROTECTED, until a newely
  established traffic Key is ready to be used instead to protect
  future SCTP chunks.

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
when the DTLS Chunk is in PROTECTION INITIALIZATION, and VALIDATION
(for retransmissions).

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
the structure of a protected SCTP packet being sent after the DTLS
Chunk VALIDATION or PROTECTED state has been reached. Such packets are
built with the SCTP common header. Only one DTLS chunk can be sent in
a SCTP packet.

## Protected Data Chunk Transmission {#data-sending}

When DTLS Chunk has reached the VALIDATION and PROTECTED state,
the DTLS chunk handler will receive control chunks and DATA chunks
from the SCTP chunk handler as a complete SCTP payload with maximum
size limited by PMTU reduced by the size of the SCTP common header and
the DTLS chunk overhead.

That plain payload will be sent to the Protection Operator in use for
that specific association, the Protection Operator will return an
encrypted DTLS 1.3 record.

An SCTP packet containing an SCTP DTLS chunk SHALL be delivered
without delay, and SCTP chunk bundling {{RFC9260}} SHALL NOT be
performed.

## Protected Data Chunk Reception {#data-receiving}

When the DTLS Chunk state machine has reached the VALIDATION or
PROTECTED state, the DTLS chunk handler will receive DTLS chunks from
the SCTP Header Handler.  Payload from DTLS chunks will be forwarded
to the Protection Operator which will return a plain SCTP Payload,
assuming verified authenticty and no replay.  The plain SCTP payload
will be forwarded to SCTP Chunk Handler that will split it in
separated chunks and will handle them according to {{RFC9260}}.

If a SCTP packet with more than one DTLS chunk is received,
thus bundling multiple chunks, the receiver SHALL handle the
first DTLS chunk and ignore any subsequent chunk.

Metadata, such as ECN, reception time, IP packet size, source and
destination address or path ID, belonging to the received SCTP packet
SHALL be tied to the relevant set of chunks and forwarded
transparently to the SCTP endpoint.

### SCTP Header Handler

The SCTP Header Handler is responsible for correctness of the SCTP
common header, it receives the SCTP packet from the lower transport
layer, discriminates among associations and forwards the payload and
relevant data to the SCTP Protection Operator for handling.

In the opposite direction it creates the SCTP common header and fills
it with the relevant information for the specific association and
delivers it towards the lower transport layer.

# Abstract API  {#abstract-api}

This section describes an abstract API that is needed between a key
establishment part and the DTLS 1.3 protection chunk. This is an
example API and there are alternative implementations.

## Cipher Suit Capabilities

The key-management function needs to know which cipher suits defined
for usage with DTLS 1.3 that are supported by the DTLS chunk and its
protection operations block. All TLS cipher suit that are defined are
listed in the TLS cipher suit registry {{TLS-CIPHER-SUITS}} at IANA
and are identified by a 2-byte value. Thus this needs to return a list
of all supported cipher suits to the higher layer.

Request : Get Cipher Suits

Parameters : none

Reply   : Cipher Suits

Parameters : list of cipher suits

## Establish Client Write Keying Material

The DTLS Chunk can use one of out of multiple sets of cipher suit and
corresponding key materials.

The following information needs to be provided when setting Client Write (transmit) Keying material:

Request : Establish Client Write Key and IV

Paramters :

* SCTP Assocation:
: Reference to the relevant SCTP assocation to set the keying material for.

* Restart indication:
: A bit indicating wheter the Key is for restart purposes

* DTLS Connection ID: : If DTLS connection ID (CID) has been negotiated by
  the key-management its field length and value are include. The field length
  can be set to zero (0) to indicate that CID is not used.

* DTLS Epoch:
: The DTLS epoch these keys are valid for. Note that Epoch lower than
  3 are not expected as they are used during DTLS handshake.

* Cipher Suit:
: 2 bytes cipher suit identification for the DTLS 1.3 Cipher suit used
  to identify the DTLS AEAD algorithm to perform the DTLS record protection.
  The cipher suite is fixed for a (SCTP Assocation, Key) pair.

* Write Key and IV:
: The cipher suit specific binary object containing all necessary
information for protection operations. The secret will used by the DTLS 1.3 client to
encrypt the record. Binary arbitrary long object depending on the
cipher suit used.


Reply : Established or Failed


## Establish Server Write Keying Material

The DTLS Chunk can use one of out of multiple sets of cipher suit and
corresponding key materials.

The following information needs to be provided when setting Server Write (transmit) Keying material:

Request : Establish Server Write Key and IV

Paramters :

* SCTP Assocation:
: Reference to the relevant SCTP assocation to set the keying material for.

* Restart indication:
: A bit indicating wheter the Key is for restart purposes

* DTLS Connection ID: : If DTLS connection ID (CID) has been negotiated by
  the key-management its field length and value are include. The field length
  can be set to zero (0) to indicate that CID is not used.

* DTLS Epoch:
: The DTLS epoch these keys are valid for. Note that Epoch lower than
  3 are note expected as they are used during DTLS handshake.

* Cipher Suit:
: 2 bytes cipher suit identification for the DTLS 1.3 Cipher suit used
  to identify the DTLS AEAD algorithm to perform the DTLS record protection.
  The cipher suite is fixed for a (SCTP Assocation, Key) pair.

* Write Key and IV:
: The cipher suit specific binary object containing all necessary
information for protection operations. The secret will used by the DTLS 1.3 client to
encrypt the record. Binary arbitrary long object depending on the
cipher suit used.

Reply : Established or Failed


## Destroy Client Write Keying Material

A function to destroy the Client Write (transmit) keying material for a given epoch for a given
Key for a given SCTP Association.

Request : Destroy client write key and IV

Paramters :

* SCTP Association

* Restart indication

* DTLS CID

* DTLS Epoch

Reply: Destroyed

Parameters : true or false

## Destroy Server Write Keying Material

A function to destroy the Server Write (transmit) keying material for a given epoch for a given
Key for a given SCTP Association.

Request : Destroy server write key and IV

Paramters :

* SCTP Association

* Restart indication

* DTLS CID

* DTLS Epoch

Reply: Destroyed

Parameters : true or false

## Set Key to Use

Set which key to use to protect the future SCTP packets sent by the
SCTP Association.

Request : Set Key used

Paramters :

* SCTP Association

* Restart indication

* DTLS CID

* DTLS Epoch

Reply: Key set

Parameters : true or false

## Get q

Get q, the number of protected messages (AEAD encryption invocations) for
a given epoch.

Request : Get q

Paramters :

* SCTP Association

* Restart indication

* DTLS CID

* DTLS Epoch

Reply: q

Parameters : non-negative integer

## Get v

Get v, the number of attacker forgery attempts
(failed AEAD decryption invocations) for a given epoch.

Request : Get v

Paramters :

* SCTP Association

* Restart indication

* DTLS CID

* DTLS Epoch

Reply: v

Parameters : non-negative integer


## Configure Replay Protection

The DTLS replay protection in this usage is expected to be fairly
robust. Its depth of handling is related to maximum network path
reordering that the receiver expects to see during the SCTP
association. However as the actual reordering in number of packets are
a combination of how delayed one packet may be compared to another
times the actual packet rate this can grow for some applications and
may need to be tuned. Thus, having the potential for setting this a
more suitable value depending on the use case should be considered.

Note this sets this configuration to be used across any DTLS key
context for a given SCTP Association and traffic/restart usages.

Request : Configure Replay Protection

Paramters :

* SCTP Association

* Restart indication


* Configuration parameters

Reply: Replay Protection Configured

Parameters : true or false


# Implementation Considerations

For each DTLS Key Contexts, there are certain crypto state infomration
that needs to be handled thread safe to avoid nonce re-use and correct
replay protection.


# IANA Considerations {#IANA-Consideration}

This document defines two new registries in the Stream Control
Transmission Protocol (SCTP) Parameters group that IANA
maintains. Theses registries are for the extra cause codes for
protection related errors and the Protection Options identifiers for
the PVALID chunk. It also adds registry entries into several other
registries in the Stream Control Transmission Protocol (SCTP)
Parameters group:

*  Two new SCTP Chunk Types

*  One new SCTP Chunk Parameter Type

*  One new SCTP Error Cause Codes

And finally the update of one registred SCTP Paylod Protocol
Identifier.

## Protection Error Cause Codes Registry {#IANA-Extra-Cause}

IANA is requested to create a new registry called "Protection Error
Cause Codes". This registry is part of the Stream Control Transmission
Protocol (SCTP) Parameters grouping.

The purpose of this registry is to enable identification of different
protection related errors when using DTLS chunk and a protection
engine.  Entries in the registry requires a Meaning, a reference to
the specification defining the error, and a contact. Each entry will
be assigned by IANA a unique 16-bit unsigned integer
identifier. Values 0-65534 are available for assignment. Value 65535
is reserved for future extension. The proposed general form of the
registry is depicted below in {{iana-protection-error-cause}}.

| Cause Code | Meaning | Reference | Contact |
| 0 | Error in the Protection Operator List | RFC-To-Be | Authors |
| 1 | Error During Protection Handshake | RFC-To-Be | Authors|
| 2 | Failure in Protection Operators Validation | RFC-To-Be | Authors |
| 3 | Timeout During KEY Handshake or Validation | RFC-To-Be | Authors |
| 4-65534 | Available for Assignment | RFC-To-Be | Authors |
| 65535 | Reserved | RFC-To-Be | Authors |
{: #iana-protection-error-cause title="Protection Error Cause Code" cols="r l l l"}

New entries are registered following the Specification Required policy
as defined by {{RFC8126}}.

## SCTP Protection Solution Identifiers {#IANA-Protection-Solution-ID}

IANA is requested to create a new registry called "SCTP Protection
Solutions". This registry is part of the of the Stream
Control Transmission Protocol (SCTP) Parameters grouping.

The purpose of this registry is to assign Protection Solution
Identifier for any security solution that is either using the DTLS
Chunk combined with a key-management method, offered as an alternative
to DTLS chunk, or themselves want to use the PVALID message mechanism
to detect downgrade attacks. Any security solution that is offered
through a parameter exchange during the SCTP handshake are potential
to be included here.

Each entry will be assigned a 16-bit unsigned integer value from the suitable range.

| Identifier | Solution Name | Reference | Contact |
| 0 | DTLS 1.3 Chunk with Pre- | RFC-TBD | Draft Authors |
| 1-4095 | Available for Assignmnet using Specification Required policy | | |
| 4096-65535 | Available for Assignmnet using First Come, First Served policy | | |
{: #iana-psi title="PVALID Protection Solution Indicators" cols="r l l l"}

New entries in the range 0-4095 are registered following the Specification Required policy
as defined by {{RFC8126}}.  New entries in the range 4096-65535 are first come, first served.

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
| TBA8 | DTLS 1.3 Chunk Protected Association | RFC-To-Be |
{: #iana-chunk-parameter-types title="New Chunk Type Parameters Registered" cols="r l l"}


## SCTP Error Cause Codes

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Error Cause Codes" registry, IANA is requested to add the new
entry depicted below in in {{iana-error-cause-codes}} with a
reference to this document. The registry at time of writing was
available at:
https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml#sctp-parameters-24

| ID Value | Error Cause Codes | Reference |
| TBA9 | DTLS Chunk Error | RFC-To-Be |
{: #iana-error-cause-codes title="Error Cause Codes Parameters Registered" cols="r l l"}


## SCTP Payload Protocol Identifier {#sec-iana-ppid}

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Payload Protocol Identifiers" registry, IANA is requested to update the
entry depicted below in in {{iana-payload-protection-id}} with a
reference to this document. The registry at time of writing was
available at:
https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml#sctp-parameters-25

| ID Value | SCTP Payload Protocol Identifier | Reference |
| 4242 | DTLS Chunk Key-Management Messages | RFC-To-Be |
{: #iana-payload-protection-id title="Protection Operator Protocol Identifier Registered" cols="r l l"}

It is also requested that a new Payload Protocol Identifier is
assigned to be used to identify the PVALID message {{pvalid}}.

| ID Value | SCTP Payload Protocol Identifier | Reference |
| 4243 | DTLS Chunk PVALID Messages | RFC-To-Be |
{: #iana-payload-protection-id-pvalid title="PVALID Protocol Identifier Registered" cols="r l l"}


# Security Considerations {#Security-Considerations}

All the security and privacy considerations of the security protocol
used as the Protection Operator applies.

DTLS replay protection MUST NOT be turned off.

## Privacy Considerations

Use of the SCTP DTLS chunk provides privacy to SCTP by protecting user
data and much of the SCTP control signaling. The SCTP association is
identifiable based on the 5-tuple where the destination IP and
port are fixed for each direction. Advanced privacy features such
as changing DTLS Connection ID and sequence number encryption might
therefore have limited effect.

## AEAD Limit Considerations

Section 4.5.3 of {{RFC9147}} defines limits on the number of records
q that can be protected using the same key as well as limits on the
number of received packets v that fail authentication with each key.
To adhere to these limits the key management function can
periodically poll the DTLS protection operation function to see
when a limit have been reached or is closed to being reached.
Instead of periodic polling, a callback can be used.

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
that the negotiation of the Protection Operators is validated. This
is similar to the downgrade protection in TLS 1.3 specified in
Section 4.1.3. of {{RFC8446}} where downgrade protection is not
provided when TLS 1.2 with static RSA is used. It is RECOMMENDED
to only support a limited set of strongly profiled protection
solutions.

# Acknowledgments

   The authors thank Michael Tüxen for his invaluable comments
   helping to cope with Association Restart, ASCONF handling and
   restructuring the Protection Operator architecture. We also like
   to thank Amanda Baber with IANA for feedback on our IANA registry.

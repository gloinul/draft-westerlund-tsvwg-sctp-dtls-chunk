---
docname: draft-ietf-tsvwg-sctp-dtls-chunk-latest
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
-
   ins: M. Tüxen
   name: Michael Tüxen
   org: Münster University of Applied Sciences
   abbrev: Münster Univ. of Appl. Sciences
   street: Stegerwaldstrasse 39
   code: 48565
   city: Steinfurt
   country: Germany
   email: tuexen@fh-muenster.de

informative:
  RFC6458:
  RFC8446:
  I-D.ietf-tsvwg-rfc4895-bis:
  I-D.ietf-tsvwg-dtls-chunk-key-management:
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
    date: Jul 2025

  ETSI-TS-38.413:
    target: "https://www.etsi.org/deliver/etsi_ts/138400_138499/138413/18.05.00_60/ts_138413v180500p.pdf"
    title: "NG Application Protocol (NGAP) version 18.5.0 Release 18"
  date: March 2025

normative:
  RFC4820:
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

updates:
  RFC5061:

--- abstract

This document describes a method for adding DTLS Authentication
and Cryptographic protection to the Stream Control Transmission
Protocol (SCTP).

The SCTP DTLS chunk defined in this document is intended to enable
communications privacy for applications that use SCTP as their
transport protocol and allows applications to communicate in a
way that is designed to prevent eavesdropping and detect tampering
or message forgery.

Applications using SCTP DTLS chunk can use all transport
features provided by SCTP and its extensions but with some limitations,
and in the case of Dynamic Address Reconfiguration RFC 5061 requires
updates.

--- middle

# Introduction {#introduction}

   This document defines a DTLS chunk for the Stream Control
   Transmission Protocol (SCTP), as defined in {{RFC9260}}.

   This specification defines the actual DTLS chunk, how to enable
   its usage, how it interacts with the SCTP association establishment
   to enable endpoint authentication, key-establishment, and key
   updates.

   The DTLS chunk is designed to enable mutual DTLS based
   authentication of endpoints, data confidentiality, DTLS based
   data origin authentication, data integrity protection, and data replay
   protection for SCTP packets after the SCTP association has been
   established. It is dependent on a DTLS Key Management Method that is
   defined separately to achieve all these capabilities. The
   DTLS Key Management Method uses an API to provision the SCTP
   association's DTLS chunk protection with key-material to enable and
   rekey the protection operations.

   Applications using SCTP DTLS chunk can use most transport features
   provided by SCTP and its extensions. However, there can be some
   limitations or additional requirements for them to function such as
   those noted for SCTP restart {{sec-restart}} and an actual update
   of the specification of Dynamic Address Reconfiguration {{RFC5061}},
   see {{sec-asconf}}. Due to DTLS chunk's
   level of integration as discussed in next section it will provide
   its security functions on all content of the SCTP packet, and will
   thus not impact the potential to utilize any SCTP functionalities
   or extensions that are possible to use between two SCTP peers with
   full security and SCTP association state.

   DTLS is considered version 1.3 as specified in {{RFC9147}} whereas
   other versions are explicitly not part of this document.


# Conventions

{::boilerplate bcp14}

# Protocol Overview {#protocol-overview}

The DTLS chunk can be used for secure and confidential transfer of
SCTP packets. This is implemented inside the SCTP protocol.
Once an SCTP packet has been received and the SCTP common header has
been used to identify the SCTP association, the DTLS chunk is processed by
the Chunk Protection Operator that will perform replay protection, decrypt,
verify authenticity, and if the DTLS chunk is successfully processed provides
the protected SCTP chunks for further processing.
{{sctp-DTLS-chunk-layering}} is an example
illustrating the DTLS chunk processing in regard
to SCTP and the Upper Layer Protocol (ULP) using
DTLS 1.3 as the DTLS Key Management Method. Here the DTLS Key Management Method
contains validation, i.e. using certificates, handshaking,
updating policies etc.

~~~~~~~~~~~ aasvg
+---------------+ +-------------------------------+
|      ULP      | |            DTLS 1.3           |
|               | |    +---------------------+    |
|               | | +->+    Key Exporter     +--+ |
|               | | |  +---------------------+  | |
|               | | |                           | |
|               | | |  +---------------------+  | |
|               | | +--+    Key Management   +  | |
|               | | |  +----------+----------+  | |
|               | | |             |             | |
|               | | | ContentType |             | |
|               | | |  +----------+----------+  | |
|               | | +->|        Record       |  | |
|               | |    | Protection Operator |  | |
+               | |    +----------+----------+  | |
+-------+-------+ +-----------------------------+-+
        ^                          ^            |
        |                          |            |
        +--+-----------------------+            | keys
      PPID |                                    |
           V                                    V
+-----------------------------------------------+-+
|                    +---------------------+    | |
|        SCTP        |         Chunk       |<---+ |
|                    | Protection Operator |      |
|                    +---------------------+      |
+-------------------------------------------------+
~~~~~~~~~~~
{: #sctp-DTLS-chunk-layering title="DTLS Chunk Layering
in Regard to SCTP and ULP" artwork-align="center"}

In the outgoing direction, once the SCTP stack has created the
unprotected SCTP packet containing control and/or DATA chunks,
SCTP chunks will be processed by the Chunk Protection Operator to be
protected. The result of this computation is a DTLS 1.3 record
encapsulated in a SCTP chunk which is named the DTLS chunk.

The Chunk Protection Operator performs protection operations on all
chunks of an SCTP packet. Information protection is kept during the lifetime of
the association and no information is sent unprotected except the
initial SCTP handshake, any initial DTLS Key Management traffic, the SCTP
common header, the SCTP DTLS chunk header, and the INIT and INIT-ACK
chunks during an SCTP Restart procedure.

The support of the DTLS chunk and the DTLS Key Management Method to use is
negotiated by the peers at the setup of the SCTP association using a
new parameter. The DTLS Key Management and application traffic is multiplexed
using the PPID. The dedicated PPID 4242 is defined for use by all DTLS Key
Management Methods. The DTLS Key Management Method uses an API to
key the Chunk protection operation function. Usage of the DTLS 1.3
handshake for initial mutual authentication and key establishment as
well as periodic re-authentication and rekeying with Diffe-Hellman of
the DTLS chunk protection is defined in separate documents,
(see {{dtls-management-method}}). To prevent
downgrade attacks affecting the DTLS Key Management negotiation
the DTLS Key Management Method should implement specific procedures when
deriving keys.

When the endpoint authentication and key establishment has been
completed, the association is considered to be secured and the ULP is
informed about that. From this time on it's possible for the ULPs to
exchange data securely with its peer.

A DTLS chunk will never be retransmitted, retransmission is implemented
by SCTP endpoint at chunk level as specified in {{RFC9260}}. DTLS replay
protection will be used to suppress duplicated DTLS chunks.

# Protocol Considerations

## DTLS Considerations {#DTLS-engines}

The DTLS Chunk architecture splits DTLS 1.3 as shown in
{{sctp-DTLS-chunk-layering}}, where the DTLS Key Management Method
is done at DTLS 1.3 block level, acting as a parallel User Level Protocol
and a Chunk Protection Operator functionality inside the SCTP
Protocol Stack.

DTLS Key Management Method is the set of data and procedures that take care of key
distribution, verification, and update, DTLS connection setup, update and
maintenance.

Chunk Protection Operator functionality is the set of data and procedures
taking care of User Data encryption into DTLS Record and DTLS record
decryption into User Data.

DTLS 1.3 operations requires to directly handshake messages with the
remote peer for connection setup and other features, this kind of
handshake is part of the DTLS Key Management Method.  DTLS Key Management Method
achieves these features behaving as a user of the SCTP
association.  DTLS Key Management Method sends and receives its own data via the
SCTP User Level interface.  DTLS Key Management Method's own data are
distinguished from any other data by means of a dedicated PPID using
the value 4242 (see {{iana-payload-protection-id}}).

Once DTLS Key Management Method has established a DTLS 1.3 connection, it can
derive primary and restart keys and set the Chunk Protection Operator
for SCTP Packet Payload encryption/decryption via an API to create the
necessary DTLS key contexts. Both a DTLS Key context for normal use
(primary) and a DTLS Key context for SCTP association restart needs to
be created.

In this document we use the terms DTLS Key context for indicating a
Key and IV, produced by the DTLS Key Management, and all relevant data that
needs to be provided to the Chunk Protection Operator for DTLS encryption
and decryption.  DTLS Key context includes Keys and IV for sending and
receiving, replay window, last used sequence number. Each DTLS key
context is associated with a three-value tuple identifying the context,
consisting of SCTP Association, the restart indicator, and the DTLS epoch.

The DTLS Connection ID in the DTLS Record layer used in the
DTLS Chunk MUST NOT be used.

The first established DTLS key context for any SCTP association MUST use epoch=3.
This ensures that the epoch of the DTLS key context will normally match the epoch of
a DTLS Management Method´s connection.

The Replay window for the DTLS Sequence Number will need to take into
account that heartbeat (HB) chunks are sent concurrently over all
paths in multihomed Associations, thus it needs to be large enough to
accommodate latency differences.

Endpoints implementing DTLS Chunk MUST support DTLS records containing up to
2<sup>14</sup> (16384) bytes of plain text.

## Considerations about SCTP DTLS Key Management Methods {#dtls-management-method}

This document specifies the mechanisms for SCTP to be protected with
DTLS, it doesn't specify how the DTLS Key Management works, being limited
on what the DTLS Key Management MUST provide for achieving the protection.
Even though DTLS1.3 is indicated as protocol for providing Key
Contexts, different implementations can achieve that and different
mechanisms may be used for features such as mutual authentication,
rekeying etc.  The DTLS Key Management Method may use a number of DTLS Key
Management methods depending on what is being implemented and
available and/or according to the local policies.
DTLS Key Management Methods are defined in
their own specific documents, and needs to be registered in the IANA
Registry "SCTP DTLS Key Management Methods" to get their own unique identifier.
This document constitutes a requirement towards any DTLS Key Management Method.

Currently there are two in-band DTLS Key Management Methods defined,
they have different properties. See
{{I-D.ietf-tsvwg-dtls-chunk-key-management}} and
{{I-D.westerlund-tsvwg-sctp-DTLS-handshake}}.

## Dynamic Address Reconfiguration Considerations  {#sec-asconf}

{{RFC5061}} specifies the support for Dynamic Address Reconfiguration
in SCTP. DTLS Chunks has limited support for Dynamic Address
Reconfiguration and requires an update of {{RFC5061}}. Section 4.1.1
of {{RFC5061}} specifies that ASCONF message are
required to be sent authenticated with SCTP-AUTH {{RFC4895}}.  For
SCTP associations using DTLS Chunk this would result in the use of
redundant and non-compatible mechanisms for Authentication with
both SCTP-AUTH and the DTLS Chunk.

### Limitations

Support for Dynamic Address Reconfiguration {{RFC5061}} in an SCTP
association using DTLS Chunk is limited to the case where the packet
containing the ASCONF chunk is sent from an IP address already known.

### Change

Because SCTP-AUTH and DTLS chunks provide non-compatible authentication
mechanisms, SCTP-AUTH {{RFC4895}} MUST NOT be used once DTLS chunks have been
successfully negotiated.

## SCTP Restart Considerations  {#sec-restart}

This section deals with the handling of an unexpected INIT chunk
during an Association lifetime as described in Section 5.2 of {{RFC9260}}
with the purpose of achieving a Restart of the current Association,
thus implementing SCTP Restart.

This specification doesn't support SCTP Restart as described in
{{RFC9260}} because the COOCKIE-ECHO and COOKIE-ACK chunks
are sent encrypted (see {{protected-restart}});

When the upper layer protocols require support of SCTP Restart, as in
case of 3GPP NG-C protocol {{ETSI-TS-38.413}}, the endpoint needs to
support also initiating protected SCTP Restart procedure described in
{{protected-restart}}. Implementing initiating protected restart
procedure is RECOMMENDED, however not required as persistent secure
storage of the restart DTLS Key Context is needed.

The cases where one of the SCTP Endpoints only implements initiating
legacy SCTP Restart are described in {{sctp-rest-comp}}.


### Protected SCTP Restart {#protected-restart}

The protected SCTP Restart procedure keeps the security
characteristics of an SCTP Association using DTLS Chunk.

In protected SCTP Restart, INIT and INIT-ACK chunks are sent
strictly according to  {{RFC9260}}, but COOCKIE-ECHO and COOKIE-ACK chunks
are encrypted using DTLS Chunks and Restart DTLS Key contexts.

In order to support protected SCTP Restart, the SCTP Endpoints needs
to allocate and maintain dedicated Restart DTLS Key contexts, SCTP
packets protected by these contexts will be identified in the DTLS
chunk with the R (Restart) bit set (see {{DTLS-chunk}}).  Both SCTP
Endpoints needs to ensure that Restart DTLS key contexts is preserved
for supporting the protected SCTP Restart use case.

In order for the protected SCTP endpoint to be available for protected SCTP
Restart purposes, the DTLS chunk needs access to a DTLS Key context for
this SCTP association that needs to be kept in a well-known state so
that both SCTP Endpoints are aware of the DTLS sequence numbers and
replay window, i.e. initialized but never used. An SCTP Endpoint MUST
NEVER use the SCTP Restart DTLS Key for any other use case than SCTP
association restart.

An SCTP endpoint wanting to be able to initiate a protected SCTP
restart needs to store securely and persistently the restart Keys,
and related DTLS epoch, indexed so that when
performing a restart with the peer node it had a protected SCTP
association which can identify the right restart Key and DTLS epoch and
initialize the restart DTLS Key Context for when restarting the SCTP
association. The keys and epoch needs to be stored
secure and persistently so that they survive the events that are
causing protected SCTP Restart procedure to be used, for instance a
crash of the SCTP stack. The security considerations for persistent
secure storage of keying materials is further discussed in
{{sec-considertation-storage}}.

The SCTP Restart handshakes INIT, INIT-ACK, COOCKIE-ECHO, COOKIE-ACK
exactly as in legacy SCTP Restart case; INIT, INIT-ACK MUST be
sent plain as in the legacy, whereas COOCKIE-ECHO, COOKIE-ACK
Chunks MUST be sent as DTLS chunk protected using the restart DTLS key context.

A DTLS Chunk using the restart DTLS key context is identified by
having the R bit (Restart Indicator) set in the DTLS Chunk (see
{{sctp-DTLS-chunk-newchunk-crypt-struct}}).  There's exactly one
active Restart DTLS Context at a time, the newest. However, a crash at
the time having completed the DTLS Key Management exchange but failing to
commit the DTLS Key Context to persistent secure storage could result
in loss of the latest DTLS Key Context. Therefore, the endpoints
SHOULD retain the old restart DTLS key context until it the
DTLS Key Management confirms the new ones are committed to secure storage.
This can for example be ensure that at key-changes signals to
terminate the old DTLS Key Contexts (including the restart) is never
sent until the new restart DTLS Key Context has been committed to
storage.


~~~~~~~~~~~ aasvg

Initiator                                     Responder
    |                                             | -.
    |                                             |   +-------
    +--------------------(INIT)------------------>|   | Plain
    |<-----------------(INIT-ACK)-----------------+   +-------
    |                                             | -'
    |                                             | -.
    |                                             |   +-------
    +---------[DTLS CHUNK(COOKIE ECHO)]---------->|   | Protected
    |<--------[DTLS CHUNK(COOKIE ACK)]------------+   +-------
    |                                             | -'
    |                                             |

~~~~~~~~~~~
{: #DTLS-chunk-restart title="Handshake of SCTP Restart for DTLS in SCTP" artwork-align="center"}

The {{DTLS-chunk-restart}} shows how the control chunks being
used for SCTP Association Restart are transported within DTLS in SCTP.

The transport of COOCKIE-ECHO, COOCKIE-ACK by means of
DTLS chunk ensures that the peer requesting the restart has been
previously validated and the SCTP state machine after having reached
ESTABLISHED state moves automatically to PROTECTED state.

A restarted SCTP Association MUST continue to use the Restart DTLS Key Context,
for User Traffic until a new primary DTLS Key Context will be available. The
implementors SHOULD initiate a rekeying as soon as possible,
and derive the primary and restart keys so that the time when no
Restart DTLS Key Context is available is kept to a minimum. Note that another
restart attempt prior to having created new restart DTLS Key context
for the new SCTP association will result in the endpoints being unable
to restart the SCTP association.

After restart the next primary DTLS key context MUST use epoch=3,
i.e. the epoch value is reset. After having derived new
primary DTLS Key Context the endpoint installs the primary DTLS Key Context first,
and start using it. The new restart DTLS Key Context is only installed
after any old in-flight restart packets will have been received.

### Compatibility with Legacy SCTP Restart {#sctp-rest-comp}

An SCTP Endpoint supporting only legacy SCTP Restart and involved in
an SCTP Association using DTLS Chunks SHOULD NOT attempt to restart
the Association. The effect will be that the restart initiator will
receive INIT-ACK but then all sent packets with COOCKE-ECHO will be
dropped until the peer nodes times out the SCTP Association from lack
of any response from the restarting node.

An SCTP Endpoint supporting only legacy SCTP Restart and involved
in an SCTP Association using DTLS Chunks, when receiving an COOCKIE-ECHO
chunk protected by DTLS chunk as described in {{protected-restart}},
thus having the R bit (Restart Indicator) set in the DTLS Chunk (see
{{sctp-DTLS-chunk-newchunk-crypt-struct}}), will silently discard it.

Since an SCTP Endpoint supporting only legacy SCTP Restart and involved
in an SCTP Association using DTLS Chunks cannot use SCTP Restart
legacy procedure, in case of need to restart the Association
it SHOULD keep on retrying initiating a new Association
until the remote SCTP Endpoint have closed the existing Association
(i.e. due to timeout) and will accept a new one.
As alternative, depending on the Use Case and the Upper Layer protocol,
it MAY use a different SCTP Source port number so that the peer SCTP Endpoint
will accept the initiation of the new Association while still supervising
the old one.

# New Parameter Type {#new-parameter-type}

This section defines the new parameter type that will be used to
negotiate the use of the DTLS chunk during association setup, its
DTLS Key Management Method and indicate preference in relation to different
DTLS Key Management Methods. {{sctp-DTLS-chunk-init-parameter}}
illustrates the new parameter type.

| Parameter Type | Parameter Name |
| 0x8006 | DTLS Key Management Parameter |
{: #sctp-DTLS-chunk-init-parameter title="New INIT/INIT-ACK Parameter" cols="r l"}

Note that the parameter format requires the receiver to ignore the
parameter and continue processing if the parameter is not understood.
This is accomplished (as described in {{RFC9260}}, Section 3.2.1.)  by
the use of the upper bits of the parameter type.

## DTLS Key Management Parameter {#protectedassoc-parameter}

This parameter is used to the request and acknowledge of support of
DTLS Chunk during INIT/INIT-ACK handshake and indicate preference
order among DTLS Key Management Methods (if supported).

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Parameter Type = 0x8006    |       Parameter Length        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  DTLS Key Management Id #1    |  DTLS Key Management Id #2    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
:                                                               :
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| DTLS Key Management Id #N     | Padding                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-DTLS-chunk-init-options title="DTLS Key Management Parameter" artwork-align="center"}

{: vspace="0"}
Parameter Type: 16 bits (unsigned integer)
: This value MUST be set to 0x8006.

Parameter Length: 16 bits (unsigned integer)
: This value holds the length of the parameter, which will be 2 times the
  number of DTLS Key Management identifiers  (N) plus 4.

DTLS Key Management Identifier: 16 bits (unsigned integer)
: Each DTLS Key Management Identifier ({{IANA-Protection-Solution-ID}})
  is a 16-bit unsigned integer value indicating a DTLS Key Management Method.
  The DTLS Management Methods are listed in descending order of preference, i.e. the first listed
  in the parameter is the most preferred and the last the least
  preferred by the sender in the INIT chunk. In the INIT-ACK chunk the
  endpoint chooses one of the DTLS Management Methods supported by the peer.

Padding: 0 or 16 bits (unsigned integer)
: If the number of included DTLS Management Methods is odd the
parameter MUST be padded with two bytes. The padding MUST be set to 0 by
the sender and MUST be ignored by the receiver.

# New Chunk Type {#new-chunk-type}

##  DTLS Chunk (DTLS) {#DTLS-chunk}

This section defines the new chunk type that will be used to
transport the DTLS 1.3 record containing protected SCTP payload.
{{sctp-DTLS-chunk-newchunk-crypt}} illustrates the new chunk type.

| Chunk Type | Chunk Name |
| 0x41 | DTLS Chunk (DTLS) |
{: #sctp-DTLS-chunk-newchunk-crypt title="DTLS Chunk Type" cols="r l"}

It should be noted that the DTLS chunk format requires the receiver
stop processing this SCTP packet, discard the unrecognized chunk and
all further chunks, and report the unrecognized chunk in an ERROR
chunk using the 'Unrecognized Chunk Type' error cause.  This is
accomplished (as described in {{RFC9260}} Section 3.2.) by the use of
the upper bits of the chunk type.

The DTLS chunk is used to hold the DTLS 1.3 record with the protected
payload of a plain text SCTP packet without the SCTP common header.

As the full DTLS record with the header and sequence number, etc the
start of the cipher text is likely not 32-bit aligned making in-place
encryption/decryption impossible in some plattforms. Therefore, a
variable number (0-3) of pre-padding bytes with value fixed to zero MUST
be added in the DTLS chunk payload before the DTLS Record header
(DTLS Chunk Payload), to ensure the Encrypted Record part of the
DTLSCiphertext {{RFC9147}} will start on a 32-bit boundary in relation
to the start of the DTLS Chunk. The number of these pre-padding bytes
is indicated in the DTLS Chunk header using the P bits.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| Type = 0x41   | reserved| P |R|         Chunk Length          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Pre-Padding            |                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+                               |
|                                                               |
|                            Payload                            |
|                                                               |
|                               +-------------------------------+
|                               |       Post-Padding            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-DTLS-chunk-newchunk-crypt-struct title="DTLS Chunk Structure" artwork-align="center"}

reserved: 5 bits
: Reserved bits for future use. Sender MUST set these bits to 0 and
  MUST be ignored on reception.

R: 1 bit (boolean)

: Restart indicator. If this bit is set this DTLS chunk is protected
  with by a Restart DTLS Key context.

P: 2 bit (unsigned integer 0-3)

: Payload Pre-Padding indicator. It indicates how many bytes
are inserted for padding before the DTLSCiphertext.
This allows the encrypted data to be 32 bit aligned.

Chunk Length: 16 bits (unsigned integer)
: This value holds the length of the Payload in bytes plus 4.

Pre-Padding: 0, 8, 16, or 24 bits
: Based on the Payload Pre-Padding Indicator the indicated number of
8-bit bytes of zero values are included.

Payload: variable length
: This holds the DTLSCiphertext as specified in DTLS 1.3 {{RFC9147}}.

Post-Padding: 0, 8, 16, or 24 bits
: If the length of the Payload is not a multiple of 4 bytes, the sender
  MUST pad the chunk with all zero bytes to make the chunk 32-bit
  aligned.  The Padding MUST NOT be longer than 3 bytes and it MUST
  be ignored by the receiver.

## Payload formatting in DTLS Chunk


From section 4 of {{RFC9147}}, the DTLS record header has variable length,
here reported in {{DTLSCiphertext-record-struct}}.

~~~~~~~~~~~ aasvg

    struct {
        opaque unified_hdr[variable];
        opaque encrypted_record[length];
    } DTLSCiphertext;

~~~~~~~~~~~
{: #DTLSCiphertext-record-struct title="DTLS DTLSCiphertext" artwork-align="center"}

As shown above, DTLSCiphertext record is built up with the unified_hdr
and the encrypted_record, where unified_hdr has variable format
as defined in the first byte. The format of unified_hdr is depicted
in {{DTLSCiphertext-header-struct}}.

~~~~~~~~~~~ aasvg

    0 1 2 3 4 5 6 7
    +-+-+-+-+-+-+-+-+
    |0|0|1|C|S|L|E E|
    +-+-+-+-+-+-+-+-+
    | Connection ID |   Legend:
    | (if any,      |
    /  length as    /   C   - Connection ID (CID) present
    |  negotiated)  |   S   - Sequence number length
    +-+-+-+-+-+-+-+-+   L   - Length present
    |  8 or 16 bit  |   E   - Epoch
    |Sequence Number|
    +-+-+-+-+-+-+-+-+
    | 16 bit Length |
    | (if present)  |
    +-+-+-+-+-+-+-+-+

~~~~~~~~~~~
{: #DTLSCiphertext-header-struct title="DTLSCiphertext header" artwork-align="center"}

DTLS Chunk requires encrypted_record to be 32 bit aligned as specified
in {{DTLS-chunk}}.  The size of the header of the DTLSCiphertext can
be easily computed by reading the first octet. The Length
field is redundant with the DTLS chunk's length field and can be
avoided to be used, and multiple DTLS records SHALL NOT be part of the
DTLS Chunk's payload field.  Examples of preferred DTLSCiphertext are
shown in {{DTLSCiphertext-recommended}}.

~~~~~~~~~~~ aasvg

 0 1 2 3 4 5 6 7       0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+     +-+-+-+-+-+-+-+-+
|0|0|1|0|1|1|E E|     |0|0|1|0|0|0|E E|
+-+-+-+-+-+-+-+-+     +-+-+-+-+-+-+-+-+
|    16 bit     |     |8 bit Seq. No. |
|Sequence Number|     +-+-+-+-+-+-+-+-+
+-+-+-+-+-+-+-+-+     |               |
|   16 bit      |     |   Encrypted   |
|   Length      |     /   Record      /
+-+-+-+-+-+-+-+-+     |               |
|               |     +-+-+-+-+-+-+-+-+
|  Encrypted    |
/  Record       /       DTLSCiphertext
|               |         Structure
+-+-+-+-+-+-+-+-+         (minimal)

  DTLSCiphertext
    Structure
  (recommended)

~~~~~~~~~~~
{: #DTLSCiphertext-recommended title="DTLSCiphertext recommended structure" artwork-align="center"}

Thus the size of the DTLSCiphertext header, using the first octet B, is computed as follows:

size = 1 + (B & 0x08) ? 2 : 1 + (B & 0x04) ? 2 : 0

In order the encrypted_record to be 32 bit aligned, P bit in the DTLS Chunk header are computed
as follows:

P = (4 - (size & 0x03)) & 0x03

# Error Handling {#error_handling}

This specification introduces a new set of error causes that are to be
used when SCTP endpoint detects a faulty condition. The special case is
when the error is detected by the DTLS Key Management Method that may provide
additional information.

## DTLS Chunk Protected Association Parameter Missing {#enoprotected}

When an initiator SCTP endpoint sends an INIT chunk that doesn't
contain the DTLS Key Management Parameter or a supported DTLS Key Management Method
towards an SCTP endpoint that only accepts protected
associations, SCTP will send an ABORT
chunk in response to the INIT chunk (Section 5.1 of {{RFC9260}}
including the error cause 'Policy Not Met' (TBA10)
(see {{IANA-Extra-Cause}} and the DTLS Key Management Method
identifiers {{protectedassoc-parameter}} in the missing
param Information field.
It may also include additional parameters representing other
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
|    DTLS Key Management Parameter  |     Missing Param Type #2     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Missing Param Type #N-1    |     Missing Param Type #N     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-DTLS-init-chunk-missing-protected title="ERROR Missing Protected Association Parameter" artwork-align="center"}

Note: Cause Length in bytes is equal to following with the number of
missing parameters as N: 8 + N * 2 according to {{RFC9260}}, section
3.3.10.2. Also the Protection Association ID may be present in any of
the N missing params, no order implied by the example in
{{sctp-DTLS-init-chunk-missing-protected}}.

## No Common DTLS Key Management Method {#enocommonpsi}

If the responder to do not support any of the DTLS Management Methods
offered by the association initiator in the Protection Soluiton
Parameters {{sctp-DTLS-chunk-init-options}} SCTP will send an ABORT
chunk in response to the INIT chunk (Section 5.1 of {{RFC9260}},
including the error cause "No Common DTLS Key Management" (TBA11)
(see {{IANA-Extra-Cause}}).


## Critical Error from DTLS {#eengine}

The Chunk Protection Operator MAY inform local SCTP endpoint about errors.
When an Error in the DTLS 1.3 compromises the protection mechanism,
the Chunk Protection Operator may stop processing data altogether, thus the
local SCTP endpoint will not be able to send or receive any chunk for
the specified Association.  This will cause the SCTP Association to be
closed by legacy timer-based mechanism. Since the Association
protection is compromised no further data will be sent and the remote
peer will also experience timeout on the Association.

## Non-critical Error in the Protection {#non-critical-errors}

A non-critical error in Chunk Protection Operator means that the
Chunk Protection Operator is capable of recovering without the need
of the whole SCTP Association to be re-established.

From SCTP perspective, a non-critical error will be perceived
as a temporary problem in the transport and will be handled
with retransmissions and SACKS according to {{RFC9260}}.

When the Chunk Protection Operator will experience a non-critical error,
an ABORT chunk MUST NOT be sent.

# Procedures {#procedures}

## Establishment of a Protected Association {#establishment-procedure}

An SCTP Endpoint acting as initiator willing to create a DTLS 1.3
chunk protected association sends to the remote peer an INIT
chunk containing the DTLS 1.3 Chunk Protected Association parameter
(see {{protectedassoc-parameter}}) indicating supported and preferred
DTLS Key Management method (see {{sctp-DTLS-chunk-init-options}}).

An SCTP Endpoint acting as responder, when receiving an INIT chunk
with a DTLS Key Management Parameter, will reply with
INIT-ACK with its own DTLS MKey anagement Parameter
containing the selected DTLS Key Management Method out of the set of supported
ones. In case there are no common set of supported DTLS Key Management Methods that are
accepted by the responder, and the endpoints' policy require secured
association it MUST reply with an ABORT chunk, include the error
cause "No DTLS Key Management Method" (TBA11) (see {{IANA-Extra-Cause}}).
Otherwise, the responder MAY send an INIT-ACK without the DTLS Key Management Parameter
to indicate that it is willing to create a session without security.

Additionally, an SCTP Endpoint acting as responder willing to support
only protected associations considers an INIT chunk not containing
the DTLS 1.3 Chunk Protected Association parameter or another
Protection Solution accepted by own security policy solution as an error,
thus it will reply with an ABORT chunk according to what specified in
{{enoprotected}} indicating that for this endpoint mandatory DTLS Key Management
Parameter is missing.

When initiator and responder have agreed on a DTLS Chunk protected
association and the DTLS Key Management Method by means of handshaking
INIT/INIT-ACK the SCTP association establishment continues until it
has reached the ESTABLISHED state.

When the SCTP session has been established follow the process defined
by the selected DTLS Key Management Method for establishing DTLS Key Contexts
and installing them.

### Offering Multiple Security Solutions

An initiator of an SCTP association may want to offer multiple
different DTLS Key Management Methods for DTLS Chunk or in combination
with other DTLS Key Management Methods in addition to DTLS 1.3 chunks for the
SCTP association.
Multiple DTLS Key Management Methods offered in the INIT chunk will be
ordered based on the priority, where the most preferred will be
in the first position and the least preferred in the last.
The INIT-ACK chunk will only contain the chosen DTLS Key Management Method.
Offers with multiple DTLS Key Management Methods need to
consider the downgrade attack risks (see {{Downgrade-Attacks}}).

The initiator MAY include in its INIT additional DTLS Key Management Methods
that are compatible to offer in parallel with DTLS Chunks. This
may include SCTP-AUTH {{I-D.ietf-tsvwg-rfc4895-bis}}. This will result
in that a number of different SCTP parameters may be included that are
not possible to use simultaneously. Instead the responder needs to parse
these parameters to figure out which sets of solutions that are
offered that the implementation support, and apply its security
policies to select the most appropriate. For example an offer of DTLS
Chunks and SCTP-AUTH, could be interpreted as three different
solutions with different properties, namely DTLS Chunks,
DTLS/SCTP {{RFC6083}}, and SCTP-AUTH {{I-D.ietf-tsvwg-rfc4895-bis}} only.
However, here the DTLS Key Management Parameter can
indicate both preference and which of the solutions that are preferred.

The responder selects one security solutions and includes it in the
response (INIT-ACK). If DTLS chunks was selected and the
DTLS Key Management Method follows the recommendation for down-grade
prevention the endpoints know that down-grade did not happen.

## Termination of a Protected Association {#termination-procedure}

Besides the procedures for terminating an association explained in
{{RFC9260}}, DTLS 1.3 chunk MUST ask the SCTP endpoint for terminating an
association when having an internal error or by detecting a security
violation. Note that the closure of any DTLS Key Management Method doesn't
compromise the capability of sending and receiving protected
SHUTDOWN-COMPLETE chunks as that capability only relies on the
Key Context and not on the DTLS Key Management Method from where it has
been derived.

## Considerations on DTLS Key Management Method {#key-management-considerations}

It is up to the upper layer to manage the keys for the DTLS chunk.
The meaning of DTLS Key Management Method is described in {{dtls-management-method}}.

The DTLS Key Management Method SHOULD use a dedicated PPID to ensure that the
DTLS Key Management Method related user messages are handled by the appropriate layer.

When performing DTLS Key Management, the keys for receiving SHOULD be installed
before the corresponding send keys at the peer. For mitigating downgrade
attacks the key derivation MUST include the DTLS Key Management Method Identifiers
that were sent and received.

The communication is only protected after both sides have configured the keys
for sending and both sides have enforced the protection.

# DTLS Chunk Handling {#dtls-chunk-handling}

The DTLS chunk MUST NOT be bundled with any other chunk.
In particular, it MUST be the first and only chunk.

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
{: #sctp-DTLS-encrypt-chunk-states-1 title="Unprotected SCTP Packet" artwork-align="center"}

The diagram shown in {{sctp-DTLS-encrypt-chunk-states-1}} describes
the structure of an unprotected SCTP packet as described in {{RFC9260}}.

~~~~~~~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Common Header                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          DTLS Chunk                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~
{: #sctp-DTLS-encrypt-chunk-states-2 title="Protected SCTP Packets" artwork-align="center"}

The diagram shown in {{sctp-DTLS-encrypt-chunk-states-2}} describes
the structure of a protected SCTP packet being sent.
Such packets are built with the SCTP common header.
Only one DTLS chunk can be sent in a SCTP packet.

## Sending of DTLS Chunks {#dtls-chunk-sending}

When the credentials for sending DTLS chunks have been configured by the
application, all SCTP packets are sent with a DTLS chunk.

When an SCTP packet needs to be sent, the sequence of chunks is used
as DTLSInnerPlaintext.content and DTLSInnerPlaintext.type is set to
application_data. Then the DTLSCiphertext is computed and used as the
payload of the DTLS chunk. Finally the SCTP common header is prepended.

When the DTLS chunk is used, the DTLS chunk header and the overhead of DTLS
has to be considered to ensure that the final SCTP packet does not exceed
the PMTU.

## Receiving of DTLS Chunks {#dtls-chunk-receiving}

When an SCTP packet containing a DTLS chunk bundled with any other
chunk is received, the packet MUST be silently discarded.

After the application has restricted the SCTP packet handling to protected
SCTP packets only, a SCTP packet not containing a DTLS chunk MUST be
silently discarded.

When processing the payload of the DTLS chunk (i.e. the DTLSCiphertext),
the Restart flag in addition to the unified_hdr is used to find the keys for
processing the encrypted_record.

After the encrypted_record has been verified and decrypted, the
corresponding chunks (the DTLSInnerPlaintext.content) are processed as
defined in the corresponding specifications.

# Abstract API  {#abstract-api}

This section describes an abstract API that is needed between a
DTLS Key Management Method and the DTLS Chunk. This is an
example API and there are alternative implementations.

This API enables the cryptographical protection operations by setting
client/server write keys, sequence number keys, and IVs for primary and
restart DTLS contexts. The write key is the used by the cipher suite
for DTLS record protection (Section 5.2 of {{RFC8446}}. The initilization
vector (IV) is random material used to XOR with the sequence
number to create the nonce per Section 5.3 of {{RFC8446}}.
The sequence number key is used to encrypt the sequence number
(Section 4.2.3 of {{RFC9147}}).

## Cipher Suit Capabilities

The DTLS Key Management Method needs to know which cipher suits defined
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

Parameters :

* SCTP Association:
: Reference to the relevant SCTP association to set the keying material for.

* Restart indication:
: A bit indicating whether the Key is for restart purposes

* DTLS Epoch:
: The DTLS epoch these keys are valid for. Note that Epoch lower than
  3 are not expected as they are used during DTLS handshake.

* Cipher Suit:
: 2 bytes cipher suit identification for the DTLS 1.3 Cipher suit used
  to identify the DTLS AEAD algorithm to perform the DTLS record protection.
  The cipher suite is fixed for a (SCTP Association, Key) pair.

* Write Key, Sequence Number Key and IV:
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

Parameters :

* SCTP Association:
: Reference to the relevant SCTP association to set the keying material for.

* Restart indication:
: A bit indicating whether the Key is for restart purposes

* DTLS Epoch:
: The DTLS epoch these keys are valid for. Note that Epoch lower than
  3 are note expected as they are used during DTLS handshake.

* Cipher Suit:
: 2 bytes cipher suit identification for the DTLS 1.3 Cipher suit used
  to identify the DTLS AEAD algorithm to perform the DTLS record protection.
  The cipher suite is fixed for a (SCTP Association, Key) pair.

* Write Key, Sequence Number Key and IV:
: The cipher suit specific binary object containing all necessary
information for protection operations. The secret will used by the DTLS 1.3 client to
encrypt the record. Binary arbitrary long object depending on the
cipher suit used.

Reply : Established or Failed


## Destroy Client Write Keying Material

A function to destroy the Client Write (transmit) keying material for a given epoch for a given
Key for a given SCTP Association.

Request : Destroy client write key and IV

Parameters :

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

Parameters :

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

Parameters :

* SCTP Association

* Restart indication

* DTLS CID

* DTLS Epoch

Reply: Key set

Parameters : true or false

## Require Protected SCTP Packets

A function to configure an SCTP association to require that normal
SCTP packets being received must be protected in a DTLS Chunk going forward.

Parameters:

* SCTP Association

Reply: Acknowledgement


## Get AEAD Encryption Invocations

Get the number of AEAD encryption invocations (protected messages) for
a given epoch.

Request : Get AEAD Encryption Invocations

Parameters :

* SCTP Association

* Restart indication

* DTLS CID

* DTLS Epoch

Reply: AEAD Encryption Invocations

Parameters : non-negative integer

## Get AEAD Decryption Invocations

Get the number of AEAD decryption invocations for a given epoch.

Request : Get AEAD Decryption Invocations

Parameters :

* SCTP Association

* Restart indication

* DTLS CID

* DTLS Epoch

Reply: AEAD Decryption Invocations

Parameters : non-negative integer

## Get Failed AEAD Decryption Invocations

Get the number of failed AEAD decryption invocations for a given epoch.

Request : Get Failed AEAD Decryption Invocations

Parameters :

* SCTP Association

* Restart indication

* DTLS CID

* DTLS Epoch

Reply: Failed AEAD Decryption Invocations

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
context for a given SCTP Association and primary/restart usages.

Request : Configure Replay Protection

Parameters :

* SCTP Association

* Restart indication

* Configuration parameters

Reply: Replay Protection Configured

Parameters : true or false

# Socket API Considerations {#socket-api}

This section describes how the socket API defined in {{RFC6458}} needs to be
extended to provide a way for the application to control the usage of the
DTLS chunk.

A 'Socket API Considerations' section is contained in all SCTP-related
specifications published after {{RFC6458}} describing an extension for which
implementations using the socket API as specified in {{RFC6458}} would require
some extension of the socket API.
Please note that this section is informational only.

Please also note, that the API described in this section can change in a
non-backwards compatible way during the evolution of this document due to
changed functionality or gained experience during the implementation.

A socket API implementation based on {{RFC6458}} is extended by supporting
several new ``IPPROTO_SCTP``-level socket options and a new flag for
``recvmsg()``.

## ``SCTP_ASSOC_CHANGE`` Notification
When an ``SCTP_ASSOC_CHANGE`` notification (specified in Section 6.1.1 of
{{RFC6458}}) is delivered indicating a ``sac_state`` of ``SCTP_COMM_UP`` or
``SCTP_RESTART`` for an SCTP association where both peers support the
DTLS chunk, ``SCTP_ASSOC_SUPPORTS_DTLS`` should be listed in the
``sac_info`` field.

## A New Flag for ``recvmsg()`` (``MSG_PROTECTED``)

This flag is returned by ``recvmsg()`` in ``msg_flags`` for all user messages
for which all DATA chunks where received in protected SCTP packets.
This also means that if ``sctp_recvv()`` is used, ``MSG_PROTECTED`` is returned
in the ``*flags`` argument.

## Functions

### ``sctp_dtls_nr_cipher_suits()``

``sctp_dtls_nr_cipher_suits()`` returns the number of cypher suits supported
by the SCTP implementation.

The function prototype is:

~~~ c
unsigned int
sctp_dtls_nr_cipher_suits(void);
~~~

This function can be used in combination with ``sctp_dtls_cipher_suits()``.

### ``sctp_dtls_cipher_suits()``

``sctp_dtls_cipher_suits()`` returns the cypher suits supported by the
SCTP implementation.

The function prototype is:

~~~ c
int
sctp_dtls_cipher_suits(uint8_t cipher_suits[][2], unsigned int n);
~~~

and the arguments are
``cipher_suits``:
: An array where the supported cypher suits are stored. A cypher suit is
  represented by two ``uint8_t`` using the IANA assigned values in the
  TLS cipher suit registry {{TLS-CIPHER-SUITS}}.

``n``:
The number of cipher suits which can be stored in ``cipher_suits``.

``sctp_dtls_cipher_suits`` returns ``-1``, if ``n`` is smaller than the number
of cipher suites supported by the stack. If ``n`` is equal to or larger than
the number of cipher suites supported the the SCTP implementation, the
cipher suits are stored in ``cipher_suits`` and the number of supported
cipher suits is returned.

## Socket Options

The following table provides an overview of the ``IPPROTO_SCTP``-level socket
options defined by this section.

| Option Name                          | Data Type                    | Set | Get |
| ``SCTP_DTLS_LOCAL_KMIDS``            | ``struct sctp_dtls_kmids``   | X   | X   |
| ``SCTP_DTLS_REMOTE_KMIDS``           | ``struct sctp_dtls_kmids``   |     | X   |
| ``SCTP_DTLS_SET_SEND_KEYS``          | ``struct sctp_dtls_keys``    | X   |     |
| ``SCTP_DTLS_ADD_RECV_KEYS``          | ``struct sctp_dtls_keys``    | X   |     |
| ``SCTP_DTLS_DEL_RECV_KEYS``          | ``struct sctp_dtls_keys_id`` | X   |     |
| ``SCTP_DTLS_ENFORCE_PROTECTION``     | ``struct sctp_assoc_value``  | X   | X   |
| ``SCTP_DTLS_REPLAY_WINDOW``          | ``struct sctp_assoc_value``  | X   | X   |
| ``SCTP_DTLS_STATS``                  | ``struct sctp_dtls_stats``   |     | X   |
{: #socket-options-table title="Socket Options" cols="l l l l"}

``sctp_opt_info()`` needs to be extended to support:

* ``SCTP_DTLS_LOCAL_KMIDS``,
* ``SCTP_DTLS_REMOTE_KMIDS``,
* ``SCTP_DTLS_ENFORCE_PROTECTION``,
* ``SCTP_DTLS_REPLAY_WINDOW``, and
* ``SCTP_DTLS_STATS``.

### Get or Set the Local DTLS Key Management Identifiers (``SCTP_DTLS_LOCAL_KMIDS``)

This socket option sets the DTLS Key Management identifiers which will be sent
to the peer during the handshake.
It can also be used to retrieve the DTLS Key Manangement identifiers which were
sent during the handshake.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_dtls_pmids {
        sctp_assoc_t sds_assoc_id;
        uint16_t sdp_nr_kmids;
        uint16_t sdp_kmids[];
};
~~~

``sdp_assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  For ``setsockopt()``, only ``SCTP_FUTURE_ASSOC`` can be used.
  For ``getsockopt()``, it is an error to use ``SCTP_{CURRENT|ALL}_ASSOC``.

``sdp_nr_kmids``:
: The number of entries in ``sdp_kmids``.

``sdp_kmids``:
: The DTLS Key Management identifiers which will be or have been sent to the peer
  in the sequence they were contained in the DTLS 1.3 Chunk Protected
  Association parameter and in host byte order.

This socket option can be used with setsockopt() for SCTP endpoints in the
``SCTP_CLOSED`` or ``SCTP_LISTEN`` state to configure the protection method
identifiers to be sent.
When used with ``getsockopt()`` on an SCTP endpoint in the ``SCTP_LISTEN``
state, the protection method identifiers which will be sent can be retrieved.
 If the SCTP endpoint is in a state other that ``SCTP_CLOSED`` or
``SCTP_LISTEN``, the protection method identifiers which have been sent can
be retrieved.

### Get the Remote DTLS Key Mangement Identifiers (``SCTP_DTLS_REMOTE_KMIDS``)

This socket option reports the DTLS Key Mangement identifiers reported by the
peer during the handshake.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_dtls_pmids {
        sctp_assoc_t sds_assoc_id;
        uint16_t sdp_nr_kmids;
        uint16_t sdp_kmids[];
};
~~~

``sdp_assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  It is an error to use ``SCTP_{FUTURE|CURRENT|ALL}_ASSOC``.

``sdp_nr_kmids``:
: The number of entries in ``sdp_kmids``.

``sdp_kmids``:
: The DTLS Key Management identifiers reported by the peer in the sequence they
  were contained in the DTLS Key Management parameter and in host byte order.

This socket option will fail on any SCTP endpoint in state ``SCTP_CLOSED``,
``SCTP_COOKIE_WAIT`` and ``SCTP_COOKIE_ECHOED``.

### Set Send Keys (``SCTP_DTLS_SET_SEND_KEYS``)

Using this socket option allows to add a particular set of keys used for
sending DTLS chunks.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_dtls_keys {
        sctp_assoc_t sdk_assoc_id;
        uint8_t sdk_cipher_suit[2];
        uint8_t sdk_restart;
        uint16_t sdk_key_len;
        uint16_t sdk_iv_len;
        uint16_t sdk_sn_key_len;
        uint32_t sdk_unused; /* if sizeof(sctp_assoc_t) == 4 */
        uint64_t sdk_epoch;
        uint8_t *sdk_key;
        uint8_t *sdk_iv;
        uint8_t *sdk_sn_key;
};
~~~

``sdk_assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  It is an error to use ``SCTP_{FUTURE|CURRENT|ALL}_ASSOC``.

``sdk_cipher_suit``:
: The cipher suit for which the keys are used.

``sdk_restart``:
: If the value is ``0``, the regular keys are added, if a value different
  from ``0`` is used, the restart keys are added.

``sdk_key_len``:
: The length of the initialization vector specified in ``sdk_key``.

``sdk_iv_len``:
: The length of the initialization vector specified in ``sdk_iv``.

``sdk_sn_key_len``:
: The length of the sequence number key specified in ``sdk_sn_key``.

``sdk_unused``:
: This field is ignored.

``sdk_epoch``:
: The epoch for which the keys are added.

``sdk_key``:
: A pointer to the key.

``sdk_iv``:
: A pointer to the initialization vector.

``sdk_sn_key``:
A pointer to the sequence number key.

This socket option can only be used on SCTP endpoints in states other then
``SCTP_LISTEN``, ``SCTP_COOKIE_WAIT`` and ``SCTP_COOKIE_ECHOED``.
If the socket options is successful, all affected DTLS chunks sent will use the
specified keys until the keys are changed again by another call of this
socket option.

### Add Receive Keys (``SCTP_DTLS_ADD_RECV_KEYS``)

Using this socket option allows to add a particular set of keys used for
receiving DTLS chunks.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_dtls_keys {
        sctp_assoc_t sdk_assoc_id;
        uint8_t sdk_cipher_suit[2];
        uint8_t sdk_restart;
        uint16_t sdk_key_len;
        uint16_t sdk_iv_len;
        uint16_t sdk_sn_key_len;
        uint32_t sdk_unused; /* if sizeof(sctp_assoc_t) == 4 */
        uint64_t sdk_epoch;
        uint8_t *sdk_key;
        uint8_t *sdk_iv;
        uint8_t *sdk_sn_key;
};
~~~

``sdk_assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  It is an error to use ``SCTP_{FUTURE|CURRENT|ALL}_ASSOC``.

``sdk_cipher_suit``:
: The cipher suit for which the keys are used.

``sdk_restart``:
: If the value is ``0``, the regular keys are added, if a value different
  from ``0`` is used, the restart keys are added.

``sdk_key_len``:
: The length of the initialization vector specified in ``sdk_key``.

``sdk_iv_len``:
: The length of the initialization vector specified in ``sdk_iv``.

``sdk_sn_key_len``:
: The length of the sequence number key specified in ``sdk_sn_key``.

``sdk_unused``:
: This field is ignored.

``sdk_epoch``:
: The epoch for which the keys are added.

``sdk_key``:
: A pointer to the key.

``sdk_iv``:
: A pointer to the initialization vector.

``sdk_sn_key``:
A pointer to the sequence number key.

This socket option can only be used on SCTP endpoints in states other then
``SCTP_LISTEN``, ``SCTP_COOKIE_WAIT`` and ``SCTP_COOKIE_ECHOED``.

### Delete Receive Keys (``SCTP_DTLS_DEL_RECV_KEYS``)

Using this socket option allows to remove a particular set of keys used for
receiving DTLS chunks.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_dtls_keys_id {
   sctp_assoc_t sdki_assoc_id;
   uint32_t sdki_restart;
   uint64_t sdki_epoch;
}
~~~

``sdki_assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  It is an error to use ``SCTP_{FUTURE|CURRENT|ALL}_ASSOC``.

``sdki_restart``:
: If the value is ``0``, the regular keys are removed, if a value different
  from ``0`` is used, the restart keys are removed.

``sdki_epoch``:
: The epoch for which the keys are removed.

This socket option can only be used on SCTP endpoints in states other then
``SCTP_CLOSED``, ``SCTP_LISTEN``, ``SCTP_COOKIE_WAIT`` and
``SCTP_COOKIE_ECHOED``.

### Set or Get Protection Enforcement (``SCTP_DTLS_ENFORCE_PROTECTION``)

Enabling this socket option on an SCTP endpoint enforces that received
SCTP packets are only processed, if they are protected.
All received packets with the first chunk not being an INIT chunk, INIT ACK
chunk, or DTLS chunk will be silently discarded.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_assoc_value {
   sctp_assoc_t assoc_id;
   uint32_t assoc_value;
};
~~~

``assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  It is an error to use ``SCTP_{FUTURE|CURRENT|ALL}_ASSOC``.

``assoc_value``:
  The value `0` represents that the option is off, any other value represents
  that the option is on.

This socket option is off by default on any SCTP endpoint.
Once protection has been enforced by enabling this socket option on an
SCTP endpoint, it cannot be disabled again.
Whether protection has been enforced on an SCTP endpoint can be queried
in any state other than ``SCTP_CLOSED``.
Protection can be enforced in any state other than ``SCTP_CLOSED``,
``SCTP_COOKIE_WAIT`` and ``SCTP_COOKIE_ECHOED``.

### Get Statistic Counters (``SCTP_DTLS_STATS``)

This socket options allows to get various statistic counters for a
specific SCTP endpoint.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_dtls_stats {
   sctp_assoc_t sds_assoc_id;
   uint32_t sds_dropped_unprotected;
   uint32_t sds_aead_failures;
   uint32_t sds_recv_protected;
   uint32_t sds_sent_protected;
   /* There will be added more fields before the WGLC. */
   /* There might be additional platform specific counters. */
};
~~~

``sds_assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  It is an error to use ``SCTP_{FUTURE|CURRENT|ALL}_ASSOC``.

``sds_dropped_unprotected``:
: The number of unprotected packets received for this SCTP endpoint after
  protection was enforced.

``sds_aead_failures``:
: The number of AEAD failures when processing received DTLS chunks.

``sds_recv_protected``:
: The number of DTLS chunks successfully processed.

``sds_sent_protected``:
: The number of DTLS chunks sent.

### Get or Set the Replay Protection Window Size (``SCTP_DTLS_REPLAY_WINDOW``)

This socket option can be used to configure the replay protection for SCTP
endpoints.

The following structure is used as the ``option_value``:

~~~ c
struct sctp_assoc_value {
   sctp_assoc_t assoc_id;
   uint32_t assoc_value;
};
~~~

``assoc_id``:
: This parameter is ignored for one-to-one style sockets.
  For one-to-many style sockets, this parameter indicates upon which
  SCTP association the caller is performing the action.
  It is an error to use ``SCTP_{CURRENT|ALL}_ASSOC``.

``assoc_value``:
  The maximum number of DTLS chunks in the replay protection window.

# Implementation Considerations

## Privacy Padding of SCTP Packets

To reduce the potential information leakage from packet size
variations one may select to pad the SCTP Packets to uniform packet
sizes. This size may be either the maximum used, or in block sized
increments. However, the padding needs to be done inside of the
encryption envelope.

Both SCTP and DTLS contains mechanisms to pad SCTP payloads, and DTLS
records respectively. If padding of SCTP packets are desired to hide
actual message sizes it RECOMMEDED to use the SCTP Padding Chunk
{{RFC4820}} to generate a consistent SCTP payload size. Support of
this chunk is only required on the sender side, any SCTP receiver will
safely ignore the PAD Chunk. However, if the PAD chunk is not
supported DTLS padding MAY be used.

It needs to be noted that independent if SCTP padding or DTLS padding
is used the padding is not taken into account by the SCTP congestion
control. Extensive use of padding has potential for worsen congestion
situations as the SCTP association will consume more bandwidth than
its derived share by the congestion control.

The use of SCTP PAD chunk is recommended as it at least can enable
future extension or SCTP implementation that account also for the
padding. Use of DTLS padding hides this packet expansion from SCTP.


# IANA Considerations {#IANA-Consideration}

This document defines two new registries in the Stream Control
Transmission Protocol (SCTP) Parameters group that IANA
maintains. Theses registries are for the extra cause codes for
protection related errors.
It also adds registry entries into several other
registries in the Stream Control Transmission Protocol (SCTP)
Parameters group:

*  One new SCTP Chunk Types

*  One new SCTP Chunk Parameter Type

*  Three new SCTP Error Cause Code

And finally the update of one registered SCTP Payload Protocol
Identifier.

## DTLS Key Management Method Identifiers {#IANA-Protection-Solution-ID}

IANA is requested to create a new registry called "DTLS Key Management Method".
This registry is part of the of the Stream
Control Transmission Protocol (SCTP) Parameters grouping.

The purpose of this registry is to assign DTLS Key Management Method
Identifier for any DTLS Key Management Method that is either the DTLS
Chunk combined with a DTLS Key Management Method, offered as an alternative
to DTLS chunk. Any DTLS Key Management Method that is offered
through a parameter exchange during the SCTP handshake are potential
to be included here.

Each entry will be assigned a 16-bit unsigned integer value from the suitable range.

| Identifier | Solution Name | Reference | Contact |
| 0 | DTLS Chunk with Pre- | RFC-TBD | Draft Authors |
| 1-4095 | Available for Assignment using Specification Required policy | | |
| 4096-65535 | Available for Assignment using First Come, First Served policy | | |
{: #iana-psi title="DTLS Key Management Method Identifiers" cols="r l l l"}

New entries in the range 0-4095 are registered following the Specification Required policy
as defined by {{RFC8126}}.  New entries in the range 4096-65535 are first come, first served.

## SCTP Chunk Type

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Chunk Types" registry, IANA is requested to add the one new entry
depicted below in in {{iana-chunk-types}} with a reference to this
document. The registry at time of writing was available at:
https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml#sctp-parameters-1

| ID Value | Chunk Type | Reference |
| 0x41 | DTLS Chunk (DTLS) | RFC-To-Be |
{: #iana-chunk-types title="New Chunk Type Registered" cols="r l l"}

The registration table for the chunk flags of this chunk
type is initially:

| Chunk Flag Value | Chunk Flag Name | Reference |
| 0x01 | R bit | RFC-To-Be |
| 0x02| Unassigned |  |
| 0x04| Unassigned |  |
| 0x08| Unassigned |  |
| 0x10| Unassigned |  |
| 0x20| Unassigned |  |
| 0x40| Unassigned |  |
| 0x80| Unassigned |  |
{: #iana-chunk-flags title="DTLS Chunk Flags" cols="r l l"}

## SCTP Chunk Parameter Types

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Chunk Parameter Types" registry, IANA is requested to add the new
entry depicted below in in {{iana-chunk-parameter-types}} with a
reference to this document. The registry at time of writing was
available at:
https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml#sctp-parameters-2

| ID Value | Chunk Parameter Type | Reference |
| 0x8006   | DTLS Key Management  | RFC-To-Be |
{: #iana-chunk-parameter-types title="New Chunk Type Parameters Registered" cols="r l l"}


## SCTP Error Cause Codes {#IANA-Extra-Cause}

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Error Cause Codes" registry, IANA is requested to add the new
entries depicted below in in {{iana-error-cause-codes}} with a
reference to this document. The registry at time of writing was
available at:
https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml#sctp-parameters-24

| ID Value | Error Cause Codes | Reference |
| TBA10 | Policy Not Met | RFC-To-Be |
| TBA11 | No Common Protection | RFC-To-Be |
{: #iana-error-cause-codes title="Error Cause Codes Parameters Registered" cols="r l l"}


## SCTP Payload Protocol Identifier {#sec-iana-ppid}

In the Stream Control Transmission Protocol (SCTP) Parameters group's
"Payload Protocol Identifiers" registry, IANA is requested to update the
entry depicted below in in {{iana-payload-protection-id}} with a
reference to this document. The registry at time of writing was
available at:
https://www.iana.org/assignments/sctp-parameters/sctp-parameters.xhtml#sctp-parameters-25

| ID Value | SCTP Payload Protocol Identifier | Reference |
| 4242 | DTLS Key Management Messages | RFC-To-Be |
{: #iana-payload-protection-id title="Protection Operator Protocol Identifier Registered" cols="r l l"}


# Security Considerations {#Security-Considerations}

All the security and privacy considerations of the security protocol
used as the Chunk Protection Operator applies.

DTLS replay protection MUST NOT be turned off.

## Privacy Considerations

Use of the SCTP DTLS chunk provides privacy to SCTP by protecting user
data and much of the SCTP control signaling. The SCTP association is
identifiable based on the 5-tuple where the destination IP and
port are fixed for each direction. Advanced privacy features such
as sequence number encryption might
therefore have limited effect.

## AEAD Limit Considerations

Section 4.5.3 of {{RFC9147}} defines limits on the number of records
q that can be protected using the same key as well as limits on the
number of received packets v that fail authentication with each key.
To adhere to these limits the DTLS Key Management Method can
periodically poll the DTLS protection operation function to see
when a limit have been reached or is closed to being reached.
Instead of periodic polling, a callback can be used.

## Downgrade Attacks {#Downgrade-Attacks}

Downgrade attacks may attempt to force the DTLS Key Management Method
by altering the containt of INIT chunk, for instance by removing
all offered DTLS Key Management Methods but the one desired. This is possible
if the attacker is an on-path attacker that can modify packet
because INIT and INIT-ACK chunks are plain text.

Preventing the downgrade attacks is implemented by using at the initiator
the list of offered DTLS Key Management Method sent in the INIT chunk plus
the selected DTLS Key Management Method received in the INIT-ACK chunk from the responder
for deriving the keys from the handshaked secrets obtained during
DTLS initial handshake.
At the responder, the list of offered DTLS Key Management Methods received in
the INIT chunk plus the selected DTLS Key Management Method that is sent
in the INIT-ACK chunk will be used for deriving the keys from the handshaked
secrets obtained during DTLS initial handshake.

If the attacker suceeds in changing the DTLS Key Management Methods in either
INIT, NINT-ACK or both chunks, the peers will not be able deriving the
same keys and the Association will not be possible to proceed.

Thus, as long as the DTLS Key Management Method includes the ordered list of protection
solutions indicators present in the parameter part of the INIT chunk
for the SCTP Association in its key-derivation the association will be
protected from down-grade.

In case any DTLS Key Management Method do not include the parameter content in
its key-derivation down-grade might be possible if that DTLS Key Management Method
method is selected. It is up to endpoint policies to determine
which protection it deems necessary against down-grade attacks.

## Persistent Secure Storage of Restart Key Context {#sec-considertation-storage}

The Restart DTLS Key Context needs to be stored securely and persistent. Securely
as access to this security context may enable an attacker to perform a restart,
resulting a denial of service on the existing SCTP Association. It can also
give the attacker access to the ULP. Thus the storage needs to provide at least
as strong resistant against exfiltration as the main DTLS Key Context store.

When it comes to how to realize persistent storage that is highly
dependent on the ULP and how it can utilize restarted SCTP
Associations. One way can be to have an actual secure persistent storage
solution accessible to the endpoint. In other use cases the persistence part
might be accomplished be keeping the current restart DTLS Key Context with
the ULP State if that is sufficiently secure.


# Acknowledgments

The authors thank Hannes Tschofenig and Tirumaleswar Reddy for their
participation in the design team and their contributions to this document.
We also like to thank Amanda Baber with IANA for feedback on our IANA registry.

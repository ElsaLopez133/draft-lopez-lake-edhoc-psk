---
title: "EDHOC with Pre-Shared Key (PSK) Authentication"
docname: draft-ietf-lake-edhoc-psk-latest
category: std

v3xml2rfc:
  silence:
  - Found SVG with width or height specified

submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
consensus: true
v: 3
area: Security
workgroup: LAKE Working Group
venue:
  group: LAKE
  type: Working Group
  mail: lake@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/lake/
  github: lake-wg/psk
  latest: https://lake-wg.github.io/psk/#go.draft-ietf-lake-edhoc-psk.html

author:
 -
    name: Elsa Lopez-Perez
    organization: Inria
    email: elsa.lopez-perez@inria.fr
 -
    name: Göran Selander
    organization: Ericsson
    email: goran.selander@ericsson.com
 -
    name: John | Preuß Mattsson
    organization: Ericsson
    email: john.mattsson@ericsson.com
 -
    name: Rafael Marin-Lopez
    organization: University of Murcia
    email: rafa@um.es

normative:

  RFC9528:
  RFC9052:
  RFC9053:
  RFC8742:
  RFC8949:
  RFC8610:
  RFC8392:
  SP-800-56A:
    target: https://doi.org/10.6028/NIST.SP.800-56Ar3
    title: Recommendation for Pair-Wise Key-Establishment Schemes Using Discrete Logarithm Cryptography
    seriesinfo:
      "NIST": "Special Publication 800-56A Revision 3"
    author:
      -
        ins: E. Barker
      -
        ins: L. Chen
      -
        ins: A. Roginsky
      -
        ins: A. Vassilev
      -
        ins: R. Davis
    date: April 2018

informative:

--- abstract

This document specifies a Pre-Shared Key (PSK) authentication method for the Ephemeral Diffie-Hellman Over COSE (EDHOC) key exchange protocol. The PSK method improves computational efficiency while ensuring mutual authentication, ephemeral key exchange, and identity protection, while also offering quantum resistance. It is particularly suited for systems where nodes share a PSK out-of-band and enables efficient session resumption with less computational overhead. This document details the PSK method flow, key derivation changes, message formatting, processing, and security considerations.

--- middle

# Introduction

This document defines a Pre-Shared Key (PSK) authentication method for the Ephemeral Diffie-Hellman Over COSE (EDHOC) key exchange protocol {{RFC9528}}. The PSK method balances the complexity of credential distribution with computational efficiency. While symmetrical key distribution is more complex than asymmetrical approaches, PSK authentication offers greater computational efficiency compared to the methods outlined in {{RFC9528}}. The PSK method retains mutual authentication, asymmetric ephemeral key exchange, and identity protections established by {{RFC9528}}. By utilizing the PSK for both authentication and key derivation, this method provides quantum resistance.

EDHOC with PSK authentication benefits systems where two nodes nodes share a Pre-Shared Key (PSK) provided out-of-band. This applies to scenarios like the Authenticated Key Management Architecture (AKMA) in mobile systems or the Peer and Authenticator in Extensible Authentication Protocol (EAP) systems. The PSK method enables the nodes to perform ephemeral key exchange, achieving Perfect Forward Secrecy (PFS). This ensures that even if the PSK is compromised, past communications remain secure against active attackers, while future communications are protected from passive attackers. Additionally, by leveraging the PSK for both authentication and key derivation, the method offers quantum resistance key exchange and authentication.

Another key use case of PSK authentication in the EDHOC protocol is session resumption. This enables previously connected parties to quickly reestablish secure communication using pre-shared keys from a prior session, reducing the overhead associated with key exchange and asymmetric authentication. By using PSK authentication, EDHOC allows session keys to be refreshed with significantly lower computational overhead compared to public-key authentication.

Section 3 provides an overview of the PSK method flow and credentials. Section 4 outlines the changes to key derivation compared to {{RFC9528}}, Section 5 details message formatting and processing, and Section 6 discusses security considerations. How to derive keys for resumption is described in Section TODO.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Readers are expected to be familiar with the terms and concepts described in CBOR {{RFC8949}}, CBOR Sequences {{RFC8742}}, COSE Structures and Processing {{RFC9052}}, COSE Algorithms {{RFC9053}}, CWT and CCS {{RFC8392}}, and the Concise Data Definition Language (CDDL) {{RFC8610}}, which is used to express CBOR data structures.

# Protocol

In this method, the Pre-Shared Key identifier (ID_CRED_PSK) is sent in message_3. The ID_CRED_PSK allows retrieval of CRED_PSK, a COSE_Key compatible authentication credential that contains the PSK. Through this document we will refer to the Pre-Shared Key authentication method as EDHOC-PSK.

## Credentials

Initiator and Responder are assumed to have a PSK with good amount of randomness and the requirements that:

- Only the Initiator and the Responder have access to the PSK.
- The Responder is able to retrieve the PSK using ID_CRED_PSK.

where:

- ID_CRED_PSK is a COSE header map containing header parameters that can identify a pre-shared key. For example:

~~~~~~~~~~~~
ID_CRED_PSK = {4 : h'0f' }
~~~~~~~~~~~~

- CRED_PSK is a COSE_Key compatible authentication credential, i.e., a CBOR Web Token (CWT) or CWT Claims Set (CCS) {{RFC8392}} whose 'cnf' claim uses the confirmation method 'COSE_Key' encoding the PSK. For example:

~~~~~~~~~~~~
{                                               /CCS/
  2 : "mydotbot",                               /sub/
  8 : {                                         /cnf/
    1 : {                                       /COSE_Key/
       1 : 4,                                   /kty/
       2 : h'0f',                               /kid/
      -1 : h'50930FF462A77A3540CF546325DEA214'  /k/
    }
  }
}
~~~~~~~~~~~~

The purpose of ID_CRED_PSK is to facilitate the retrieval of the PSK.
It is RECOMMENDED that it uniquely identifies the CRED_PSK as the recipient might otherwise have to try several keys.
If ID_CRED_PSK contains a single 'kid' parameter, then the compact encoding is applied; see {{Section 3.5.3.2 of RFC9528}}.
The authentication credential CRED_PSK substitutes CRED_I and CRED_R specified in {{RFC9528}}, and, when applicable, MUST follow the same guidelines described in {{Section 3.5.2 and Section 3.5.3 of RFC9528}}.

## Message Flow of PSK

The ID_CRED_PSK is sent in message_3, encrypted using a key derived from the ephemeral shared secret, G_XY. The Responder authenticates the Initiator first.
{{fig-variant2}} shows the message flow of PSK authentication method.

~~~~~~~~~~~~ aasvg
Initiator                                                   Responder
|                  METHOD, SUITES_I, G_X, C_I, EAD_1                |
+------------------------------------------------------------------>|
|                             message_1                             |
|                                                                   |
|                      G_Y, Enc( C_R, EAD_2 )                       |
|<------------------------------------------------------------------+
|                             message_2                             |
|                                                                   |
|                   Enc( ID_CRED_PSK, AEAD( EAD_3 ) )               |
+------------------------------------------------------------------>|
|                             message_3                             |
|                                                                   |
|                           AEAD( EAD_4 )                           |
|<------------------------------------------------------------------+
|                             message_4                             |
~~~~~~~~~~~~
{: #fig-variant2 title="Overview of Message Flow of PSK." artwork-align="center"}

This approach provides protection against passive attackers for both Initiator and Responder.
message_4 remains optional, but is needed to authenticate the Responder and achieve mutual authentication in EDHOC if not relaying on external applications, such as OSCORE. With this fourth message, the protocol achieves both explicit key confirmation and mutual authentication.

# Key Derivation {#key-der}

The pseudorandom keys (PRKs) used for PSK authentication method in EDHOC are derived using EDHOC_Extract, as done in {{RFC9528}}.

~~~~~~~~~~~~
PRK  = EDHOC_Extract( salt, IKM )
~~~~~~~~~~~~

where the salt and input keying material (IKM) are defined for each key.
The definition of EDHOC_Extract depends on the EDHOC hash algorithm selected in the cipher suite.

{{fig-variant2key}} lists the key derivations that differ from those specified in {{Section 4.1.2 of RFC9528}}.

~~~~~~~~~~~~
PRK_3e2m    = PRK_2e
PRK_4e3m    = EDHOC_Extract( SALT_4e3m, CRED_PSK )
KEYSTREAM_3 = EDHOC_KDF( PRK_3e2m, TBD1, TH_3, plaintext_length_3 )
K_3         = EDHOC_KDF( PRK_4e3m, TBD2, TH_3, key_length )
IV_3        = EDHOC_KDF( PRK_4e3m, TBD3, TH_3, iv_length )
~~~~~~~~~~~~
{: #fig-variant2key title="Key Derivation of EDHOC PSK Authentication Method." artwork-align="center"}

where:

- KEYSTREAM_3 is used to encrypt the ID_CRED_PSK in message_3.
- TH_3 = H( TH_2, PLAINTEXT_2, CRED_PSK )

Additionally, the definition of the transcript hash TH_4 is modified as:

- TH_4 = H( TH_3, ID_CRED_PSK, ? EAD_3, CRED_PSK )

# Message Formatting and Processing

This section specifies the differences in message formatting and processing compared to {{Section 5 of RFC9528}}.

## Message 1

Message 1 is formatted and processed as specified in {{Section 5.2 of RFC9528}}.

## Message 2

### Formatting of Message 2

Message 2 is formatted as specified in {{Section 5.3.1 of RFC9528}}.

### Responder Composition of Message 2

CIPHERTEXT_2 is calculated with a binary additive stream cipher, using a keystream generated with EDHOC_Expand, and the following plaintext:

* PLAINTEXT_2B = ( C_R, ? EAD_2 )
* CIPHERTEXT_2 = PLAINTEXT_2B XOR KEYSTREAM_2

Contrary to {{RFC9528}}, ID_CRED_R, MAC_2, and Signature_or_MAC_2 are not used. C_R, EAD_2, and KEYSTREAM_2 are defined in {{Section 5.3.2 of RFC9528}}.

### Initiator Processing of Message 2

Compared to {{Section 5.3.3 of RFC9528}}, ID_CRED_R is not made available to the application in step 4, and steps 5 and 6 are skipped

## Message 3

### Formatting of Message 3

Message 3 is formatted as specified in {{Section 5.4.1 of RFC9528}}.

### Initiator Composition of Message 3

* CIPHERTEXT_3 is calculated with a binary additive stream cipher, using a keystream generated with EDHOC_Expand, and the following plaintext:

   * PLAINTEXT_3A = ( ID_CRED_PSK / bstr / -24..23, CIPHERTEXT_3B )

      * If ID_CRED_PSK contains a single 'kid' parameter, i.e., ID_CRED_PSK = { 4 : kid_PSK }, then the compact encoding is applied, see {{Section 3.5.3.2 of RFC9528}}.

   * Compute KEYSTREAM_3 as in {{key-der}}, where plaintext_length is the length of PLAINTEXT_3A. For the case of plaintext_length exceeding the EDHOC_KDF output size, see {{Appendix G of RFC9528}}.

   * CIPHERTEXT_3 = PLAINTEXT_3A XOR KEYSTREAM_3

* CIPHERTEXT_3B is the 'ciphertext' of COSE_Encrypt0 object as defined in {{Section 5.2 and Section 5.3 of RFC9528}}, with the EDHOC AEAD algorithm of the selected cipher suite, using the encryption key K_3, the initialization vector IV_3 (if used by the AEAD algorithm), the parameters described in {{Section 5.2 of RFC9528}}, plaintext PLAINTEXT_3B and the following parameters as input:

  - protected = h''
  - external_aad = << ID_CRED_PSK, TH_3 >>
  - K_3 and IV_3 as defined in {{key-der}}
  - PLAINTEXT_3B = ( ? EAD_3 )

The Initiator computes TH_4 = H( TH_3, ID_CRED_PSK, PLAINTEXT_3B, CRED_PSK ), defined in {{key-der}}.

### Responder Processing of Message 3

## Message 4

Message 4 is formatted and processed as specified in {{Section 5.5 of RFC9528}}.

Compared to {{RFC9528}}, a fourth message does not only provide key confirmation but also Responder authentication. To authenticate the Responder and achieve mutual authentication, a fourth message is mandatory.

After verifying message_4, the Initiator is assured that the Responder has calculated the key PRK_out (key confirmation) and that no other party can derive the key. The Initiator MUST NOT persistently store PRK_out or application keys until the Initiator has verified message_4 or a message protected with a derived application key, such as an OSCORE message, from the Responder and the application has authenticated the Responder.

# Security Considerations

When evaluating the security considerations, it is important to differentiate between the initial handshake and session resumption phases.

  1. **Initial Handshake**: a fresh CRED_PSK is used to establish a secure connection.
  2. **Session Resumption**: the same PSK identifier (ID_CRED_PSK) is reused each time EDHOC is executed.
    While this enhances efficiency and reduces the overhead of key exchanges, it presents privacy risks if not managed properly.
    Over multiple resumption sessions, initiating a full EDHOC session changes the resumption PSK, resulting in a new ID_CRED_PSK.
    The periodic renewal of the CRED_PSK and ID_CRED_PSK helps mitigate long-term privacy risks associated with static key identifiers.

## Identity Protection

The current EDHOC methods protect the Initiator’s identity against active attackers and the Responder’s identity against passive attackers (See [Section 9.1 of RFC9528](https://www.rfc-editor.org/rfc/rfc9528.html#section-9.1)).
With EDHOC-PSK authentication method, both the Initiator's and Responder's identities are protected against passive attackers, but not against active attackers.

## Number of Messages

The current EDHOC protocol consists of three mandatory messages and an optional fourth message.
In the case of EDHOC-PSK authentication method, message_4 remains optional, but mutual authentication is not guaranteed without it, or an OSCORE message or any application data that confirms that the Responder owns the PSK. Additionally, with this fourth message the protocol achieves explicit key confirmation in addition to mutual authentication.

## External Authorization Data

The Initiator and Responder can send information in EAD_3 and EAD_4 or in OSCORE messages in parallel with message_3 and message_4.
This is possible because the Initiator knows that only the Responder with access to the CRED_PSK can decrypt the information.

## Attacks

EDHOC-PSK authentication method offers privacy and resistance to passive attacks but might be vulnerable to certain active attacks due to delayed authentication.

## Independence of Session Keys

NIST mandates that that an ephemeral private key shall be used in exactly one key-establishment transaction (see Section 5.6.3.3 of {{SP-800-56A}}). This requirement is essential for preserving session key independence and ensuring forward secrecy. The EDHOC protocol complies with this NIST requirement.

In other protocols, the reuse of ephemeral keys, particularly when combined with implementation flaws such as the absence of public key validation, has resulted in critical security vulnerabilities. Such weaknesses have allowed attackers to recover the so called “ephemeral” private key from a compromised session, thereby enabling them to compromise the security of both past and future sessions between legitimate parties. Assuming breach and minimizing the impact of compromise are fundamental zero-trust principles.

# Privacy Considerations

# Unified Approach and Recommendations

For use cases involving the transmission of application data, application data can be sent concurrently with message_3, maintaining the protocol's efficiency.
In applications such as EAP-EDHOC, where application data is not sent, message_4 is mandatory. Thus, EDHOC-PSK authentication method does not include any extra messages.
Other implementations may continue using OSCORE in place of EDHOC message_4, with a required change in the protocol's language to:
      The Initiator SHALL NOT persistently store PRK_out or application keys until the Initiator has verified message_4 or a message protected with a derived application key, such as an OSCORE message.

This change ensures that key materials are only stored once their integrity and authenticity are confirmed, thereby enhancing privacy by preventing early storage of potentially compromised keys.

Lastly, whether the Initiator or Responder authenticates first is not relevant when using symmetric keys.
This consideration was important for the privacy properties when using asymmetric authentication but is not significant in the context of symmetric key usage.

# IANA Considerations

This document has IANA actions.

## EDHOC Method Type Registry

## EDHOC Exporter Label Registry

## EDHOC Info Label Registry

--- back

# CDDL Definitions {#CDDL}

This section compiles the CDDL definitions for easy reference, incorporating errata filed against {{RFC9528}}.

~~~~~~~~~~~ CDDL
suites = [ 2* int ] / int

ead = (
  ead_label : int,
  ? ead_value : bstr,
)

EAD_1 = (1* ead)
EAD_2 = (1* ead)
EAD_3 = (1* ead)
EAD_4 = (1* ead)

message_1 = (
  METHOD : int,
  SUITES_I : suites,
  G_X : bstr,
  C_I : bstr / -24..23,
  ? EAD_1,
)

message_2 = (
  G_Y_CIPHERTEXT_2 : bstr,
)

PLAINTEXT_2B = (
  C_R : bstr / -24..23,
  ? EAD_2,
)

message_3 = (
  CIPHERTEXT_3 : bstr,
)

PLAINTEXT_3A = (
  ID_CRED_PSK : header_map / bstr / -24..23,
  CIPHERTEXT_3B : bstr,
)

PLAINTEXT_3B = (
  ? EAD_3
)

message_4 = (
  CIPHERTEXT_4 : bstr,
)

PLAINTEXT_4 = (
  ? EAD_4,
)

error = (
  ERR_CODE : int,
  ERR_INFO : any,
)

info = (
  info_label : int,
  context : bstr,
  length : uint,
)
~~~~~~~~~~~

# Change Log

RFC Editor: Please remove this appendix.

* From -02 to -03

  * Updated abstract and Introduction
  * Changed message_3 to hide the identity lenght from passive attackers
  * CDDL Definitions
  * Security considerations of independence of Session Keys
  * Editorial changes

* From -01 to -02

  * Changes to message_3 formatting and processing

* From -00 to -01

  * Editorial changes and corrections

# Acknowledgments
{:numbered="false"}

The authors want to thank
{{{Christian Amsüss}}},
{{{Scott Fluhrer}}},
and
{{{Marco Tiloca}}}
for reviewing and commenting on intermediate versions of the draft.

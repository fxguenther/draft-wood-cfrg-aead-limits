---
title: Usage Limits on AEAD Algorithms
abbrev: AEAD Limits
docname: draft-wood-cfrg-aead-limits-latest
date:
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -  ins: F. Günther
    name: Felix Günther
    org: ETH Zurich
    email: mail@felixguenther.info
 -  ins: M. Thomson
    name: Martin Thomson
    org: Mozilla
    email: mt@lowentropy.net
 -  ins: C. A. Wood
    name: Christopher A. Wood
    org: Cloudflare
    email: caw@heapingbits.net

normative:
  GCM:
    title: "Recommendation for Block Cipher Modes of Operation: Galois/Counter Mode (GCM) and GMAC"
    date: 2007-11
    author:
      - ins: M. Dworkin
    seriesinfo:
      NIST: Special Publication 800-38D
  GCMProofs:
    title: "Breaking and Repairing GCM Security Proofs"
    target: https://eprint.iacr.org/2012/438.pdf
    date: 2012-08-01
    author:
      - ins: T. Iwata
      - ins: K. Ohashi
      - ins: K. Minematsu
  ChaCha20Poly1305Bounds:
    title: "A Security Analysis of the Composition of ChaCha20 and Poly1305"
    author:
      - ins: G. Procter
    date: 2014-08-11
    target: https://eprint.iacr.org/2014/613.pdf
  AEBounds:
    title: "Limits on Authenticated Encryption Use in TLS"
    author:
      - ins: A. Luykx
      - ins: K. Paterson
    date: 2016-03-08
    target: http://www.isg.rhul.ac.uk/~kp/TLS-AEbounds.pdf
  MUSecurity:
    title: "Public-Key Encryption in a Multi-user Setting: Security Proofs and Improvements"
    author:
      - ins: M. Bellare
      - ins: A. Boldyreva
      - ins: S. Micali
    date: 2000-05
    target: https://cseweb.ucsd.edu/~mihir/papers/musu.pdf

informative:
  NonceDisrespecting:
    target: https://eprint.iacr.org/2016/475.pdf
    title: "Nonce-Disrespecting Adversaries -- Practical Forgery Attacks on GCM in TLS"
    author:
      - ins: H. Bock
      - ins: A. Zauner
      - ins: S. Devlin
      - ins: J. Somorovsky
      - ins: P. Jovanovic
    date: 2016-05-17

--- abstract

An Authenticated Encryption with Associated Data (AEAD) algorithm provides
confidentiality and integrity.  Excessive use of the same key can give an
attacker advantages in breaking these properties.  This document provides simple
guidance for users of common AEAD functions about how to limit the use of keys
in order to bound the advantage given to an attacker.  It considers limits in
both single- and multi-user settings.

--- middle

# Introduction

An Authenticated Encryption with Associated Data (AEAD) algorithm
provides confidentiality and integrity. {{!RFC5116}} specifies an AEAD
as a function with four inputs -- secret key, nonce, plaintext,
and optional associated data -- that produces ciphertext output and error code
indicating success or failure. The ciphertext is typically composed of the encrypted
plaintext bytes and an authentication tag.

The generic AEAD interface does not describe usage limits.  Each AEAD algorithm
does describe limits on its inputs, but these are formulated as strict
functional limits, such as the maximum length of inputs, which are determined by
the properties of the underlying AEAD composition.  Degradation of the security
of the AEAD as a single key is used multiple times is not given a thorough
treatment.

These limits might also be influenced by the number of "users" of
a given key. In the traditional setting, there is one key shared between a two
parties. Any limits on the maximum length of inputs or encryption operations
apply to that single key. The attacker's goal is to break security
(confidentiality or integrity) of that specific key. However, in practice, there
are often many users with independent keys. In this "multi-user" setting, the
attacker is assumed to have done some offline work to help break security of
single key (or user), where the attacker cannot choose which key is attacked.
As a result, AEAD algorithm limits may depend on offline work and the number
of users. However, given that a multi-user attacker does not target any specific
user, acceptable advantages may differ from that of the single-user setting.

The number of times a single pair of key and nonce can be used might also be
relevant to security.  For some algorithms, such as AEAD_AES_128_GCM or
AEAD_AES_256_GCM, this limit is 1 and using the same pair of key and nonce has
serious consequences for both confidentiality and integrity; see
{{NonceDisrespecting}}.  Nonce-reuse resistant algorithms like
AEAD_AES_128_GCM_SIV can tolerate a limited amount of nonce reuse.

It is good practice to have limits on how many times the same key (or pair of
key and nonce) are used.  Setting a limit based on some measurable property of
the usage, such as number of protected messages or amount of data transferred,
ensures that it is easy to apply limits.  This might require the application of
simplifying assumptions.  For example, TLS 1.3 specifies limits on the number of
records that can be protected, using the simplifying assumption that records are
the same size; see Section 5.5 of {{?TLS=RFC8446}}.

Currently, AEAD limits and usage requirements are scattered among peer-reviewed
papers, standards documents, and other RFCs. Determining the correct limits for
a given setting is challenging as papers do not use consistent labels or
conventions, and rarely apply any simplifications that might aid in reaching a
simple limit.

The intent of this document is to collate all relevant information about the
proper usage and limits of AEAD algorithms in one place.  This may serve as a
standard reference when considering which AEAD algorithm to use, and how to use
it.

# Requirements Notation

{::boilerplate bcp14}

# Notation

This document defines limitations in part using the quantities below.

| Symbol  | Description |
|-:-|:-|
| n | Number of bits per block |
| k | Size of the AEAD key (in bits) |
| t | Size of the authentication tag (in bits) |
| l | Length of each message (in blocks)
| s | Total plaintext length in all messages (in blocks) |
| q | Number of user encryption attempts |
| v | Number of attacker forgery attempts |
| p | Adversary attack probability |
| o | Offline adversary work (in number of encryption and decryption queries; multi-user setting only) |
| u | Number of users or keys (multi-user setting only) |

For each AEAD algorithm, we define the confidentiality and integrity advantage
roughly as the advantage an attacker has in breaking the corresponding security
property for the algorithm. Specifically:

- Confidentiality advantage (CA): The advantage of an attacker succeeding in breaking
the confidentiality properties of the AEAD. In this document, the definition of
confidentiality advantage is the increase in the probability that an attacker is
able to successfully distinguish an AEAD ciphertext from the output of a random
function.

- Integrity advantage (IA): The probability of an attacker succeeding in breaking
the integrity properties of the AEAD. In this document, the definition of
integrity advantage is the probability that an attacker is able to forge a
ciphertext that will be accepted as valid.

Each application requires a different application of limits in order to keep CA
and IA sufficiently small.  For instance, TLS aims to keep CA below 2^-60 and IA
below 2^-57. See {{?TLS=RFC8446}}, Section 5.5.

# Calculating Limits

Once an upper bound on CA and IA are determined, this document
defines a process for determining two overall limits:

- Confidentiality limit (CL): The number of bytes of plaintext and maybe
  authenticated additional data (AAD) an application can encrypt before giving
  the adversary a non-negligible CA.

- Integrity limit (IL): The number of bytes of ciphertext and maybe authenticated
  additional data (AAD) an application can process, either successfully or not,
  before giving the adversary a non-negligible IA.

For an AEAD based on a block function, it is common for these limits to be
expressed instead in terms of the number of blocks rather than bytes.
Furthermore, it might be more appropriate to track the number of messages rather
than track bytes.  Therefore, the guidance is usually based on the total number
of blocks processed (s).  To aid in calculating limits for message-based
protocols, a formulation of limits that includes a maximum message size (l) is
included.

All limits are based on the total number of messages, either the number of
protected messages (q) or the number of forgery attempts (v); which correspond
to CL and IL respectively.

Limits are then derived from those bounds using a target attacker probability.
For example, given a confidentiality advantage of `v * (8l / 2^106)` and attacker
success probability of `p`, the algorithm remains secure, i.e., the adversary's
advantage does not exceed the probability of success, provided that
`v <= (p * 2^106) / 8l`. In turn, this implies that `v <= (p * 2^106) / 8l`
is the corresponding limit.

# Single-User AEAD Limits {#limits}

This section summarizes the confidentiality and integrity bounds and limits for modern AEAD algorithms
used in IETF protocols, including: AEAD_AES_128_GCM {{!RFC5116}}, AEAD_AES_256_GCM {{!RFC5116}},
AEAD_AES_128_CCM {{!RFC5116}}, AEAD_CHACHA20_POLY1305 {{!RFC8439}}, AEAD_AES_128_CCM_8 {{!RFC6655}}.

The CL and IL values bound the total number of encryption and forgery queries (q and v).
Alongside each value, we also specify these bounds.

## AEAD_AES_128_GCM and AEAD_AES_256_GCM

The CL and IL values for AES-GCM are derived in {{AEBounds}} and summarized below.
For this AEAD, n = 128 and t = 128 {{GCM}}. In this example, the length s is the sum
of AAD and plaintext, as described in {{GCMProofs}}.

### Confidentiality Limit

~~~
CA <= ((s + q + 1)^2) / 2^129
~~~

This implies the following usage limit:

~~~
q + s <= p^(1/2) * 2^(129/2) - 1
~~~

Which, for a message-based protocol with `s <= q * l`, if we assume that every
packet is size `l`, produces the limit:

~~~
q <= (p^(1/2) * 2^(129/2) - 1) / (l + 1)
~~~

### Integrity Limit

~~~
IA <= 2 * (v * (l + 1)) / 2^128
~~~

This implies the following limit:

~~~
v <= (p * 2^127) / (l + 1)
~~~

## AEAD_CHACHA20_POLY1305

The only known analysis for AEAD_CHACHA20_POLY1305 {{ChaCha20Poly1305Bounds}}
combines the confidentiality and integrity limits into a single expression,
covered below:

<!-- I've got to say that this is a pretty unsatisfactory situation. -->

~~~
CA <= v * ((8 * l) / 2^106)
IA <= v * ((8 * l) / 2^106)
~~~

This advantage is a tight reduction based on the underlying Poly1305 PRF {{!Poly1305=DOI.10.1007/11502760_3}}.
It implies the following limit:

~~~
v <= (p * 2^103) / l
~~~

## AEAD_AES_128_CCM

The CL and IL values for AEAD_AES_128_CCM are derived from {{!CCM-ANALYSIS=DOI.10.1007/3-540-36492-7_7}}
and specified in the QUIC-TLS mapping specification {{?I-D.ietf-quic-tls}}. This analysis uses the total
number of underlying block cipher operations to derive its bound. For CCM, this number is the sum of:
the length of the associated data in blocks, the length of the ciphertext in blocks, the length of
the plaintext in blocks, plus 1.

In the following limits, this is simplified to a value of twice the length of the packet in blocks,
i.e., 2l represents the effective length, in number of block cipher operations, of a message with
l blocks. This simplification is based on the observation that common applications of this AEAD carry
only a small amount of associated data compared to ciphertext. For example, QUIC has 1 to 3 blocks of AAD.

For this AEAD, n = 128 and t = 128.

### Confidentiality Limit

~~~
CA <= (2l * q)^2 / 2^n
   <= (2l * q)^2 / 2^128
~~~

This implies the following limit:

~~~
q <= sqrt((p * 2^126) / l^2)
~~~

### Integrity Limit

~~~
IA <= v / 2^t + (2l * (v + q))^2 / 2^n
   <= v / 2^128 + (2l * (v + q))^2 / 2^128
~~~

This implies the following limit:

~~~
v + (2l * (v + q))^2 <= p * 2^128
~~~

In a setting where `v` or `q` is sufficiently large, `v` is negligible compared to
`(2l * (v + q))^2`, so this this can be simplified to:

~~~
v + q <= p^(1/2) * 2^63 / l
~~~

## AEAD_AES_128_CCM_8

The analysis in {{!CCM-ANALYSIS}} also applies to this AEAD, but the reduced tag
length of 64 bits changes the integrity limit calculation considerably.

~~~
IA <= v / 2^t + (2l * (v + q))^2 / 2^n
   <= v / 2^64 + (2l * (v + q))^2 / 2^128
~~~

This results in reducing the limit on `v` by a factor of 2^64.

~~~
v * 2^64 + (2l * (v + q))^2 <= p * 2^128
~~~

# Multi-User AEAD Limits {#mu-limits}

In the public-key, multi-user setting, {{MUSecurity}} proves that the success
probability in attacking one of many independently users is bounded by the
success probability of attacking a single user multiplied by the number of users
present. Each user is assumed to have an independent and identically distributed
key, though some may share nonces with some very small probability. Absent
concrete multi-user bounds, this means the attacker advantage in the multi-user
setting is the product of the single-user advantage and the number of users.

This section summarizes the confidentiality and integrity bounds and limits for
the same algorithms as in {{limits}}, except in the multi-user setting. The CL
and IL values bound the total number of encryption and forgery queries (q and v).
Alongside each value, we also specify these bounds.

## AEAD_AES_128_GCM and AEAD_AES_256_GCM

Concrete multi-user bounds for AEAD_AES_128_GCM and AEAD_AES_256_GCM exist
due to {{?GCM-MU=DOI.10.1145/3243734.3243816}}. AES-GCM without nonce
randomization is also discussed in {{?GCM-MU}}, though this section does not
include those results as they do not apply to protocols such as TLS 1.3 {{?RFC8446}}.

### Confidentiality Limit

<!-- From (1) in {{GCM-MU}}, assuming n=2^7, \sigma = (v+q)*l, B = \sigma/u, dropping the last term
  (with denominator 2^(k+n), and dropping the first term since the adversary's
  offline work dominates -->
~~~
CA <= ((v + q) * l)^2 / (u * 2^128)
~~~

This implies the following limit:

~~~
v + q <= sqrt(p * u * 2^128) / l
~~~

### Integrity Limit

<!-- From Bad_8 advantage contribution to the inequality from 4.3 in {{GCM-MU}},
  assuming \sigma = (v+e)*l -->
~~~
CA <= (1 / 2^1024) + ((2 * (v + q)) / 2^256)
        + ((2 * o * (v + q)) / 2^(k + 128))
        + (128 * ((v + q) + ((v + q) * l)) / 2^k)
~~~

When k = 128, the last term in this inequality dominates. Thus, we can simplify
this to:

~~~
CA <= (128 * ((v + q) + ((v + q) * l)) / 2^128)
~~~

This implies the following limit:

~~~
v + q <= (p * 2^128) / (128 * (l + 1))
~~~

When k = 256, the second and fourth terms in the CA inequality dominate. Thus, we
can simplify this to:

~~~
CA <= ((2 * (v + q)) / 2^256)
        + (128 * ((v + q) + ((v + q) * l)) / 2^256)
~~~

This implies the following limit:

~~~
v + q <= (p * 2^255) / ((64 * l) + 65)
~~~

## AEAD_CHACHA20_POLY1305, AEAD_AES_128_CCM, and AEAD_AES_128_CCM_8

There are currently no concrete multi-user bounds for AEAD_CHACHA20_POLY1305,
AEAD_AES_128_CCM, or AEAD_AES_128_CCM_8. Thus, to account for the additional
factor `u`, i.e., the number of users, each `p` term in the confidentiality and
integrity limits is replaced with `p / u`.

### AEAD_CHACHA20_POLY1305

The combined confidentiality and integrity limit for AEAD_CHACHA20_POLY1305 is
as follows.

~~~
v <= ((p / u) * 2^106) / 8l
  <= (p * 2^103) / (l * u)
~~~

### AEAD_AES_128_CCM and AEAD_AES_128_CCM_8

The integrity limit for AEAD_AES_128_CCM is as follows.

~~~
v + q <= (p / u)^(1/2) * 2^63 / l
~~~

Likewise, the integrity limit for AEAD_AES_128_CCM_8 is as follows.

~~~
v * 2^64 + (2l * (v + q))^2 <= (p / u) * 2^128
~~~

# Security Considerations {#sec-considerations}

Many of the formulae in this document depend on simplifying assumptions that are
not universally applicable.  When using this document to set limits, it is
necessary to validate all these assumptions for the setting in which the limits
might apply.  In most cases, the goal is to use assumptions that result in
setting a more conservative limit, but this is not always the case.

# IANA Considerations

This document does not make any request of IANA.

--- back

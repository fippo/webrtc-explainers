# WebRTC RTP Header Extension Encryption

## Authors:
- [Philipp Hancke](https://github.com/fippo)

## Introduction
WebRTC uses encrypted SRTP to transfer audio and video data. The packet format is the RTP
format specified in [RFC3550], with an unencrypted header and a payload that is encrypted by
SRTP ([RFC3711]):
```
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |V=2|P|X|  CC   |M|     PT      |       sequence number         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                           timestamp                           |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           synchronization source (SSRC) identifier            |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
   |            contributing source (CSRC) identifiers             |
   |                             ....                              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

RTP header extensions follow the contributing sources in the RTP header and are a structured way to attach metadata to
individual packets such as an "audio level" or the "absolute send time". As part of the
RTP header the values are not encrypted and the mechanism defined in [RFC6904] is cumbersome.

The IETF published [CRYPTEX] (RFC 9335) in January 2023 to address the shortcomings. CRYPTEX
reuses the existing SRTP encryption to additionally cover the RTP header extensions and the
contributing source (CSRC) identifiers, and is negotiated with a single `a=cryptex` SDP
attribute rather than the per-extension negotiation of RFC 6904.

While this and the [W3C API] have been around for a while, this recently became available
in [libSRTP] which is a widely used library that browsers depend on. Note that the W3C API
is currently marked as a feature at risk, since there was no implementation.

## Code sample
Header extension encryption is offered by default, with fallback to unencrypted when the
remote peer does not support it, so no code changes are required:
```javascript
const pc = new RTCPeerConnection();
```

## Design notes
The original issue got [some implementation feedback](https://github.com/w3c/webrtc-extensions/issues/47#issuecomment-4554321288)
from me. Apart from editorial issues with alignment between the RFC and the W3C semantics, there are two open points:
* it is questionable whether a per-transceiver flag is actually required since
  the configuration is global and the API does not offer per-transceiver control
  (it only exposes a per-transceiver readonly signal for what was negotiated).
* the policy enum currently only allows negotiating or requiring encryption; an
  additional value that allows not offering cryptex at all might make the rollout
  easier.

## Links
[RFC3550]: https://datatracker.ietf.org/doc/html/rfc3550#section-5.1
[RFC3711]: https://datatracker.ietf.org/doc/html/rfc3711
[RFC6904]: https://datatracker.ietf.org/doc/html/rfc6904
[CRYPTEX]: https://datatracker.ietf.org/doc/html/rfc9335
[W3C API]: https://w3c.github.io/webrtc-extensions/#rtp-header-extension-encryption
[libSRTP]: https://github.com/cisco/libsrtp

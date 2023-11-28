# WebRTC RTCRtpSender setParameters() extensions for controlling key frame generation

## Authors:
- [Philipp Hancke](https://github.com/fippo)
- [Max Solodovnikov](https://github.com/solmaks)

## Introduction
WebRTC hides many details of how video encoding is done. The generation of key frames (i.e. video
frames which can be decoded without any dependencies) is one of these. In particular for
multiparty video conferencing key frames are required when a new participant joins.
In many of these cases, the server required for multiparty conferencing will request the
generation of a keyFrame via an RTCP protocol message. However, some applications may want
to also control what spatial layers are encoded as described in
  https://webrtchacks.com/suspending-simulcast-streams/
In those cases, turning off a high resolution spatial layer will cause participants
receiving that layer to "switch down". In order to switch down a key frame is required.
This can cause the following flow events between the browser and the server:
```
Browser: turns off high spatial layer
SFU: transitions participants on that layer to a lower spatial layer (possibly after a timeout)
SFU: send a PLI to browser to be able to forward that layer
Browser: sends a key frame
```
This introduces a delay of at least one round trip time (<200ms) which is perceived
as a video freeze for the remote participants. Generating a key frame when turning off the
high spatial layer avoids this delay and improves the perceived video quality.

## Code sample
```
const sender = pc1.getSenders()[0];
const parameters = sender.getParameters();
parameters.encodings[1].active = false; // Disable upper spatial layers
parameters.encodings[2].active = false;
sender.setParameters(parameters, {encodingOptions:
  [{keyFrame: true}, {keyFrame: false}, {keyFrame: false}]});
```

## Design notes
There has been quite some back and forth on where this API should live and how it should work.
For example it has been proposed to have a dedicated `generateKeyFrame` method as part on the
RTCRtpSender object. As specified in
  https://w3c.github.io/webrtc-encoded-transform/#dom-rtcrtpscripttransformer-generatekeyframe
this does not take simulcast into account and creates a unneccessary dependency on the
Encoded Transform spec. It can also lead to race conditions where the separate calls to
`generateKeyFrame` and `setParameters` might end up yielding two key frames.

The need for a keyframe often coincides with other changes to the encoding parameters which
are controlled via the sender's `getParameters` and `setParameters` methods.
One particular example here is changing the scaleResolutionDownBy factor which affects the size
of the encoded frame which requires a key frame to be encoded.

Since requesting a key frame is a transient operation, adding
it to the encoding parameters which are returned by getParameters seemed inappropriate. The
way out of this was to add a second argument to setParameters.

See also the various working group meetings where this was discussed:
* https://www.w3.org/2023/05/16-webrtc-minutes.html#t04
* https://lists.w3.org/Archives/Public/www-archive/2023May/att-0000/WEBRTCWG-2023-05-16.pdf#page=25
* https://www.w3.org/2023/06/27-webrtc-minutes.html#t05
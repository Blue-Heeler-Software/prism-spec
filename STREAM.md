# Prism Stream

Prism Stream carries a live, real-time byte stream over a sequence of
[Prism Code](FORMAT.md) symbols. It is a payload protocol: it rides inside the
symbol payload and changes nothing about the symbol format. Audio is its first
and only defined profile, in section 5.

**Status: the transport is implemented and unit-tested, and has not yet been
verified end to end through a camera.** The frame format and the receiver logic
below are proven in software against injected loss; the latency and quality of a
real optical link are not yet measured.

---

## 1. What a stream is, and why it is not a transfer

[Aphotic](APHOTIC.md), the file-transfer protocol, delivers one exact object
eventually, taking as many laps as the losses demand. A live stream is the
opposite discipline. It wants whatever is ready **now**, on a playout clock, and
it would rather conceal a gap than wait for a retransmission that would arrive
too late to matter. A fountain peels a fixed object into completeness; a stream
is unbounded and disposable.

So Prism Stream does not use the fountain. It shares only the symbol layer
beneath it, which gives it one property worth stating plainly:

> A Prism symbol is Reed-Solomon protected and CRC-32 checked, so a decoded
> frame is **byte-exact or absent**; there is no such thing as a corrupted one.
> A stream therefore handles only **erasure**, never corruption. This is why the
> frame below carries no checksum of its own.

## 2. Loss, and the redundancy window

A stream survives lost frames with plain redundancy rather than an erasure code.
Each transmitted frame carries the newest run of packets, so a packet first sent
in one frame is sent again in the next several, newest first. A receiver that
loses a whole frame recovers its packets from any later frame that still carries
them, up to the window's depth.

The one honest cost is latency. All redundancy is of **past** data, because the
future has not been produced yet, so recovering a lost packet from a later
frame's copy means the receiver must not have played it yet. The recoverable
outage is therefore the smaller of two depths, the sender's redundancy window
and the receiver's playout buffer, and both cost delay:

```
recoverable outage  =  min(window depth, playout buffer depth)  frames
playout latency      =  playout buffer depth                     frames
```

Deeper recovery buys itself with latency, one for one. This is why Prism Stream
is a broadcast and push-to-talk medium and not a two-way call, quite apart from
the link being one-way.

## 3. Sequence numbers and sessions

Every packet has a 24-bit sequence number, monotonic within a session and
wrapping at 2^24. A **session** is a 16-bit identifier chosen once per stream. A
receiver seeing a new session id treats it as a new stream, from a different
source or a restart, and discards everything it held: the old buffer can never
continue it. A **codec change within one session** resets the receiver the same
way, since packets of the old and new codec cannot share a buffer.

Sequence numbers are not transmitted per packet. A frame states the sequence of
its **newest** packet in its header, and the packet at list position `i` carries
sequence `(headSequence - i) mod 2^24`. Packets are always fixed size for a
given codec, so no per-packet length is transmitted either.

## 4. Frame format

One Prism Stream frame is the byte payload of one symbol. All multi-byte fields
are big-endian.

| offset | size | field |
|--------|------|-------|
| 0..1   | 2 | magic, `0x50 0x56` (`"PV"`) |
| 2      | 1 | profile version, currently 1 |
| 3..4   | 2 | session identifier |
| 5      | 1 | codec identifier (high nibble) and flags (low nibble) |
| 6..8   | 3 | head sequence: the sequence of the newest packet, 24-bit |
| 9      | 1 | packet count `K` |
| 10..   |   | `K` codec packets, newest first, each of the codec's fixed size |

The packet count `K` MUST be in the range 1 to 200; the upper bound is a
defensive limit so a malformed count cannot make a reader allocate wildly. The
frame length MUST be exactly `10 + K * packetBytes`; a payload that does not
account for every byte is not a Prism Stream frame. A reader distinguishes a
stream frame from an [Aphotic](APHOTIC.md) transfer page and a plain document by
the magic: `PV` here, `PS` for a transfer page, neither for a document.

**Flags** (low nibble of byte 5):

| bit | meaning |
|-----|---------|
| 0 | silence: this run is comfort noise, the source paused |
| 1 | talkspurt: the first frames of a resumed burst after silence |

The talkspurt flag is a hint a receiver uses to reset codec state and show that
the source is active. It rides several consecutive frames rather than one, so a
single torn frame cannot swallow it.

## 5. The audio profile

The only codec identifiers defined so far carry speech. The transport never
looks inside a packet; it needs only the fixed packet size, to slice the frame,
and the frame duration, to reason about latency.

| id | codec | packet bytes | frame ms | note |
|---:|-------|-------------:|---------:|------|
| 0 | AMR narrowband, 4.75 kbit/s | 13 | 20 | the platform speech codec on every mobile handset; needs no separate build |
| 8 | Codec2 1300 | 7 | 40 | a fifth of the bytes of AMR, the robust target |
| 9 | Codec2 700C | 4 | 40 | maximum redundancy per byte |

Codec2's low absolute rate is what makes a deep redundancy window affordable:
the window costs bytes linearly in the codec rate, and a speech codec at 700 to
1300 bit/s leaves room for seconds of redundancy inside one symbol's payload.

> An audio-rate codec is not the only thing a Prism Stream could carry. The
> transport is codec-agnostic; a future profile could define a different codec
> table, or a variable-rate codec, at which point a per-packet length and an
> explicit timestamp, both omitted here because a fixed-rate codec needs
> neither, would be added under a new profile version.

## 6. Receiver behaviour (informative)

How a receiver turns frames back into a stream is a quality-of-implementation
matter, not part of the wire format. The reference receiver:

- holds a jitter buffer a fixed number of packets deep, which is the stream's
  latency, and begins playback once a full buffer has arrived;
- plays in sequence order, on the playout clock, concealing a missing packet
  rather than waiting for it;
- never rewinds: a packet arriving behind the playhead is too late and is
  dropped, not inserted;
- for a push-to-talk burst too brief to fill the buffer, offers an explicit
  flush that begins playback immediately with whatever depth is available;
- resyncs toward the intended latency if it falls too far behind, so buffer size
  and latency stay bounded under a fast source or a catch-up burst.

---

## Acknowledgements

The separation of a generic streaming layer from both the symbol format and the
audio profile follows a review of the format by **NomNomski**, who argued that
sequencing, loss handling, session identity and codec semantics are payload
concerns that must not touch the optical symbol. This document, and the
[Aphotic](APHOTIC.md) transfer protocol beside it, are that separation carried
out.

## License

This specification is licensed under the Creative Commons Attribution 4.0
International License (CC BY 4.0). SPDX-License-Identifier: CC-BY-4.0

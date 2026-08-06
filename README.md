# Prism Code

Prism Code is an open colour 2D barcode. It stacks several ordinary QR Codes
into the red, green and blue channels of one symbol, so a single code carries
several times the data of the black and white QR Code it is built from, while
staying locatable by an ordinary QR detector.

This repository is the **specification**, published for independent
implementation and review. The reference implementation lives elsewhere; nothing
here depends on it.

## The documents

- **[FORMAT.md](FORMAT.md)** is the normative specification: exactly what bytes
  and colours a valid symbol contains. Read this to build an encoder or a
  decoder.
- **[DECODER.md](DECODER.md)** describes the reference decoder. It is
  informative, not normative: an alternative decoder conforms if it recovers the
  payload the format defines, however it does so internally.
- **[BIT-LOADING.md](BIT-LOADING.md)** documents per-channel bit loading, a
  format extension that was implemented and then falsified on real hardware. It
  is kept so the dead end, and the measurements that closed it, are not lost.

Start with [FORMAT.md](FORMAT.md) section 0, which states plainly which
configurations are proven through a camera and which are only defined.

## Status

This is a working specification, not yet a frozen standard.

- Format version 2 at one bit per channel is proven on real handsets.
- Higher bit depths, and format version 3's per-channel loading, are defined but
  have not decoded from a camera. Section 0 of the specification is explicit
  about this.
- Canonical conformance vectors do not exist yet. Until they do, no
  implementation can demonstrate interoperability, and the specification says so.

Corrections and review are welcome through this repository's issues.

## About the name

Prism Code is an independent open format. Every plane of a symbol is a QR Code,
and **QR Code is a registered trademark of DENSO WAVE INCORPORATED**; this
project is not endorsed by or affiliated with the trademark holder, and
implementers are responsible for their own assessment of any intellectual
property that applies to QR Code encoding and decoding in their jurisdiction.

## License

This specification is licensed under the Creative Commons Attribution 4.0
International License (CC BY 4.0). You may share and adapt it, including for
commercial use, with attribution. See [LICENSE](LICENSE).

SPDX-License-Identifier: CC-BY-4.0

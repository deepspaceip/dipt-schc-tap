# dipt-schc-tap

An Ethernet-level header compressor based on SCHC (Static Context Header Compression,
[RFC 8724](https://www.rfc-editor.org/rfc/rfc8724)) that sits between two Linux TAP interfaces.

Frames leaving through one TAP interface have their IPv6 and UDP headers compressed before they are
forwarded to the other TAP interface. Frames arriving in the opposite direction are decompressed.
Running one `schc-tap` instance at each end of a link compresses the headers on that link without
changes to the applications or the IP stack on either host.

Important: this is a research prototype meant to experiment with header compression for IP in deep
space, not a production-ready tool.

Current version: 0.9.0

### Features

- **SCHC compression of IPv6 + UDP headers**: uses field descriptors, matching operators and
  compression/decompression actions as defined in RFC 8724
- **Address compression per netmask pair**: for each configured source/destination prefix pair, the
  prefix bits of both addresses are elided and only the remaining host bits are sent
- **Computed length fields**: the IPv6 payload length and UDP length are not sent; they are
  recomputed on decompression
- **Flow label**: elided when zero, sent otherwise
- **Pass-through**: frames that match no rule (e.g. IPv4, or IPv6 without UDP) are forwarded with
  a rule ID of 0 prepended to the payload and restored unchanged by the peer
- **Many rules**: rule IDs are encoded as LEB128 variable-length integers, so more than 255 rules
  are supported

### Build

Run `cargo build --release`. The `schc-tap` binary will be created under
`target/release/schc-tap`.

Run `cargo test` to execute the compression/decompression round-trip tests.

### Usage

```bash
schc-tap [OPTIONS] <IN_TAP_IFNAME> <OUT_TAP_IFNAME> [NETMASK_PAIRS]...
```

Arguments:

- `<IN_TAP_IFNAME>`: name of the TAP interface on the uncompressed side (created by the tool)
- `<OUT_TAP_IFNAME>`: name of the TAP interface on the compressed side (created by the tool)
- `[NETMASK_PAIRS]...`: zero or more `<source>/<len>,<destination>/<len>` IPv6 prefix pairs for
  which addresses are compressed, e.g. `2001:db8:1::/64,2001:db8:2::/64`

Options:

- `-s`, `--swap-netmask-pairs`: swap source and destination in each netmask pair. Use this on the
  peer, so that both ends can be given the same netmask pairs on the command line
- `-v`, `--verbose`: debug logging
- `-q`, `--quiet`: only log warnings and errors
- `-V`, `--version`: print the version and exit

Creating TAP interfaces requires root privileges (or `CAP_NET_ADMIN`).

Running the tool does the following:

1. Create the two TAP interfaces. The addresses are currently hardcoded: `10.10.0.22/24` and
   `fe80::22/64` on `<IN_TAP_IFNAME>`, `10.10.0.33/24` and `fe80::33/64` on `<OUT_TAP_IFNAME>`
2. Build the rule set: for each netmask pair, and then once more without address compression,
   one rule for UDP over IPv6 with a zero flow label and one with a non-zero flow label
3. Forward every Ethernet frame read from `<IN_TAP_IFNAME>` to `<OUT_TAP_IFNAME>` after compressing
   it (uplink)
4. Forward every Ethernet frame read from `<OUT_TAP_IFNAME>` to `<IN_TAP_IFNAME>` after
   decompressing it (downlink). A frame that carries a valid IPv6 packet is assumed not to be
   compressed and is forwarded as is
5. Log the frame size before and after each compression (`C`) or decompression (`D`)

Example, with two hosts A and B connected through the compressed side:

```bash
# On host A
sudo schc-tap tap-in tap-out 2001:db8:a::/64,2001:db8:b::/64

# On host B (same pairs, swapped)
sudo schc-tap --swap-netmask-pairs tap-in tap-out 2001:db8:a::/64,2001:db8:b::/64
```

Traffic must be routed into `<IN_TAP_IFNAME>`, and `<OUT_TAP_IFNAME>` must be bridged or otherwise
connected to the link toward the peer.

### Compressed frame format

The Ethernet header (14 bytes) is kept unchanged. The Ethernet payload is replaced by:

```
+-------------------+-----------------------+----------------------+
| Rule ID (LEB128)  | Compression residue   | Original payload     |
|                   | (bit-packed, padded)  | (remaining bytes)    |
+-------------------+-----------------------+----------------------+
```

A Rule ID of `0` means the frame was not compressed and the original Ethernet payload follows.
Rule `n` (for `n >= 1`) refers to the n-th rule in the rule set. Both ends must therefore run with
the same netmask pairs, in the same order, so that their rule sets match.

### Limitations

- Only IPv6 addresses are supported in netmask pairs; IPv4 traffic is passed through uncompressed
- Only a single IPv6 header followed by UDP is compressed (no extension headers)
- The TAP interface addresses are hardcoded
- No SCHC fragmentation

### License

Licensed under the Apache License, Version 2.0 ([LICENSE](LICENSE)).

### Acknowledgements

Developed by Adolfo Ochagavía. With special thanks to Marc Blanchet
([Viagénie inc.](https://www.viagenie.ca/)) for funding this work.

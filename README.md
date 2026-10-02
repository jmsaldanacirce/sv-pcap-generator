<!--Copyright (C) 2024 Savoir-faire Linux, Inc.
SPDX-License-Identifier: Apache-2.0 -->

# sv-pcap-generator

sv-pcap-generator is a tool used to generate IEC61850 Sample Values
PCAP (Packet Capture) files. You can then replicate IEC61850 SV trafic
on a network using tools such as `bittwist` or `tcpreplay`.

## Table of Contents

- [Introduction](#introduction)
- [Features](#features)
- [Installation](#installation)
- [Usage](#usage)
- [Release notes](#release-notes)

## Introduction

## Features

- Generation of SV PCAP files along various IEC61850 parameters
(frequency, appid, number of streams, etc)
- IEC61850 compliant pacing (250µs for 50Hz electrical network, 208µs
for 60Hz electrical network by default; configurable to other sample
rates, see `-c`/`--samples_per_cycle` below)
- Supports pcap loopback to make longer trafic generation

### Improvements July 2026

- New `-m/--nb_asdu` option: number of ASDUs (streams) bundled into a single frame. `nb_streams` is chunked into groups of this size (the last group can be smaller if it doesn't divide evenly), and each group becomes one frame.

- Dynamic BER length encoding: the original code hard-coded single-byte lengths (e.g. `0x6` + len(svIDFirst)`), which only worked because there was always exactly one ASDU. Bundling multiple ASDUs easily pushes lengths past 127 bytes, so a proper `ber_length()` helper has been added that emits short-form or long-form BER lengths as needed, used for every TLV (ASDU, SeqOfASDU, savPDU, SeqOfData).

- Frames are built bottom-up per iteration: `build_asdu()` → `build_savpdu()` → `build_sv_pdu()` → `Ethernet` frame, then wrapped in a correctly-sized pcap record header (also switched the pcap `incl_len/orig_len` to proper 4-byte little-endian fields instead of the single-byte-with-zero-padding trick, since frame sizes now regularly exceed 255 bytes).

- Samples are computed once per loop iteration and reused across all ASDUs/frames at that timestamp, matching the original's behavior (same waveform value regardless of stream).
Added validation: `nb_asdu` must be 1–255 (the `NumOfASDU` field is a single byte) and ≤ nb_streams.

### Improvements October 2026

- New `-c/--samples_per_cycle` option: decouples the SV frame rate from the
  simulated mains frequency. The sample rate (`sampling_rate`) is now
  `samples_per_cycle * frequency` instead of a hard-coded 80. The default
  of 80 reproduces the previous behaviour exactly (4000 Hz at `-f 50`,
  4800 Hz at `-f 60`). This lets you target other sample rates — e.g. the
  single-ASDU-per-frame, 96000 Hz profile sometimes used for instrument
  transformers — without the simulated waveform's own frequency drifting
  away from a real 50/60 Hz signal (which is what would happen if you
  just inflated `-f` on its own).

- New `--smpcnt_reset {second,cycle}` option. IEC 61850-9-2's `SmpCnt`
  field is an unsigned 16-bit integer (0-65535). The original script
  resets it once per second (`smpCnt = i % sampling_rate`), which only
  fits in 16 bits while `sampling_rate <= 65536`. At sample rates above
  that (96000 Hz, for instance) this silently overflowed. `--smpcnt_reset
  second` (the default) now refuses to run in that case rather than
  producing a malformed capture; `--smpcnt_reset cycle` resets `SmpCnt`
  once per mains cycle instead (`smpCnt = i % samples_per_cycle`), which
  stays comfortably inside 16 bits for any realistic `samples_per_cycle`
  value and is required to reach rates above 65536 Hz. Note this is a
  convention, not something every SV receiver assumes — check what your
  receiving IED/tool expects before relying on it.

- The `-l/--loop` option no longer has a fixed upper bound of 65536.
  The whole pcap is built in memory before being written to disk, at
  roughly `(frame_len + 16)` bytes per frame per loop iteration, so the
  practical limit is now your available memory/disk rather than an
  arbitrary cap. A lower bound of 1 still applies.

## Installation

### Requirements

Following Python packages are needed:

```
pip install numpy
```

To run merge_pcap script, `wireshark` package is needed.

## Usage

### Example 1

To generate a IEC61850 SV pcap on 8 streams, with 4000 SV for each
streams, for a 50Hz electrical network, run:

```
python3 generate_pcap.py -n 8 -l 4000 -f 50 output.pcap
```

### Example 2

```
python3 gen_sv_pcap.py -n 96 -m 6 -l 4000 -f 60 sv_6asdu.pcap
```

Breakdown of the flags:

`-n 96` — total number of streams (96 here just so it divides evenly into groups of 6; use whatever you actually need) `-m 6` — bundle 6 ASDUs per frame, so this produces 96 / 6 = 16 frames per loop iteration `-l 4000` — 4000 loop iterations (default) `-f 60` — 60 Hz sampling frequency (default) `sv_6asdu.pcap` — output file

If you want to keep all your other defaults (`start_id`, `svID` prefix/digits, RMS values, MAC addresses, `VLAN`, etc.), you can just add `-m 6` to whatever command you were already running, e.g.:

```
python3 gen_sv_pcap.py -n 64 -m 6 output.pcap
```

Note `64` isn't evenly divisible by 6, so this gives you ten frames of 6 ASDUs plus a final frame of 4 ASDUs per loop iteration — the script handles that remainder automatically rather than erroring out.

### Example 3: 96000 Hz, 1 ASDU per frame (instrument transformer profile)

By default `samples_per_cycle` is 80, so the sample rate is tied to the
mains frequency as 80 × frequency (4000 Hz at 50 Hz, 4800 Hz at 60 Hz).
To reach a different sample rate — such as the 96000 Hz, one-ASDU-per-frame
profile used for instrument transformers — set `-c/--samples_per_cycle`
so that `samples_per_cycle * frequency` equals the target rate, and switch
`SmpCnt` to reset once per cycle instead of once per second (required
above 65536 Hz, see Improvements above):

```
# 50 Hz network: 1920 samples/cycle x 50 Hz = 96000 Hz
python3 generate_pcap.py -n 1 -m 1 -f 50 -c 1920 --smpcnt_reset cycle -l 192000 instrument_transformer_96kHz.pcap

# 60 Hz network: 1600 samples/cycle x 60 Hz = 96000 Hz
python3 generate_pcap.py -n 1 -m 1 -f 60 -c 1600 --smpcnt_reset cycle -l 192000 instrument_transformer_96kHz.pcap
```

`-m 1` is actually the default and can be omitted — it is shown above only
to make the "1 ASDU per frame" requirement explicit. `-l 192000` above
generates 2 seconds of traffic at 96000 Hz; adjust to taste (no fixed
upper limit any more, see Improvements above).

Optionally, you can run `merge_sv_pcap.py` script to merged multiple SV pcap
file. This is useful to generate discontinuity to test electrical lines
protections.

To do so, run:

```
./merge_sv_pcap.py 1.pcap 2.pcap 3.pcap -o merged.pcap -f 50
```

The `merge_sv_pcap.py` can be also used to repeat a pcap multiple time with `-n` argument

```
./merge_sv_pcap.py my_sv_recored.pcap -o 10_interations.pcap -n 10 -f 50
```

In both case a delay between the merged pcap is inserted base on the current
frequency.

## Release notes

### Version v0.1

- Initial release

### Version v1.0.0

- Rewrite merge_pcap in Python
- Improve merge_pcap to support multiple pcap with different duration
- Add VLAN ID, Priority and MAC addresses options

### Version v1.0.1

July 2026 improved by Jose Saldana at CIRCE Technology center, within the framework of the Horizon Europe project ESTELAR (Grant Agreement No. 101192574).

Additions:

- number of ASDUs option.
- Dynamic BER length encoding.
- Frames are built bottom-up per iteration.
- Samples are computed once per loop iteration and reused across all ASDUs/frames at that timestamp.

### Version v1.0.2

October 2026 improved by Jose Saldana at CIRCE Technology center, within the framework of the Horizon Europe project ESTELAR (Grant Agreement No. 101192574).

Additions:

- `-c/--samples_per_cycle` option to decouple the SV frame rate from the
  simulated mains frequency, enabling sample rates other than 4000/4800 Hz
  (e.g. 96000 Hz for instrument transformer profiles).
- `--smpcnt_reset {second,cycle}` option, with validation to prevent
  `SmpCnt`'s 16-bit field from silently overflowing at sample rates above
  65536 Hz.
- Removed the fixed 65536 upper limit on `-l/--loop`; only a lower bound
  of 1 remains, so capture length is now limited by available
  memory/disk rather than an arbitrary cap.

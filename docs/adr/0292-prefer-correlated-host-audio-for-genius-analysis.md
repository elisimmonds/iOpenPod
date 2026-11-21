# ADR-0292: Prefer correlated Host audio for Genius analysis

- Status: Accepted
- Date: 2026-11-19
- Amends: ADR-0179 and ADR-0274

## Context

Cold Genius analysis of retained music remains limited by iPod USB reads even
after serializing device reads and overlapping profiling. The same recordings
often remain available on the Host. A path hint alone is insufficient: the Host
file may have changed since Sync or the iPod may contain another recording.

## Decision

Prepared incoming media remains authoritative for new and replaced Tracks. For
retained Tracks, reuse a valid device-bound profile first. Otherwise prefer a
Host counterpart established by unchanged committed Sync Details or the current
reviewed Sync comparison. Validate persistent Track identity, device path, size
and filesystem-aware modification time, plus the Host path, size and modification
time. Observe and read Host files through Storage without following links. No
title/artist matching, new media scan, or device-audio hashing is required.

Use the Host format when decoding: a FLAC original may correspond to device AAC.
Correlated Host audio uses the same two-to-six-worker pool as staged media, outside
the device-read restriction. Missing, stale, unsupported, or unreadable initial
Host sources fall back to the device. Optional Host decoding failure rejoins the
bounded device queue; cancellation, disconnect, and source mutation still stop
preparation. Stream-selection digests cannot recover a chosen stream, so those
correlations retain the prepared/device source rather than guessing.

Cache original-Host profiles under their own observed file identity and analysis
version. Do not label those profiles as analyses of byte-identical device media:
lossy conversion can change measurements. Both Host and device observations remain
dependencies through publication. Correlation permits analysis only; it cannot
authorize a media replacement, retagging, or a new committed Sync association.

## Consequences

Mix creation, Genius refresh, full Sync, and missing-BPM computation share this
selection policy. Current reviewed correlations work before their first commit;
outside Sync, only the committed helper is consulted. A valid warm profile still
avoids decoding. Physical speedup depends on Host availability and compute cost;
virtual tests verify the selected transport, six-worker admission, cache reuse,
fallback, and publication-time source checks without claiming hardware timings.

## Amendment: key correlated Host profiles by persistent facts (2026-11-20)

A correlated Host profile's cache key uses the Host path, size, and modification
time. It omits `st_dev` and `st_ino`. Network filesystems receive a new device
number when mounted (a Linux CIFS share is an anonymous `0:N` device allocated in
mount order), and not every server keeps stable file IDs, so keys that included
them could discard every correlated profile after a remount or restart and reread
the whole Host library. ADR-0179's device-profile amendment made the same choice
for FAT iPods.

Within one analysis, the observed Host file is still validated against its
complete identity, including device and inode, before and after decoding and
again through publication. Path, size, and modification time are the same facts
the correlation itself requires, so the key does not accept evidence the
correlation would reject. Existing identity-bound entries become unused and age
out under the normal bounds.

# Findings: WebRTC absolute timestamp offset with useAbsoluteTimestamp

## 1) Problem

### Summary
When useAbsoluteTimestamp is enabled and a stream is ingested via WebRTC and then read out again via WebRTC, absolute timestamps can be shifted by an apparently random offset.

### Observed behavior
- Absolute time reported downstream is available, but offset from the expected value.
- Offset appears random between runs/sessions.
- This is most visible in audio paths that rewrite RTP timestamps before output.

### Root cause
In some WebRTC egress audio branches, RTP timestamps are intentionally regenerated from a local timeline (curTimestamp), but NTP was being derived using a timestamp base from the incoming timeline.

That mixed two different RTP time domains in the same subtraction:
- outgoing regenerated RTP timestamp
- incoming original RTP base timestamp

As a result, sender report NTP↔RTP mapping could contain an arbitrary offset even though mapping was present.

### Why the existing guard did not prevent it
The warning "received RTP packet without absolute time, skipping it" protects only the case where absolute mapping is unavailable.

In this bug, mapping was available but incorrect, so packets were accepted and forwarded with wrong absolute-time alignment.

### Relevant code areas
- internal/protocols/webrtc/from_stream.go
- internal/protocols/webrtc/outgoing_track.go
- internal/protocols/webrtc/incoming_track.go
- internal/protocols/webrtc/to_stream.go

## 2) Fix

### What was changed
A consistent timestamp-domain conversion was enforced for rewritten RTP paths:

- Added helper:
  - timestampToNTP(baseNTP, baseTimestamp, timestamp, clockRate)
- In rewritten-audio branches, NTP is now computed from:
  - baseTimestamp := curTimestamp (snapshot of outgoing RTP base for the unit)
  - pkt.Timestamp (outgoing regenerated RTP timestamp)

So conversion now uses one domain:
- ntp = baseNTP + (outgoingRTP - outgoingRTPBase) / clockRate

### Affected branches
- Opus rewritten timestamps
- G711 8k rewritten timestamps
- G711 non-8k re-encode path
- LPCM re-encode path

### Why this fixes it
RTCP sender reports are generated from the outgoing RTP timeline and provided NTP. Using matching outgoing RTP deltas for NTP conversion keeps SR mapping coherent and removes random offsets.

### Validation
- Existing focused test for resampling path passes.
- Added regression-style test for timestamp base semantics:
  - verifies conversion with regenerated base
  - verifies mismatch if incoming base is used

## Upstream Issue Draft

### Title
WebRTC loopback with useAbsoluteTimestamp can produce random absolute-time offset due to mixed RTP timestamp domains

### Description
When publishing via WebRTC and reading via WebRTC with useAbsoluteTimestamp enabled, absolute timestamps can be shifted by a random offset.

Root cause appears to be mixed RTP domains during NTP conversion in WebRTC egress audio branches that rewrite RTP timestamps:
- outgoing regenerated RTP timestamp was compared against incoming RTP base timestamp
- this can generate incorrect SR NTP↔RTP mapping while still reporting mapping as available

### Steps to reproduce
1. Configure a path with useAbsoluteTimestamp enabled.
2. Publish audio via WebRTC.
3. Read the same path via WebRTC.
4. Inspect absolute timestamps (or SR-derived playout timestamps).

### Expected behavior
Absolute timestamps remain aligned through WebRTC in/out loopback.

### Actual behavior
Absolute timestamps are available but shifted by an apparently random offset.

### Scope
Observed on audio branches that regenerate RTP timestamps for browser compatibility.

### Suspected files
- internal/protocols/webrtc/from_stream.go
- internal/protocols/webrtc/outgoing_track.go
- internal/protocols/webrtc/incoming_track.go

## Pull Request Description Draft

### Summary
Fix WebRTC absolute timestamp offset when useAbsoluteTimestamp is enabled by using a consistent RTP timestamp domain during NTP conversion in rewritten-audio egress paths.

### Problem
Some WebRTC egress audio paths regenerate RTP timestamps but previously computed NTP deltas against incoming RTP base timestamps. This mixed timestamp domains and could produce random absolute-time offsets in RTCP sender report mapping.

### Solution
- Add timestampToNTP helper to centralize conversion.
- In rewritten audio branches, compute NTP using:
  - baseTimestamp captured from curTimestamp (outgoing timeline)
  - current pkt.Timestamp (same outgoing timeline)
- Keep conversion formula coherent with outgoing SR generation.

### Files changed
- internal/protocols/webrtc/from_stream.go
- internal/protocols/webrtc/from_stream_test.go

### Testing
- Focused WebRTC package tests executed successfully.
- Added regression test covering timestamp-base semantics for rewritten timelines.

### Impact
- Fixes random absolute-time offsets in WebRTC in→out loopback with useAbsoluteTimestamp enabled.
- No behavior change intended for paths that do not rewrite RTP timestamps.

### Notes
The existing "received RTP packet without absolute time, skipping it" warning remains correct for unavailable mapping; this change addresses incorrect-but-available mapping.
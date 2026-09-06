# Part 3 network use cases -> requirements

`mixboard.txt` Part 3 covers use beyond one desktop, in two chapters:

- **"IP streams, cameras, multi-room NetAudio, VPN and recipes"** — RTSP/IP-camera ingest, multi-room over Wi-Fi, VPN.
- **"Bluetooth, 2.4 GHz dongles, Wi-Fi bands, roaming and clocks"** — Bluetooth, 2.4 GHz dongles, Wi-Fi roaming, clock drift.

This document turns those headings into requirements for the `network-ext` area and records the 12 matrix rows (`F-0395` to `F-0406`, `T-network-ext-011` to `T-network-ext-022`) this plan adds to `feature-test-matrix.csv` to cover them, continuing the 10 rows (`F-0295`-`F-0301`, `F-0358`-`F-0360`, `T-network-ext-001` to `010`) already present for the sync transport and basic multi-room grouping. All new rows use milestone `M7`, matching the existing `network-ext` rows.

## Chapter 1: IP streams, cameras, multi-room NetAudio, VPN and recipes

### Heading 1 — "Can Mixboard ingest RTSP/RTP/HTTP/HLS/Icecast? No."

The reference documents this as a hard gap: the incumbent cannot ingest any of these. Per decision spine #7 (fix defects by design) and the general goal of a superset feature set, OPENMIX closes this gap rather than reproducing it.

**Requirements derived**:
- An input source type that decodes an RTSP/RTP stream (as an IP camera publishes) into a virtual input strip. -> `F-0395` / `T-network-ext-011`.
- An input source type that decodes HLS or Icecast/SHOUTcast into a virtual input strip. -> `F-0397` / `T-network-ext-013`.

### Heading 2 — IP-camera audio into Mixboard and Mixboard audio to a camera speaker

The reference's recipe chapter covers both directions: pulling a camera's microphone audio in, and pushing operator audio out to a camera's speaker (a "talk back" backchannel), typically over ONVIF Profile T / RTSP backchannel.

**Requirements derived**:
- Backchannel output: route a bus to an IP camera's speaker over RTSP/ONVIF backchannel. -> `F-0396` / `T-network-ext-012`.
- Since RTSP/IP-camera sources will, in practice, occasionally drop (camera reboot, network blip), and decision spine #7 requires auto-reconnect as a general defect fix, that fix explicitly extends to these new network ingest sources rather than being scoped only to local hardware devices. -> `F-0398` / `T-network-ext-014`.

### Heading 3 — Multi-room over Wi-Fi with NetAudio

Already substantially covered by the existing rows (`T-network-ext-005` stream grouping, `T-network-ext-006` clock-sync, `T-network-ext-008` simultaneous plain/sync streams). The reference's recipe chapter additionally implies rooms are joined and left as a normal part of use (someone brings a new speaker online, someone turns one off), not only configured once at setup.

**Requirement derived**:
- A receiver can join or leave a running stream group at runtime without restarting or reconfiguring the group's other members. -> `F-0399` / `T-network-ext-015`.

### Heading 4 — NetAudio over the internet / VPN

The reference documents VPN as the recommended way to run NetAudio beyond a LAN, since NetAudio's discovery mechanisms assume LAN broadcast. A routed VPN tunnel (e.g. WireGuard) does not carry broadcast traffic, so operation over such a tunnel depends on explicit unicast endpoint configuration working end to end in both the plain and sync transports.

**Requirement derived**:
- Both transports operate correctly over a routed point-to-point VPN link using only explicit IP:port configuration, with no dependency on broadcast discovery. -> `F-0400` / `T-network-ext-016`.

### Heading 5 — Non-Windows endpoints and automation control

The reference notes that non-Windows endpoints exist in these deployments (a Linux or embedded receiver in a multi-room setup) and are driven through automation/scripting rather than a native GUI on that endpoint. This is the network-use-case argument for why the control protocol has to work standalone on any platform, without a GUI dependency — the requirement is captured here for its network-topology motivation, even though the control protocol itself lives in the `control-api` area.

**Requirement derived**:
- A non-Windows/headless client can join a multi-room group and be fully controlled (membership, volume, mute) purely over the control protocol. -> `F-0401` / `T-network-ext-017`.

### Heading 6 — Practical recipes

The recipes chapter is where the concrete acceptance-test shapes above were drawn from (RTSP ingest via a sidecar, VPN tunnel configuration, multi-room grouping); it does not introduce a requirement of its own beyond what headings 2-5 already cover.

### Heading 7 — Latency figures collected (with source)

Informs the performance budgets in `03-test-kit.md` rather than a new functional requirement: the reference's collected figures (5-20 ms small-buffer NetAudio, 100+ ms large-buffer) are the baseline the sync-transport's alignment tolerance (`T-network-ext-002`) is measured as "materially better than."

## Chapter 2: Bluetooth, 2.4 GHz dongles, Wi-Fi bands, roaming and clocks

### Heading 1 — Facts that govern every wireless scenario

Background/context; no new requirement beyond confirming that wireless endpoints are first-class citizens the `network-ext` area must handle, not an edge case bolted onto the wired device model.

### Heading 2 — Bluetooth on Windows 10/11 with Mixboard

The reference documents Bluetooth as usable but with real latency and reliability caveats (profile switching, dropouts) that the incumbent does not clearly surface to the user.

**Requirement derived**:
- A bus can be routed to a Bluetooth A2DP output; the actual measured latency is disclosed in the UI rather than hidden; the endpoint auto-reconnects and resumes routing after a radio drop. -> `F-0402` / `T-network-ext-018`.

### Heading 3 — 2.4 GHz proprietary USB dongles, DECT, USB wireless mic receivers

These devices run their own, independent clock, asynchronous to the engine's main clock, and share spectrum with Wi-Fi, which the reference flags as a source of interference-driven glitches.

**Requirements derived**:
- An input strip fed by such a device is automatically delay- and clock-compensated against the engine's main clock so it stays sample-aligned with wired strips. -> `F-0403` / `T-network-ext-019`.
- A diagnostic surfaces rising packet-loss/retry rates consistent with 2.4 GHz band congestion and suggests a corrective action, rather than leaving the operator to guess at the cause of a glitch. -> `F-0406` / `T-network-ext-022`.

### Heading 4 — Wi-Fi bands for NetAudio/NetAudio Receptor/AirPlay while moving

Covers band selection (2.4 GHz vs 5 GHz) trade-offs while a device is mobile; informs the roaming requirement in the next heading rather than adding a distinct one of its own.

### Heading 5 — Roaming scenarios

The reference describes a person moving between rooms/access points while a NetAudio session is active, and the incumbent generally requiring a manual reconnect after a roam.

**Requirement derived**:
- A NetAudio/sync-transport client roaming between access points on the same SSID keeps its session alive across the handoff, reconnecting automatically within a documented bound, with the event logged. -> `F-0404` / `T-network-ext-020`.

### Heading 6 — Mixing devices with different sample rates/clocks; USB bandwidth

Distinguishes a one-time sample-rate mismatch (handled by ordinary SRC, already covered in `engine`/`netaudio`) from a slow clock drift accumulating over a long session between two nominally-matched devices with slightly different crystals — the latter is a network-endpoint-specific concern this area needs to own.

**Requirement derived**:
- A network endpoint's clock drift relative to the sender is continuously tracked and compensated over long sessions, distinct from one-time per-stream SRC, so no clicks or dropouts accumulate. -> `F-0405` / `T-network-ext-021`.

### Heading 7 — Alternatives and companions

Background on other tools used alongside the incumbent in these scenarios (Snapcast, Jamulus, etc.); informs `07-research-notes.md`'s prior-art list rather than adding a new requirement — OPENMIX's sync transport already targets the alignment quality these tools are chosen for.

### Heading 8 — Quick decision rules derived from the above

The reference's own summary heuristics (when to use Bluetooth vs Wi-Fi vs a wired dongle, when to suspect band congestion) are exactly what the congestion diagnostic (`F-0406`) surfaces automatically instead of leaving as a manual decision tree for the operator to apply.

## Rows added to the matrix

| id | test_id | feature | owner | milestone |
|---|---|---|---|---|
| F-0395 | T-network-ext-011 | RTSP/RTP audio ingest from IP cameras | opus+flash | M7 |
| F-0396 | T-network-ext-012 | Two-way IP-camera audio: backchannel to camera speaker | opus | M7 |
| F-0397 | T-network-ext-013 | HLS/Icecast internet-radio ingest as an input source | opus+flash | M7 |
| F-0398 | T-network-ext-014 | Automatic reconnect for dropped IP-camera/HLS/Icecast sources | opus | M7 |
| F-0399 | T-network-ext-015 | Multi-room dynamic join/leave without restart | opus | M7 |
| F-0400 | T-network-ext-016 | Sync/plain transport operation over a routed VPN tunnel | opus | M7 |
| F-0401 | T-network-ext-017 | Non-Windows/headless endpoint joins a multi-room group via the control protocol | opus+flash | M7 |
| F-0402 | T-network-ext-018 | Bluetooth output endpoint with disclosed latency and auto-reconnect | opus+flash | M7 |
| F-0403 | T-network-ext-019 | 2.4 GHz wireless-dongle input latency/clock compensation | opus | M7 |
| F-0404 | T-network-ext-020 | Wi-Fi roaming handoff for NetAudio clients | opus | M7 |
| F-0405 | T-network-ext-021 | Long-session clock-drift reconciliation for network endpoints | opus | M7 |
| F-0406 | T-network-ext-022 | 2.4 GHz band congestion diagnostic | flash | M7 |

These 12 rows bring `feature-test-matrix.csv` from 394 to 406 rows total, and `network-ext` from 10 to 22 rows. IDs and test IDs continue the existing numbering with no gaps or reuse; the CSV's header and column order are unchanged.

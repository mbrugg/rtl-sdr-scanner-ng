# SDR Scanner NG Project Description

## 1. Project Goal

The project aims to build a scalable SDR-based spectrum monitoring system that can continuously receive wideband I/Q data, detect unknown transmissions, estimate their center frequency and bandwidth, and record selected narrowband signals with pre-trigger context.

The system is intended to support multiple SDR/baseband sources and eventually use a Zynq-7020 platform to reduce host-side compute and bandwidth load by moving suitable streaming DSP tasks closer to the I/Q source.

The primary objective is not to record all raw wideband I/Q continuously. Instead, the system should convert a continuous wideband stream into:

- signal events and metadata,
- selected narrowband I/Q recordings,
- optional demodulated audio or decoded payloads,
- long-term searchable transmission records.

## 2. Core Design Principles

### 2.1 Detect First, Extract Later

The system should not assume that all transmissions are known in advance. Center frequency, bandwidth, duration, and modulation may be unknown.

The basic processing model is:

```text
Wideband I/Q
   ↓
wideband or coarse-band detector
   ↓
estimated emission object
   ↓
dynamic narrowband extraction
   ↓
recording, demodulation, classification, storage
```

A detected emission should be represented as structured metadata:

```text
Emission
  - center frequency
  - lower and upper spectral edge
  - estimated bandwidth
  - signal power / SNR estimate
  - first seen timestamp
  - last seen timestamp
  - confidence/state
```

### 2.2 Stable Flowgraph, Dynamic Control

The preferred design is to keep one stable processing pipeline per SDR source and control it through events/messages instead of starting a new heavy processing graph for every signal.

This avoids unnecessary overhead from:

- repeated graph creation/destruction,
- duplicated full-band I/Q fan-out,
- multiple independent filters operating on the same wideband stream,
- unnecessary transport between local blocks.

### 2.3 Pre-Trigger Capture

Signal detection always introduces some delay. To avoid losing the beginning of a transmission, the system should maintain a rolling I/Q ring buffer.

Example:

```text
Detector confirms signal at t = 120 ms
Recorder includes samples from t = -500 ms onward
```

This allows short or bursty transmissions to be captured more completely.

## 3. Recommended Host-Side Reference Architecture

The first implementation should run mainly on the host. This keeps algorithm development fast and avoids premature FPGA complexity.

```text
┌──────────────────────────── Host Scanner System ────────────────────────────┐
│                                                                              │
│  SDR / Zynq I/Q input                                                         │
│       ↓                                                                      │
│  wideband I/Q ring buffer                                                     │
│       ├───────────────────────────────┐                                      │
│       │                               │                                      │
│       ↓                               ↓                                      │
│  FFT / PSD detector              dynamic extractor                           │
│       ↓                               ↑                                      │
│  noise floor estimator                │                                      │
│       ↓                               │                                      │
│  spectral island detector             │                                      │
│       ↓                               │                                      │
│  emission tracker ───────────────→ start/update/stop events                  │
│                                       │                                      │
│                                       ↓                                      │
│                            narrowband I/Q recorder                           │
│                                       ↓                                      │
│                       demodulation / classification / storage                 │
└──────────────────────────────────────────────────────────────────────────────┘
```

The host reference design should validate:

- detector behavior,
- bandwidth estimation,
- emission tracking,
- recorder start/stop logic,
- ring-buffered pre-trigger capture,
- storage and metadata format,
- later demodulation and classification stages.

## 4. GNU Radio Role

GNU Radio can be used as the streaming DSP engine while custom C++ logic provides scanner intelligence.

Recommended split inside a GNU Radio-based implementation:

```text
GNU Radio handles:
  - SDR source integration
  - sample streaming
  - FFT / PSD generation
  - filtering and resampling
  - optional channelization
  - custom real-time detector blocks

Application logic handles:
  - emission tracking
  - recorder state machines
  - metadata handling
  - database and UI integration
  - modulation-classifier decisions
  - long-term storage
```

Custom GNU Radio blocks may be implemented for:

- wideband detector,
- noise floor estimation,
- spectral island detection,
- I/Q ring buffer,
- dynamic DDC extraction,
- recorder sink,
- event/message bridge to the host application.

The recommended implementation style is one stable GNU Radio top block per SDR source, controlled by messages or events.

## 5. Unknown Frequencies and Variable Bandwidths

Because transmissions may appear at arbitrary frequencies and may have different occupied bandwidths, a rigid fixed-channel recorder is not sufficient.

Supported target bandwidths should include at least:

- 6.25 kHz,
- 12.5 kHz,
- 25 kHz,
- wider user-defined bandwidths.

A practical design is a hybrid detector/extractor system:

```text
Wideband I/Q
   ↓
FFT / PSD detector
   ↓
find active spectral islands
   ↓
estimate center frequency and occupied bandwidth
   ↓
select suitable recorder bandwidth
   ↓
fine DDC extraction from ring buffer or live stream
```

A later coarse channelizer can be added to reduce processing load:

```text
2 MHz baseband
   ↓
coarse channelizer
   ├─ 100 kHz subband 0
   ├─ 100 kHz subband 1
   ├─ 100 kHz subband 2
   └─ ...
```

Each coarse subband can then run lighter detection and extraction logic.

## 6. Zynq-7020 Offload Strategy

The Zynq-7020 should not initially replace the complete scanner. It should be used to reduce continuous high-rate data movement and repetitive DSP load.

### 6.1 Recommended Split

A sensible division of responsibilities is:

```text
Zynq FPGA fabric:
  - sample conditioning
  - optional decimation
  - optional FFT/PSD engine
  - optional coarse channelizer
  - optional DDC workers

Zynq ARM cores:
  - configure FPGA blocks
  - manage DMA/ring buffer
  - packetize metadata
  - handle network transport
  - lightweight signal tracker if needed

Host PC/server:
  - global scheduler
  - database
  - UI
  - long-term recordings
  - heavier classification
  - demodulation experiments
  - correlation across multiple SDRs
```

### 6.2 Best Target Architecture

```text
          ┌──────────────────────────────┐
          │            Zynq              │
          │                              │
IQ input →│ sample cleanup / decimation  │
          │        ↓                     │
          │ ring buffer in DDR           │
          │        ↓                     │
          │ FFT / PSD / detector         │
          │        ↓                     │
          │ events + optional DDC output │
          └───────────┬──────────────────┘
                      │
                      ↓
              Host scanner system
          metadata + narrowband IQ
```

This architecture is preferable to continuously sending full raw I/Q to the host if the bottleneck is host CPU, network bandwidth, or storage.

### 6.3 Best Offload Candidates

The highest-value Zynq offload candidates are:

1. sample conditioning and optional decimation,
2. ring buffer with pre-trigger capture,
3. FFT / PSD / noise-floor detection,
4. a limited number of DDC extractors for active signals,
5. coarse channelization once many simultaneous signals must be handled.

Application-level functions should remain on the host:

- UI,
- database,
- file management,
- long-term storage,
- scheduler logic,
- complex classifiers under active development,
- experimental demodulation chains.

## 7. When Zynq Offload Is Worth It

Offloading is worth considering if at least one of the following is true:

- multiple SDRs are used and the host becomes CPU-bound,
- the link from Zynq to host is bandwidth-limited,
- continuous full-band I/Q should not be streamed or stored,
- reliable pre-trigger capture is required,
- low-power standalone operation is desired,
- detection must run continuously 24/7.

Offloading may not be worth the effort if:

- only one 2 MHz RTL-SDR-class stream is processed,
- a normal PC can already handle the full pipeline,
- detection and extraction algorithms are still changing frequently,
- the detector/channelizer design is not yet validated,
- FPGA development time is the main project bottleneck.

## 8. Project Milestones

### Milestone 1 — Host Reference Scanner

Goal: prove the scanner concept without FPGA complexity.

Scope:

- Zynq or SDR sends full I/Q to host.
- Host performs FFT/PSD detection.
- Host maintains I/Q ring buffer.
- Host estimates center frequency and bandwidth.
- Host performs dynamic fine extraction.
- Host records narrowband I/Q with pre-trigger samples.

Exit criteria:

- stable detection of transmissions in a wideband stream,
- reliable start/stop event generation,
- narrowband recordings include pre-trigger context,
- metadata is stored with each recording.

### Milestone 2 — Zynq-Side Ring Buffer

Goal: move pre-trigger buffering closer to the I/Q source.

Scope:

- implement DDR-backed ring buffer on or near Zynq,
- keep host-side detector and extraction logic,
- allow host to request time windows or pre-trigger snippets,
- maintain accurate sample counters/timestamps.

Exit criteria:

- host can request historical I/Q around a detection event,
- sample timing is consistent,
- no loss of short transmission starts under normal load.

### Milestone 3 — Zynq FFT/PSD Detection

Goal: reduce continuous data rate and host CPU usage.

Scope:

- move FFT/PSD and noise-floor estimation to Zynq,
- emit compact signal events to host,
- optionally send requested I/Q snippets from ring buffer,
- host remains responsible for final extraction and storage decisions.

Exit criteria:

- Zynq emits useful signal metadata,
- host no longer needs full-rate I/Q continuously for detection,
- detection quality is comparable to the host reference implementation.

### Milestone 4 — Zynq DDC Workers

Goal: send narrowband I/Q instead of full-band I/Q for active transmissions.

Scope:

- implement a small number of configurable DDC workers,
- each worker performs frequency shift, filtering, and resampling,
- host configures workers based on detected emissions,
- host receives narrowband I/Q plus metadata.

Exit criteria:

- active transmissions can be extracted on Zynq,
- host bandwidth is reduced substantially,
- multiple simultaneous narrowband recordings are supported within resource limits.

### Milestone 5 — Coarse Channelizer Evaluation

Goal: improve scalability for dense signal environments.

Scope:

- evaluate coarse channelization, for example 50–200 kHz subbands,
- compare resource usage against independent DDC workers,
- route active subbands or emissions to host or DDC workers,
- decide whether full FPGA channelization is justified.

Exit criteria:

- measured benefit for many simultaneous signals,
- clear decision on whether to integrate channelization permanently,
- documented limits for bandwidth, number of channels, and resource usage.

## 9. Development Priorities

The recommended development order is:

```text
1. Get the scanner algorithms correct on the host.
2. Add robust pre-trigger capture with a ring buffer.
3. Validate metadata and recording format.
4. Move buffering closer to the source.
5. Move detection to Zynq only after the detector is stable.
6. Add FPGA DDC workers for active signals.
7. Add coarse channelization only when measurements justify it.
```

This keeps the project flexible while the detection and classification approach is still evolving.

## 10. Summary

The project should evolve from a host-based SDR scanner into a hybrid Zynq/host monitoring system.

The host should remain responsible for high-level intelligence, storage, UI, and experimentation. The Zynq should gradually take over deterministic streaming tasks that reduce continuous data movement and CPU load.

The biggest expected gain is not from moving the entire application into the FPGA. The main gain is to transform continuous wideband I/Q into compact signal events and selected narrowband I/Q streams before the data reaches the host.

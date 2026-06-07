# SDR Scanner NG

`rtl-sdr-scanner-ng` is an experimental next-generation SDR scanner project derived from and inspired by [`shajen/rtl-sdr-scanner-cpp`](https://github.com/shajen/rtl-sdr-scanner-cpp), [`shajen/sdr-hub`](https://github.com/shajen/sdr-hub), and [`shajen/sdr-monitor`](https://github.com/shajen/sdr-monitor).

The original projects provide a strong proof of concept: wideband SDR scanning, automatic transmission detection, simultaneous recording of multiple transmissions inside one received band, and integration with a monitor/web stack. This fork explores a more scalable scanner architecture for unknown signals, pre-trigger capture, multi-SDR operation, and optional Zynq offload.

## Status

This repository is currently a design and prototyping fork. The long-term goal is not to simply polish the existing scanner, but to evaluate a new scanner core that can scale better when many transmissions or multiple SDR/baseband sources are active.

The first implementation target is a host-based reference scanner. FPGA/Zynq acceleration should only be added after the host-side algorithms and data model are validated.

## Goals

- Detect unknown transmissions in a wideband I/Q stream.
- Estimate center frequency, occupied bandwidth, power/SNR, and duration.
- Record selected narrowband I/Q with pre-trigger context.
- Support variable bandwidths such as 6.25 kHz, 12.5 kHz, 25 kHz, and wider user-defined signals.
- Support multiple SDR/baseband sources.
- Keep one stable processing pipeline per SDR source where possible.
- Use GNU Radio and/or custom C++ DSP blocks for streaming signal processing.
- Evaluate Zynq-7020 offload for buffering, detection, and selected narrowband extraction.

## Architecture Direction

The preferred processing model is:

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

The project should avoid starting heavy independent processing graphs for every detected signal when a shared pipeline or shared ring buffer can be used instead.

For the detailed design, milestones, and Zynq/host split, see:

- [`docs/project-description.md`](docs/project-description.md)

## Recommended Milestone Path

1. **Host Reference Scanner** — Zynq or SDR sends full I/Q; host performs detection, ring buffering, extraction, and recording.
2. **Zynq-Side Ring Buffer** — move pre-trigger buffering closer to the I/Q source while keeping decisions on the host.
3. **Zynq FFT/PSD Detection** — emit compact signal events instead of requiring continuous full-rate host-side detection.
4. **Zynq DDC Workers** — extract a limited number of active narrowband signals before sending them to the host.
5. **Coarse Channelizer Evaluation** — evaluate channelization only after real measurements justify the added complexity.

## Relationship to the Original Project

This fork keeps attribution to the original work and intends to remain compatible with the broader SDR hub/monitor concept where practical. Some changes may be too invasive for direct upstream integration, so the initial work should happen in this experimental branch/fork. Smaller improvements, documentation updates, and bug fixes may still be suitable for upstream pull requests.

## Supported Devices

The original scanner uses [SoapySDR](https://github.com/pothosware/SoapySDR), which allows operation with SDR devices supported by the SoapySDR ecosystem. The next-generation scanner should preserve this flexibility where possible.

## Build

The original project can be built with Docker:

```bash
docker build -t sdr-scanner-ng .
```

Build and runtime instructions will be updated as the new scanner architecture evolves.

## Contributing

This repository is currently experimental. Contributions are welcome, especially in these areas:

- architecture review,
- GNU Radio/C++ scanner design,
- signal detection and tracking,
- ring-buffered I/Q capture,
- metadata and recording formats,
- Zynq/FPGA offload experiments,
- documentation.

Large architectural changes should be discussed before implementation.

## License and Attribution

This project is based on GPLv3-licensed upstream work and remains licensed under the GNU General Public License v3.0. Original work and project concept credit belongs to the upstream author and contributors of the `shajen` SDR projects.

- [`shajen/rtl-sdr-scanner-cpp`](https://github.com/shajen/rtl-sdr-scanner-cpp)
- [`shajen/sdr-hub`](https://github.com/shajen/sdr-hub)
- [`shajen/sdr-monitor`](https://github.com/shajen/sdr-monitor)

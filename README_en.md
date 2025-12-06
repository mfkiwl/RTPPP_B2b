# SBF Subsystem (PPP‑B2b)

This directory implements a minimal Septentrio SBF stream decoder and the BeiDou PPP‑B2b payload processing pipeline, producing real‑time precise orbit/clock corrections for PPP integrations.

## Components

- `SBFDecoder`: lightweight SBF frame handler that performs sync, length/type extraction and CRC16‑CCITT checks, then forwards block 4242 (BDSRawB2b) to the B2b decoder.
- `PPPB2bDecoder`: core B2b payload handler; decodes navigation bits, parses message structures, buffers orbit/clock corrections and maps them to internal RTCM‑style types.
- `SBFcoDecoder`: LDPC error‑correction for B2b navigation bits (BCNV3 over GF(2⁶), extended min‑sum).
- Others: `rtklib.h` and related project types required for RTCM/SSR mapping.

## Data Flow & Responsibilities

- Input: continuous SBF byte stream; each frame starts with `0x24 0x40`, header carries length and type.
- `SBFDecoder`:
  - Detect sync and split frames; validate CRC16; record block types.
  - For each frame, call `PPPB2bDecoder::input(b,len)`; when type is 4242, enter B2b processing.
- `PPPB2bDecoder`:
  - Parse B2b header (TOW, WNc, SVID, etc.), extract 31×4 bytes of navigation bits;
  - Run `SBFcoDecoder::decode_LDPC_navbitsRaw()` for error correction;
  - Use `b2b_parsecorr()` to populate `ppp_ssr_orbit/clock/mask` structures;
  - Map orbit (RAC) and clock (C0) to RTCM3‑style `t_orbCorr/t_clkCorr`, buffer per epoch, and emit results.

## Key Functions

- `SBFDecoder::Decode(buffer, len, errmsg)` (`SBF/SBFDecoder.cpp:124`):
  - Sync, frame extraction, CRC check, type recording; forwards frames to B2b.
- `PPPB2bDecoder::input(sbf_block, len)` (`SBF/PPPB2bDecoder.cpp:226`):
  - Detects type 4242 and calls `decode_b2b_payload()`.
- `PPPB2bDecoder::decode_b2b_payload(payload, payload_len)` (`SBF/PPPB2bDecoder.cpp:240`):
  - Parses header and nav bits; calls `SBFcoDecoder::decode_LDPC_navbitsRaw()`; builds `Message_header`; runs `b2b_parsecorr()`; on success, calls `emitCorrections()`.
- `PPPB2bDecoder::emitCorrections(p_sbas)` (`SBF/PPPB2bDecoder.cpp:811`):
  - Buffers and converts orbit/clock corrections; emits `newOrbCorrections/newClkCorrections` or sends to `ClockOrbit`.
- `SBFcoDecoder::decode_LDPC_navbitsRaw(navBits)` (`SBF/SBFcoDecoder.h:11`, `SBF/SBFcoDecoder.cpp:215`):
  - Extended min‑sum decoding over GF(64), outputs corrected bytes and error count.

## Types & Mapping

- `pppdata` (`SBF/PPPB2bDecoder.h:148`): holds B2b message parse results (`mestype/SSR/BDSweek/sow`, etc.).
- `ppp_ssr_orbit` (`SBF/PPPB2bDecoder.h:174`): orbit corrections (RAC, URA, IODN).
- `ppp_ssr_clock` (`SBF/PPPB2bDecoder.h:184`): clock corrections (C0, IODP).
- `t_orbCorr/t_clkCorr`: project RTCM3‑style correction types produced and emitted by `PPPB2bDecoder`.

## Usage Tips

- Feed an SBF stream (including 4242) to `SBFDecoder(staID)` by continuously appending input buffers.
- Observe emitted orbit/clock outputs via logs/signals; integrate into downstream PPP processing.
- If you only need corrected nav bits, call `SBFcoDecoder::decode_LDPC_navbitsRaw()` directly.

## Notes

- Frames failing CRC are skipped; nav‑bits starting with invalid prefixes (e.g., `EC0FC`) are ignored.
- Correction parameters (iterations, EMS width) live in `SBFcoDecoder.cpp`; adjust for robustness vs speed.
- Week rollover/epoch consistency is checked in `b2b_parsecorr()`; for real‑time streams, WNc/TOW from SBF is typically trusted.


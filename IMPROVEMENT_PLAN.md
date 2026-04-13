# Improvement Plan

This document consolidates findings from a full code quality, architecture,
performance, audio pipeline, and chip-capability review of the repository.
Issues are grouped into phases ordered by dependency and risk: correctness
bugs first, then structural changes that make later work easier, then feature
additions on top of the cleaner foundation.

---

## Phase 1 — Correctness Bugs

These are outright bugs or silent misconfiguration that should be fixed before
anything else.

### 1.1 Wrong property ID: `PROP_FM_VALID_SNR_TIME`

**File:** `legacy/dab_radio_i2c_safe2.py`

`PROP_FM_VALID_SNR_TIME` is assigned `0x3205`. Every reference implementation
of the AN649 programming guide places `FM_VALID_SNR_TIME` at `0x3203`. The AM
counterpart in the same file correctly uses `0x4203`. Property `0x3205` is
unassigned in the spec; the chip receives an undefined write and the actual SNR
settle-time register is left at its power-up default. This affects FM scan
reliability and station detection.

```python
# Fix
PROP_FM_VALID_SNR_TIME = 0x3203   # was incorrectly 0x3205
```

### 1.2 FM de-emphasis never configured

**File:** `legacy/dab_radio_i2c_safe2.py` — `configure_fmhd_frontend`

`FM_AUDIO_DE_EMPHASIS` (property `0x3900`) is never written. The chip
power-up default is 75 µs (Americas/Japan). European and Australian FM
broadcasts use 50 µs pre-emphasis; receiving them with 75 µs de-emphasis
produces audibly over-bright audio. Add a `fm_de_emphasis` field to
`RadioConfig` and write the property during frontend initialisation:

```python
# RadioConfig addition
fm_de_emphasis: str = "50us"   # "75us" | "50us" | "off"

# configure_fmhd_frontend addition
_DE_EMPHASIS = {"75us": 0x00, "50us": 0x01, "off": 0x02}
self.set_property(0x3900, _DE_EMPHASIS[self.fm_de_emphasis])
```

### 1.3 Audio output mode not restored after recording

**File:** `backend.py` — `_start_recording_locked`

When a recording starts and `audio_out` is `"analog"`, the code silently
overrides it to `"both"` and reconfigures the chip mid-playback. When
recording stops the original mode is never restored. Save the pre-recording
mode and restore it in `_stop_recording_locked` / `_finalize_recording_locked`.

### 1.4 `sample_size` / `record_format` mismatch not validated

**File:** `backend.py` — `_start_recording_locked`

`sample_size` configures the chip's I2S output depth; `record_format` is
passed independently to `arecord`. Setting `--sample-size 24` without
`--record-format S24_3LE` silently records garbled audio. Add an explicit
consistency check at recording start and raise a clear error if they disagree.

---

## Phase 2 — Python Modernisation (targeting Python 3.12 / Pi OS Trixie)

Pi OS Trixie ships Python 3.12 as the system default. Several idioms in the
codebase can be updated to match.

### 2.1 Remove `from __future__ import annotations`

PEP 563 deferred annotation evaluation is built into Python 3.12 without the
import. Remove it from `backend.py`, `server.py`, and `cli.py`.

### 2.2 Replace `typing` module generics with built-ins

`Dict`, `List`, `Tuple`, `Optional` from `typing` can all be replaced with
`dict`, `list`, `tuple`, and `X | None`. The codebase already uses `set[str]`
(the built-in form) in one place — make it consistent throughout.

```python
# Before
from typing import Any, Dict, List, Optional
def get_stations(self) -> List[Dict[str, Any]]: ...

# After
from typing import Any
def get_stations(self) -> list[dict[str, Any]]: ...
```

### 2.3 Introduce `TypedDict` for recurring payload shapes

`dict[str, Any]` is the return type for station records, status payloads,
recording metadata, DAB media state, and signal data. Define `TypedDict`
classes for each so their shapes are self-documenting and statically
checkable. Priority:

- `StationInfo`
- `StatusPayload`
- `SignalStatus`
- `RecordingMeta`
- `DabMedia`

### 2.4 Add `_assert_locked` debug helper to `RadioBackend`

The `_locked` naming convention has no runtime enforcement. A lightweight
helper catches missed lock acquisition during testing:

```python
def _assert_locked(self) -> None:
    assert self._lock._is_owned(), "Must hold self._lock before calling this method"
```

Call it at the top of every `_locked` method; strip or gate it behind a
`__debug__` check in production if desired.

### 2.5 Replace silent `except Exception: pass` in cleanup paths

Approximately ten occurrences across `AmplifierGate.close`,
`OledStatusDisplay.close`, and `RadioBackend.close` swallow all exceptions
silently. Replace with at minimum a `logging.debug(...)` call so failures
during shutdown are visible without being fatal.

---

## Phase 3 — Architecture: Module Restructuring

### 3.1 Split `backend.py` into focused sub-modules

`backend.py` is 2,540 lines containing four largely independent concerns.
Proposed structure:

```
raspiaudio_radio/
├── backend/
│   ├── __init__.py       # re-exports RadioBackend, RadioConfig
│   ├── core.py           # RadioBackend class
│   ├── config.py         # RadioConfig frozen dataclass
│   ├── hardware.py       # AmplifierGate, ButtonNavigator, OledStatusDisplay
│   ├── dab_media.py      # MOT/DLS parsing (_parse_mot_segment, _join_mot_segments, etc.)
│   ├── audio.py          # WAV utilities, device detection
│   └── scoring.py        # _dab_score, _analog_score, _station_sort_key
```

`hardware.py` and `dab_media.py` contain logic entirely independent of
`RadioBackend` state and are directly unit-testable once extracted.

### 3.2 Rename the `legacy/` driver to a proper package

`legacy/dab_radio_i2c_safe2.py` is the current production Si468x driver
despite its location. The other three files in `legacy/` are genuinely
superseded. Rename:

```
si468x/
├── __init__.py    # exports Si468xDabRadio, DAB_BAND_III, constants
└── driver.py      # content of dab_radio_i2c_safe2.py
```

Delete `legacy/dab_radio.py`, `legacy/dab_radio_i2c_fixed.py`, and
`legacy/dab_radio_i2c_safe.py` — they are dead code.

---

## Phase 4 — Performance and Concurrency

### 4.1 Release the lock during hardware wait loops

Every tune and scan operation currently holds `self._lock` through
`time.sleep()` loops, freezing the HTTP server, OLED display, and button
navigation for seconds at a time. Use a `threading.Condition` to release the
lock while waiting:

```python
# Current — lock held for the full timeout budget
while time.time() < deadline:
    status = radio.dab_digrad_status()
    if ready(status):
        return status
    time.sleep(0.05)

# Fixed — lock released during each sleep
while time.time() < deadline:
    status = radio.dab_digrad_status()
    if ready(status):
        return status
    self._condition.wait(timeout=0.05)   # releases lock, reacquires on wake
```

Affects: `_wait_dab_ready_locked`, `_wait_fm_signal_locked`,
`_wait_am_signal_locked`, `_grab_dab_services_locked`,
`_scan_am_peaks_locked`.

### 4.2 Move long hardware operations to a worker thread

Scanning (minutes) and tuning (seconds) should run on a dedicated worker
thread with a job queue. Public API methods submit work and return immediately;
callers poll `/api/status` for completion. This resolves the root cause of the
HTTP server appearing frozen during hardware operations.

### 4.3 Cache `_list_recordings_locked` result

`_status_payload_locked` calls `len(self._list_recordings_locked())` on every
invocation — a filesystem glob plus `stat()` and JSON read per file — called
approximately three times per second by the OLED thread. Maintain a
`_recordings_count` integer that is updated only when a recording starts or
stops.

### 4.4 Cache signal state with a TTL

`_read_current_signal_locked()` issues SPI commands on every call to
`_status_payload_locked`. Cache the result with a 500 ms TTL and only re-read
when the cache has expired or a tune event fires.

### 4.5 Remove `time.sleep(0.25)` recording start probe

`_start_recording_locked` sleeps 250 ms while holding the lock to check
whether `arecord` started successfully. The GStreamer replacement (Phase 5)
raises synchronously on pipeline failure — eliminating this sleep entirely.

### 4.6 Wire the INT pin for interrupt-driven DAB media

`RadioConfig.int_pin` defaults to `None`, forcing per-status-call polling of
DAB media packets over SPI. Wire the Si4689's INT output to a GPIO pin,
configure `int_pin`, and replace the `_poll_dab_media_locked()` call in
`_status_payload_locked` with an interrupt-driven callback that fires only
when the chip signals new data.

### 4.7 Systemd status via filesystem instead of subprocess

The two `subprocess.run(["systemctl", "is-enabled/is-active", ...])` calls in
`_refresh_system_service_status_locked` fire inside the lock on every cache
expiry (every two seconds). The `is-enabled` check is reliably answered by
testing for a symlink:

```python
def _service_enabled(name: str) -> bool:
    for d in Path("/etc/systemd/system").glob("*.target.wants"):
        if (d / name).exists():
            return True
    return False
```

The `sudo systemctl enable/disable` mutation command can optionally be
migrated to D-Bus via `python3-dbus`
(`org.freedesktop.systemd1.Manager.EnableUnitFiles`) to remove the `sudo`
dependency entirely.

---

## Phase 5 — Recording Pipeline: GStreamer / PipeWire

Pi OS Trixie (Debian 13, Python 3.12) ships PipeWire 1.x and WirePlumber as
the default audio system. Both `arecord` and `pyalsaaudio` go through
PipeWire's ALSA compatibility shim. The native path is GStreamer with
`pipewiresrc`.

**Dependencies to add:** `python3-gst-1.0`, `gstreamer1.0-pipewire`,
`gir1.2-wp-0.4`  
**Dependencies to remove:** `alsa-utils` (for `arecord`)

All three additions are in the Trixie repos and require no pip or third-party
sources. The `python3-gst-1.0` package explicitly requires Python 3.12,
matching the Trixie default exactly.

### 5.1 Replace `arecord -L` / `arecord -l` device detection

Replace `_list_arecord_named_devices`, `_list_arecord_capture_hardware`, and
`_auto_detect_record_device` with a WirePlumber graph query using
`gi.repository.Wp`. WirePlumber manages device routing and exposes stable node
names that survive reboots, removing the need for hint-based heuristics
entirely.

### 5.2 Replace the `arecord` Popen with a GStreamer pipeline

```python
import gi
gi.require_version("Gst", "1.0")
from gi.repository import Gst

def _build_record_pipeline(node: str, path: Path,
                            rate: int, channels: int,
                            fmt: str) -> Gst.Pipeline:
    return Gst.parse_launch(
        f'pipewiresrc target-object="{node}" ! '
        f'audio/x-raw,format={fmt},rate={rate},channels={channels} ! '
        f'wavenc ! filesink location="{path}"'
    )
```

Advantages over `arecord`:

- Native PipeWire — no ALSA shim
- Buffer management handled by GStreamer internally
- Format mismatch raises at `PLAYING` state transition, not silent corruption
- Stop via EOS event rather than SIGTERM → communicate → kill
- Xrun events surface as GStreamer bus messages
- No `shutil.which("arecord")` guard needed
- No 250 ms startup-probe sleep while holding the lock

### 5.3 Fix WAV trim memory usage

`_trim_wav_leading_seconds` calls `source.readframes(total_frames -
trim_frames)` loading the entire post-trim audio into a single `bytes` object.
For a long recording at 48 kHz stereo 16-bit this is ~11 MB/minute; a
one-hour recording allocates ~660 MB on a device that may have 512 MB total
RAM. Replace with a chunked copy loop (e.g. 4096 frames per iteration).

### 5.4 Replace fixed leading-trim heuristic with I2S lock detection

The 3-second `record_trim_leading_seconds` default removes startup noise from
every recording regardless of whether the I2S stream settled early or late.
Poll `hd_digrad_status()` for `audio_acquired == True` before starting the
pipeline — a hard signal from the chip that the I2S clock is stable. Keep the
fixed trim as a fallback for analog-only paths.

---

## Phase 6 — Si468x Chip Feature Completion

The Si4689 exposes capabilities via AN649 that are either entirely absent from
the implementation or are available but unused.

### 6.1 FM RDS/RBDS — largest missing feature

The chip has a full hardware RDS decoder. Enable it by setting `FM_RDS_CONFIG`
(0x3C02) to `0x0001` in `configure_fmhd_frontend`, then poll `FM_RDS_STATUS`
(0x34) alongside `fm_rsq_status` when an FM station is active.

Fields to extract and surface in the status payload and OLED:

| Field | Description |
|-------|-------------|
| PS    | Programme Service name (8 chars) — station name equivalent |
| RT    | RadioText (64 chars) — song/programme info equivalent to DAB DLS |
| PTY   | Programme Type (news, sport, rock, etc.) |
| TA/TP | Traffic Announcement flags |
| AF    | Alternate Frequencies for the same station |
| CT    | Clock Time |

This brings FM metadata to parity with the existing DAB DLS/artwork
implementation.

### 6.2 Hardware FM and AM seek

Replace the software scan loops in `_scan_fm_clusters_locked` and
`_scan_am_peaks_locked` with `FM_SEEK_START` (0x31) and `AM_SEEK_START`
(0x41). The chip autonomously steps through frequencies and stops at each
valid station using the validity thresholds already configured. Poll
`FM_RSQ_STATUS` / `AM_RSQ_STATUS` with the STCINT bit to detect completion.
Eliminates the 200–350 ms per-frequency sleep loops that currently hold the
global lock for the full scan duration.

### 6.3 DAB ensemble metadata

Call `DAB_GET_ENSEMBLE_INFO` (0xB4) after acquiring a DAB multiplex lock. Add
`ensemble_label` and `ensemble_ecc` to station records. Useful for grouping
stations in the UI by multiplex and for identifying national vs. local
services.

### 6.4 DAB audio codec and bitrate

Call `DAB_GET_AUDIO_INFO` (0xBD) after starting a DAB service. Add to the
status payload:

| Field            | Values                                          |
|------------------|-------------------------------------------------|
| `codec`          | `"DAB"` (MPEG-1 Layer II) or `"DAB+"` (HE-AAC v2) |
| `bitrate_kbps`   | Subchannel bitrate                              |
| `audio_mode`     | `"stereo"` / `"joint_stereo"` / `"dual_channel"` / `"mono"` |
| `sample_rate_hz` | 32000 or 48000                                  |

### 6.5 DAB service linking (DAB → FM fallback)

Implement `DAB_GET_SERVICE_LINKING_INFO` (0xB7). When a DAB service carries a
reference to an FM alternative frequency, store it with the station record.
The backend can automatically fall back to the linked FM frequency when DAB
signal falls below threshold, matching standard DAB receiver behaviour.

### 6.6 DAB mute thresholds

Configure `DAB_CTRL_DAB_MUTE_SIGNAL_LEVEL_THRESHOLD` (0xB501) and
`DAB_CTRL_DAB_MUTE_SIGLOW_THRESHOLD` (0xB505) via `RadioConfig` parameters.
These control the chip's automatic audio mute on weak or lost DAB signal;
currently left at power-up defaults.

### 6.7 FM ACF status — stereo and multipath metrics

Call `FM_ACF_STATUS` (0x33) alongside `FM_RSQ_STATUS` for active FM stations.
Add to the signal status payload:

| Field                | Description                                        |
|----------------------|----------------------------------------------------|
| `stereo`             | Whether the chip is currently decoding in stereo   |
| `blend_level`        | How far blended toward mono (0–100%)               |
| `multipath`          | Multipath interference indicator                   |
| `noise_blanker_active` | Whether the noise blanker is engaged             |

### 6.8 FM soft mute

Configure `FM_SOFTMUTE_SNR_LIMITS` (0x3500) and
`FM_SOFTMUTE_SNR_ATTENUATION` (0x3501) via `RadioConfig`. These control how
aggressively the chip attenuates audio when SNR drops below threshold,
reducing inter-station noise during scanning and on weak signals.

---

## Phase 7 — Test Coverage

With modules extracted in Phase 3, the following are directly unit-testable
with `pytest` and no hardware:

**`backend/dab_media.py`** — all pure functions:
- `_parse_mot_segment` — known-good and malformed payloads
- `_join_mot_segments` — reassembly, missing segments, last-index edge cases
- `_extract_image_payload` — JPEG/PNG signature detection and extraction
- `_decode_dab_text` — each encoding path (UTF-16-BE, UTF-8, Latin-1, fallback)
- `_infer_artist_title` — each separator variant, `"by"` pattern, no-match

**`backend/scoring.py`**:
- `_dab_score` — boundary values, weighting formula
- `_analog_score` — HD bonus, boundary values
- `_clamp_int` — lo/hi boundaries, already-clamped values

**`backend/audio.py`**:
- `_trim_wav_leading_seconds` — create synthetic WAV, trim, verify frame count
- `_wav_duration_seconds` — known WAV files, corrupt file handling

**`backend/core.py`** (integration, mocked hardware):
- Mock `Si468xDabRadio` and verify state machine transitions
- Verify the lock is not held during wait sleeps (measure wall time on scan
  calls with a slow mock)
- Verify recording start/stop state transitions

---

## Dependency Changes (Pi OS Trixie)

| Action | Package | Reason |
|--------|---------|--------|
| Remove | `alsa-utils` | `arecord` replaced by GStreamer |
| Add    | `python3-gst-1.0` | GStreamer Python bindings (requires Python 3.12) |
| Add    | `gstreamer1.0-pipewire` | Native PipeWire GStreamer element |
| Add    | `gir1.2-wp-0.4` | WirePlumber GI bindings for device discovery |

Existing dependencies (`python3-spidev`, `python3-rpi.gpio`, `python3-smbus2`,
`python3-pillow`) are unchanged.

---

## Suggested Execution Order

```
Phase 1 (bugs)
    │
    ▼
Phase 2 (modernisation)
    │
    ▼
Phase 3 (restructure + rename legacy driver)
    │
    ▼
Phase 7 (tests — now possible with extracted modules)
    │
    ├──────────────────────┐
    ▼                      ▼
Phase 4 (concurrency)   Phase 5 (recording pipeline)
    │                      │
    └──────────┬───────────┘
               ▼
          Phase 6 (chip features)
```

Phases 1–3 are prerequisite to everything else: bugs fixed, code modernised,
and modules split before new work is added. Phase 7 (tests) is placed
immediately after Phase 3 because testing a 2,500-line monolith is much
harder than testing extracted modules. Phases 4 and 5 are independent of each
other and can proceed in parallel. Phase 6 chip features build on the
concurrency fixes in Phase 4 (RDS polling and hardware seek both require the
lock to be releasable during waits).

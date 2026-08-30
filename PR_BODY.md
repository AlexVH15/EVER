## Problem

`EncoderConfig.exe` writes `preset.json` in the flat `FFENCODERCONFIG` shape — each
track carries an `encoder` string plus a pre-built pipe-delimited `options` string.
`JsonPresetReader::readEncoderConfig` only understood the nested Voukoder-style shape
(`codec` + `crf`/`preset`/`pixel_format`/… and a top-level `filter` object), so **every
option written by the config tool was silently dropped**:

- video survived by luck, via the existing `pix_fmt == AV_PIX_FMT_NONE` fallback
- audio opened with `sample_fmt == AV_SAMPLE_FMT_NONE`, so `avcodec_open2` failed and
  the export aborted with the Rockstar Editor's in-game "error exporting video"

Separately, `build.bat` only worked with Visual Studio 2022.

## Changes

**`JsonPresetReader`** — detect the flat schema (`video`/`audio` has an `options` or
`encoder` key) and read `encoder`/`options`/`filters`/`sidedata` straight through. The
nested parser stays as the fallback, so hand-edited presets and the bundled
`presets/*.json` still load unchanged.

**`FFmpegEncoder::InitializeAudioEncoder`** — if the options set no sample format, fall
back to `codec->sample_fmts[0]`, mirroring the existing video `pix_fmt` fallback. Turns
a silent config gap into a working export instead of a hard failure.

**`deploy/EVER/preset.json`** — ship it in the flat schema matching
`getDefaultEncoderConfig()`, so the default file and `EncoderConfig.exe` round-trip
identically.

**`EncoderConfig.exe`** — the "H.265 x265" template requested `yuv420p10le` +
`profile=main10`, but the bundled libx265 is an 8-bit build, so that preset always
failed to open. Make the template 8-bit and note that 10-bit HEVC needs a hardware
encoder (`hevc_nvenc` / `hevc_qsv` / `hevc_amf`). Bump the default preset `version` to 2.

**`build.bat`** — detect the installed VS product line and pick the matching CMake
generator (adds "Visual Studio 18 2026"); set `VCPKG_FORCE_DOWNLOADED_BINARIES` so
vcpkg uses its own CMake for dependency builds. Newer VS ships CMake 4.x, which removed
the old policies (`CMP0025`/`CMP0054` OLD) that pinned ports such as x265 3.4 rely on,
breaking their configure step.

## Testing

- Full build from source with Visual Studio 2026 + the `build.bat` changes: succeeds.
- Against the shipped FFmpeg 6.1 runtime DLLs: `avcodec_open2` on `aac` fails with
  `AVERROR(EINVAL)` ("Invalid audio sample format: -1") when no sample format is set,
  and opens once a valid one is set — confirming both the bug and the fix.
- In-game (FiveM): a `hevc_nvenc` preset from `EncoderConfig.exe` now loads
  (`Successfully loaded encoder configuration from: preset.json`) and exports
  successfully where it previously failed with the in-game bake error.

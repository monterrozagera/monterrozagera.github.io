---
title: Forensic Analysis of MP4 Files
permalink: /forensic-analysis-mp4.html
---

# Forensic Analysis of MP4 Files Using `rmp4_meta` (Rust)

In digital forensics, **video files** often hold key evidence — from security camera recordings to smartphone footage. But it's not always about what's visible on screen. Sometimes, the **metadata inside an MP4** file can reveal just as much, if not more.

In this article, we explore how to use a custom Rust CLI tool called `rmp4_meta` to extract **forensic metadata** from `.mp4` video files. We'll explain what information is available, how to interpret it, and why it's valuable in real-world investigations.

---

## Meet `rmp4_meta`: A Rust-Based Video Metadata Extractor

`rmp4_meta` is a lightweight, fast CLI tool built in **Rust**, designed to inspect the internal structure of MP4 files. Unlike GUI-heavy editors or bloated media players, this tool focuses on precision and minimalism — ideal for scripting, automation, or forensic pipelines.

### Features

* Parses MP4 file structure (ISO/IEC 14496-12)
* Extracts duration and timescale
* Lists all video/audio tracks
* Shows codecs used (e.g., `avc1`, `mp4a`)
* Displays video resolution, audio channels, and sample rate

---

## Installation

```bash
git clone https://github.com/monterrozagera/rmp4_meta.git
cd rmp4_meta
cargo build --release
./target/release/rmp4_meta some_video.mp4
```

---

## What You Can Learn from MP4 Metadata

Even without watching a video, MP4 metadata can help answer questions like:

### 1. **Was the video trimmed or re-encoded?**

Compare duration metadata vs. actual file size. A 4-second video with 20 MB of data may indicate re-encoding with a high bitrate, possibly to obscure the original recording time.

### 2. **Which codec was used?**

Certain surveillance systems or phone models use specific codecs. A mismatch in expected codec (e.g., `avc1` vs. `hevc`) might suggest tampering or conversion.

### 3. **What was the original resolution?**

If the resolution is low (`480x360`) but the camera claims to shoot in `1080p`, the file may have been downscaled or screen-recorded.

### 4. **Is there audio at all?**

Audio tracks are sometimes missing in fraudulent videos. `rmp4_meta` clearly shows if audio tracks are present, their sample rate, and number of channels.

### 5. **How many tracks are there?**

Multiple audio/video tracks may indicate overlays or edits. A typical smartphone recording has one video and one audio track — anything else is suspect.

---

## Example Output

```bash
File: evidence_clip.mp4
Duration: 45.82 seconds
Timescale: 1000

Track ID: 1
Type: Video
Codec: avc1
Resolution: 1280x720

Track ID: 2
Type: Audio
Codec: mp4a
Channels: 1, Sample Rate: 48000
```

From this, we can infer:

* This is likely an HD video (720p).
* Audio is mono (e.g., from a small mic).
* Duration is under 1 minute, which is typical of a clip trimmed for sharing.

---

## Use Cases in Digital Forensics

### Timeline Reconstruction

If multiple video clips are presented, analyzing timestamps, durations, and codecs can help reconstruct the **original sequence** or detect missing footage.

### Authenticity Verification

Metadata inconsistencies (e.g., strange codecs, multiple tracks, missing audio) can suggest **tampering**, transcoding, or editing.

### Tool Fingerprinting

Certain devices leave consistent metadata fingerprints. If a video claims to be from a phone model that uses `hevc`, but it’s encoded in `avc1`, that’s a red flag.

### Chain of Custody Support

During evidence intake, automatically extracting MP4 metadata helps create **audit trails** for digital files.

---

## Final Thoughts

In forensics, every bit counts — literally. While MP4 metadata won't tell you everything, it can **validate or contradict** witness statements, expose tampering, and support legal arguments.

Tools like `rmp4_meta`, built in fast and safe languages like Rust, are ideal for scalable and automated analysis.

> **Rust gives you speed and memory safety. Combine that with forensic insight, and you have a powerful tool for digital truth.**

---

## Resources

* [mp4 crate on crates.io](https://crates.io/crates/mp4)
* ISO/IEC 14496-12 (MP4 container format spec)
* Want to extract MP4 metadata in Python? Check out `pymediainfo` or `hachoir`

---

## Future Enhancements for `rmp4_meta`

* Output as JSON (`--json`) for scripting
* Show creation and modification times
* Detect incomplete or corrupt MP4s
* Compare multiple MP4s in batch mode

---

## Contact & Contributions

Have ideas or want to contribute?
Visit [github.com/monterrozagera/rmp4_meta](https://github.com/monterrozagera/rmp4_meta)

---
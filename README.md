# iptv-troubleshooting-cookbook

A working reference for the IPTV failures that actually happen — the stalls at 7:58 PM when everyone tunes in for the match, the program guide that shows yesterday's lineup, the lip-sync that drifts a full second by the third quarter, the channel that plays audio but shows a black rectangle. This repository pairs a diagnostic script, `iptv_doctor.py`, with a cross-linked set of recipes. Each recipe names one symptom, walks through the measurements that confirm or rule out a cause, and ends with a fix you can apply in minutes rather than a forum thread you have to read for an hour.

**Cross-linked cookbook of IPTV troubleshooting recipes: buffering, EPG, audio sync, codecs, network.**

The format is deliberate. A cookbook is not a manual you read cover to cover. You arrive with a problem, you turn to the page that matches it, and you follow the steps. Every recipe here is written to be entered cold — you do not need to have read the previous one. Where a fix in the buffering recipe depends on something explained in the network recipe, there is a link, not a re-explanation. The script automates the measurements that the recipes ask you to take, so the loop is: run `iptv_doctor.py`, read what it found, open the matching recipe, apply the fix, run the script again to confirm.

## Why this exists

The specific problem that started this repository: a 4K sports channel that buffered for three to eight seconds roughly every two minutes during live events, and only during live events. On-demand content from the same provider played flawlessly. Other live channels on the same provider were fine. The user had already done everything the support forums suggested — rebooted the router, swapped HDMI cables, factory-reset the set-top box, switched from Wi-Fi to Ethernet, called the ISP twice. None of it changed anything, because none of it touched the actual cause.

The actual cause was a single overloaded edge node. The provider's content delivery network served that one 4K sports feed from a CDN point-of-presence that saturated during peak concurrency. Every other stream came from a different, healthier node. No amount of cable-swapping reaches a saturated CDN edge two networks away. The only way to find it was to measure: run a traceroute during a buffering event, compare the latency to the same node during an idle hour, and watch the round-trip time climb from 34 ms to over 240 ms exactly when the stalls began. That single measurement — which takes about 90 seconds — answered a question that four phone calls and three hours of cable-swapping could not.

That is the gap this repository fills. IPTV troubleshooting advice on the open web is overwhelmingly generic ("restart your device," "check your internet," "clear the cache") because generic advice is safe and applies to everything. But IPTV failures are specific. Buffering on a wired connection with 200 Mbps of headroom is not the same problem as buffering on congested Wi-Fi, and the fix for one will not touch the other. A missing EPG is sometimes a wrong XMLTV URL and sometimes a timezone offset and sometimes a provider that simply stopped publishing guide data three weeks ago — three causes, three completely different fixes, one identical symptom. Without measurement you are guessing, and guessing in this space wastes hours.

The recipes here exist because the people who actually know how to diagnose these problems — the ones who can tell a codec mismatch from a decode-acceleration failure by looking at one log line — almost never write it down in a form a non-specialist can follow. This is the attempt to write it down.

## Quick start

Clone the repository and run the doctor against a stream URL or a playlist:

```
git clone https://github.com/yourname/iptv-troubleshooting-cookbook.git
cd iptv-troubleshooting-cookbook
python iptv_doctor.py --stream "http://example.com/live/channel/1080.ts"
```

That single command runs the full diagnostic battery — network path, throughput, container probe, codec inventory, audio/video timestamp alignment, and EPG reachability if a guide URL is configured — and prints a ranked list of findings, worst first. A typical run takes 20 to 45 seconds depending on how long you let the throughput sample run.

If you only want to check one subsystem, scope the run with a flag:

```
# Diagnose buffering only — runs a 30-second throughput and jitter sample
python iptv_doctor.py --stream "http://example.com/live/channel.ts" --check buffering

# Check whether the EPG/XMLTV feed is reachable and current
python iptv_doctor.py --epg "http://example.com/xmltv.php?username=u&password=p" --check epg

# Probe codecs and container without playing anything
python iptv_doctor.py --stream "http://example.com/live/channel.ts" --check codec

# Measure audio/video sync drift over a 60-second window
python iptv_doctor.py --stream "http://example.com/live/channel.ts" --check sync --duration 60
```

To diagnose an entire M3U playlist at once and get a per-channel health table:

```
python iptv_doctor.py --playlist ~/Downloads/playlist.m3u --check network --top 25
```

The `--top 25` flag limits the sweep to the first 25 channels so a 4,000-entry playlist does not turn into a 40-minute job. Drop the flag to test everything. Add `--json report.json` to any command to write machine-readable output you can diff between runs or feed into a monitoring dashboard.

Requirements: Python 3.9 or newer, and `ffprobe` (part of FFmpeg) on your `PATH`. The script shells out to `ffprobe` for container and codec inspection because re-implementing a media probe in Python would be slower and less accurate than the tool everyone already trusts. On Debian and Ubuntu, `sudo apt install ffmpeg` covers it; on macOS, `brew install ffmpeg`. The script checks for `ffprobe` on startup and tells you exactly what is missing rather than failing halfway through with a cryptic traceback.

## How it works

The methodology is the same one a competent support engineer uses, made repeatable. Every IPTV symptom has a short list of plausible causes, those causes sit at different layers of the stack, and each layer leaves a measurable signature. The job is to measure each layer, match the signatures against the symptom, and rank the causes by how strongly the evidence supports them. `iptv_doctor.py` does the measuring; the recipes do the matching.

The script works from the outside in, cheapest measurement first. It starts at the network path because that is both the most common failure and the fastest to test — a traceroute and a 30-second throughput sample tell you whether the bytes are even arriving on time before you spend effort inspecting what is inside them. It records round-trip latency to the stream's origin, the variance in that latency (jitter, which matters far more than raw bandwidth for live streams), and sustained throughput against the stream's declared bitrate. A 1080p H.264 stream typically needs 8 to 12 Mbps sustained; a 4K HEVC stream wants 25 Mbps or more. If your measured throughput sits below 1.5× the stream's bitrate, you have found your buffering cause and the script stops weighing other theories so heavily.

If the network path is clean, the script moves inward to the container and codecs. It runs `ffprobe` to enumerate every elementary stream — video codec and profile, audio codec and channel layout, the container format itself — and flags the combinations that specific players are known to choke on. HEVC Main 10 on a device whose hardware decoder only handles 8-bit is a black screen with working audio; AC-3 or E-AC-3 audio on a player without the matching decoder is video with silence. These are not network problems and no amount of bandwidth fixes them, so isolating them early stops you from chasing the wrong layer.

The audio-sync check is the most involved. The script samples presentation timestamps from the audio and video streams over a window you choose (default 30 seconds, extend it with `--duration`) and computes the drift between them. A constant offset — audio always 180 ms ahead — points to a fixed delay you can correct in the player with a single setting. A growing offset — drift that starts near zero and reaches 600 ms after a minute — points to a clock or framerate mismatch that a fixed correction cannot fix, because the gap keeps widening. Telling these two apart by ear is nearly impossible; telling them apart by measurement takes one run.

Finally, if you supply an EPG URL, the script fetches the XMLTV feed, checks that it parses, counts how many channels carry program data, and reports the timestamp of the most recent and furthest-future programs. A guide that parses but whose newest entry is eleven days old is a provider-side data outage, not a client misconfiguration, and that distinction saves you from editing settings that were never wrong.

Everything is ranked by confidence and printed worst-first, with a pointer to the recipe that handles each finding. The script does not apply fixes. It tells you what it found and where to read; you decide and act. That separation is intentional — an automated tool that silently changes player settings or network configuration is a tool you stop trusting the first time it guesses wrong.

## What it solves

The concrete situations this repository is built to handle, each tied to a recipe:

**Buffering that the ISP swears is not their fault.** You are paying for 300 Mbps, a speed test shows 290, and the stream still stalls. The buffering recipe walks through the measurements that distinguish the four real causes — insufficient sustained throughput despite a high burst speed, jitter on an otherwise fast link, a saturated CDN edge node serving only some channels, and an undersized player buffer — and gives the fix for each. The CDN-edge case is the one nobody on the support line can diagnose for you, because it lives two networks away from anything they control, and it is the one this repository was originally built to catch.

**A program guide that is empty, wrong, or frozen.** The EPG recipe separates the three failure modes that all look identical on screen: a guide URL that is simply wrong (you get nothing), a guide that loads but shows times shifted by several hours (a timezone offset between the XMLTV data and your player), and a guide that loads correctly but never updates because the provider stopped publishing data. The script's EPG check reports the age of the newest program entry, which immediately tells you whether you are looking at a client problem or a provider problem. A guide whose freshest entry is 11 days old is not something you can fix by re-entering the URL.

**Lip-sync that drifts during long content.** The audio-sync recipe addresses the failure where dialogue matches lip movement at the start of a two-hour film and is a half-second off by the end. It explains why a fixed audio-delay setting fixes a constant offset but makes a drifting offset worse, and it uses the script's drift measurement to tell you which one you have before you touch a single slider.

**A channel that plays sound over a black screen, or shows video in silence.** The codec recipe handles the mismatches between what a stream contains and what a device can decode. HEVC 10-bit on an 8-bit decoder, AC-3 audio on a player that only does AAC, an unusual container the player half-supports — each produces a specific half-failure, and each has a specific fix (force software decoding, transcode the offending stream, switch players, or change the provider's stream profile). The script's codec inventory hands you the exact codec, profile, and channel layout so you are not guessing what the stream actually contains.

**Intermittent failures you cannot reproduce on demand.** Several recipes lean on the script's `--json` output and the `--duration` flag so you can run a long sample during the window when the problem actually occurs — a 10-minute throughput-and-jitter capture during the 8 PM peak, written to a file you can compare against a clean midday run. Intermittent problems are the ones that survive every support call precisely because they vanish whenever someone is looking; a logged measurement is what makes them visible.

**Triaging a whole playlist.** When a provider's service degrades, it rarely takes down everything at once. The playlist mode tests channels in bulk and produces a health table, so you can see at a glance that 22 of your 25 sports channels are fine and the three that stall all route through the same origin — which turns "the service is broken" into "these specific channels share a failing node," a far more actionable statement.

## Limitations

This is a diagnostic and reference tool, and it is honest about its edges.

It does not fix anything automatically. By design, the script measures and reports; you read the matching recipe and apply the fix yourself. If you want a tool that reaches into your router or rewrites your player config without asking, this is not it, and that is a deliberate choice rather than an unfinished feature.

It cannot see inside the provider's infrastructure. When the script finds a saturated CDN edge or a stalled EPG feed, it can tell you the problem is on the provider's side, but it cannot fix a provider's overloaded node or restart their guide-data pipeline. The honest fix in those cases is sometimes "use a different channel source" or "switch providers," and several recipes say exactly that rather than pretending a client-side tweak exists.

Encrypted and DRM-protected streams are largely opaque to it. The script can probe an unencrypted MPEG-TS or HLS stream in detail, but a stream wrapped in Widevine or a provider's proprietary encryption will not expose its codecs or timestamps to `ffprobe`, and the codec and sync checks will come back empty. The network checks still work, since latency and throughput do not care what the payload is, but the deeper inspection does not.

The audio-sync measurement needs a clean sample. If the stream is also buffering, the timestamp gaps from the stalls contaminate the drift calculation, and the script may report a sync problem that is really a network problem wearing a disguise. The recipes account for this by telling you to resolve buffering first and re-measure sync afterward, but the ordering matters and the tool will not enforce it for you.

It is not a player. The script tells you a stream is healthy or names what is wrong with it, but it does not render video. Some problems — subtle macroblocking, color-space errors, subtitle rendering — are visual and need a human watching a screen. The tool narrows where to look; it does not replace looking.

It assumes a Unix-like environment for the network probes. The traceroute and latency measurements rely on tools that behave predictably on Linux and macOS. On Windows the throughput and codec checks work, but the path-tracing results are less reliable, and the recipes note where that gap shows up.

It tests one moment in time unless you tell it otherwise. A single default run captures roughly 30 seconds. Intermittent problems that surface only at peak hours will not appear in a midday run, which is why the long-duration and JSON-logging options exist — but you have to remember to use them during the window when the problem actually happens.

## Roadmap

- **Continuous monitoring mode.** A `--watch` flag that runs the diagnostic battery on a schedule (every 5 minutes, configurable) and logs results to a rolling file, so intermittent failures get captured automatically instead of requiring you to be at the keyboard when they strike.
- **HLS and DASH manifest analysis.** Deeper parsing of adaptive-bitrate manifests to report on segment durations, available rendition ladders, and whether the player is being pushed down to a lower-quality rendition — a common and invisible cause of "it looks soft but it isn't buffering."
- **A growing recipe library.** The current set covers the five highest-frequency symptom families; planned additions include subtitle and closed-caption failures, multi-audio-track selection problems, and the specific pathologies of streaming over mobile and cellular networks where latency swings are far wider.
- **Provider-fingerprint database.** An opt-in, anonymized collection of common codec profiles, EPG formats, and CDN behaviors per provider type, so the script can say "this looks like a known provider-side guide outage" with more confidence instead of inferring it fresh each time.
- **Baseline comparison.** A `--baseline` mode that stores a known-good measurement and flags how far the current run has drifted from it, turning "is this slower than usual?" from a guess into a number.
- **Player-config export.** For the fixes that come down to a specific player setting — audio delay, forced software decode, buffer size — generate the exact configuration snippet for the common players rather than describing the setting in prose.

## Recommended reading

We test IPTV providers across a 90-day rig with 5 devices and 7 weighted
criteria. Full rankings + methodology:

- [Best IPTV Service 2026 — Our independent ranking](https://streamreviewhq.com/best-iptv-service-2026/)
- [Best IPTV Subscription 2026](https://streamreviewhq.com/best-iptv-subscription-2026/)
- [Best IPTV for Firestick 2026](https://streamreviewhq.com/best-iptv-for-firestick-2026/)
- [IPTV Firestick Setup Guide 2026](https://streamreviewhq.com/iptv-firestick-setup-2026/)
- [IPTV vs Cable TV 2026](https://streamreviewhq.com/iptv-vs-cable-tv-2026/)
- [Best IPTV for World Cup 2026](https://streamreviewhq.com/best-iptv-world-cup-2026/)
- [Complete Apple TV 4K IPTV setup](https://streamreviewhq.com/complete-iptv-setup-guide-for-apple-tv-4k-2026/)
- [Beast IPTV vs IPTVTheOne — 90 days](https://streamreviewhq.com/beast-iptv-review-90-days-tested-vs-iptvtheone-2026/)
- [IPTVTheOne vs Kemo IPTV](https://streamreviewhq.com/iptvtheone-vs-kemo-iptv-which-wins-in-2026/)
- [IPTVTheOne — 90-day deep review](https://streamreviewhq.com/iptvtheone-review/)
- [Kemo IPTV review](https://streamreviewhq.com/kemo-iptv-review/)
- [OTT Navigator review](https://streamreviewhq.com/ott-navigator-review/)

Country picks:
[USA](https://streamreviewhq.com/best-iptv-usa/) ·
[UK](https://streamreviewhq.com/best-iptv-uk/) ·
[Canada](https://streamreviewhq.com/best-iptv-canada/) ·
[Australia](https://streamreviewhq.com/best-iptv-australia/) ·
[Germany](https://streamreviewhq.com/best-iptv-germany/)

Reference baseline provider: [IPTVTheOne](https://iptvtheone.com/) —
see also their [provider guide](https://iptvtheone.com/best-iptv-provider-2026-guide/),
[USA pack](https://iptvtheone.com/best-iptv-subscription-usa-2026/),
[sports pack](https://iptvtheone.com/best-iptv-sports-streaming-2026/),
[multi-connection](https://iptvtheone.com/buy-iptv-subscription-multi-connection/).

## Sister projects

- [Streamline Media Lab](https://github.com/Streamline-Media-Lab)
- [Independent Streamer Reviews](https://github.com/Independent-Streamer-Reviews)
- [Living Room Tech Hub](https://github.com/Living-Room-Tech-Hub)
- [Open Streaming Almanac](https://github.com/Open-Streaming-Almanac)
- [Stream Lab HQ](https://github.com/Stream-Lab-HQ)
- [Modern Cord Cutters](https://github.com/Modern-Cord-Cutters)
- [Cord Cutter Almanac](https://github.com/Cord-Cutter-Almanac)
- [The Set Top Review](https://github.com/The-Set-Top-Review)
- [Best Review Service](https://github.com/Best-Review-Service)

## References

- [IPTV — Wikipedia](https://en.wikipedia.org/wiki/IPTV)
- [HTTP Live Streaming (HLS) — Wikipedia](https://en.wikipedia.org/wiki/HTTP_Live_Streaming)
- [MPEG-DASH — Wikipedia](https://en.wikipedia.org/wiki/Dynamic_Adaptive_Streaming_over_HTTP)
- [XMLTV — Wikipedia](https://en.wikipedia.org/wiki/XMLTV)
- [HEVC / H.265 — Wikipedia](https://en.wikipedia.org/wiki/High_Efficiency_Video_Coding)
- [Streaming media — Wikipedia](https://en.wikipedia.org/wiki/Streaming_media)
- [Akamai — Streaming primer](https://www.akamai.com/glossary/what-is-streaming)
- [Cloudflare — Stream delivery](https://www.cloudflare.com/learning/video/what-is-streaming/)
- [Statista — Streaming market](https://www.statista.com/topics/8946/streaming-services-in-the-united-states/)
- [Nielsen — TV viewership reports](https://www.nielsen.com/insights/)

## License

MIT for the code. CC-BY-4.0 for the written notes.

---
*Last verified: July 11, 2026*

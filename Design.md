Design Report: Bidirectional Audio Interview Assistant over VNC + PulseAudio (Android 8.0+)

Conclusion: The architecture is feasible on a 4GB Linux machine. The core design is: VNC handles screen only, PulseAudio handles bidirectional audio, and the Android client runs two independent audio streams (playback + recording) over a custom protocol. Android 8.0 (API 26) has known audio stack issues, so avoid AAudio and use Java AudioTrack / AudioRecord for reliable low-latency behavior. On Linux, use the "null sink + remap source" or "pipe source" approach to create a virtual microphone; the null-sink approach is simpler and better validated by the community.

1. System Architecture Overview

The system has three parts: Linux remote host (interview software + VNC server + PulseAudio), Android client (screen receiver + bidirectional audio), and the network layer (VNC protocol + custom audio protocol).

VNC and audio are fully decoupled: VNC transports only the screen and keyboard/mouse events, while audio travels on an independent channel. This keeps memory usage well within 4GB and ensures audio latency is not affected by the VNC frame rate.

Data flow:

· Remote → Android (downlink audio): Remote PulseAudio monitor (or virtual sink monitor) → Linux custom audio service captures → encodes (Opus or raw PCM) → TCP/WebSocket → Android decodes → AudioTrack playback.
· Android → Remote (uplink microphone): Android AudioRecord captures → encodes → TCP/WebSocket → Linux audio service receives → writes into virtual microphone sink/pipe → interview software sees a "physical microphone".

2. Linux-Side Design (4GB Machine)

2.1 Virtual Microphone Creation (Option Selection)

The interview software needs to "see" a microphone. There are two proven approaches on Linux:

Option A: Null Sink + Remap Source (Recommended)

This is a widely validated community pattern. Create a null sink; PulseAudio automatically generates a monitor for it. Then remap that monitor into a source. Your Linux audio service only needs to paplay the PCM data received from Android into that sink, and the interview software will read audio from the remapped source.

Reference commands (used in similar real-world projects):

```bash
pactl load-module module-null-sink sink_name=VirtualMicSink \
  sink_properties=device.description="Virtual_Mic_Sink"

pactl load-module module-remap-source master=VirtualMicSink.monitor \
  source_name=VirtualMic \
  source_properties=device.description="Virtual_Microphone"
```

Pros: Simple; no manual audio format negotiation (PulseAudio resamples automatically). Cons: Introduces a small delay (typically <20ms).

Option B: Pipe Source

Use module-pipe-source to create a named pipe as the microphone input. The Linux audio service writes PCM directly into the FIFO. Pipe Source accepts explicit format parameters (s16le, sample rate, channel count).

Pros: Shortest path, lowest latency. Cons: Your service must write data in exactly the correct format, or the interview software will hear noise.

Recommendation: Start with Option A; evaluate Option B only after the system is stable.

2.2 Downlink Audio Capture

The remote interview software outputs to the default sink. Your Linux audio service must capture that sink's monitor (i.e., "what is actually being heard") and encode it for Android.

If only the interview software produces sound on the remote host, capturing the default sink's monitor is sufficient. If you want tighter isolation, create a dedicated sink for the interview software and capture its monitor instead.

2.3 Audio Service Implementation Notes

Implement the Linux audio service in Python or C. Responsibilities:

1. Continuously read PCM from the downlink monitor via parec or the PulseAudio API.
2. Package it per the custom protocol (Opus or raw PCM) and send over TCP to Android.
3. Listen for the Android uplink connection, receive PCM, and write it into VirtualMicSink (via paplay --raw or the PulseAudio API directly).

Parameter consistency: Uplink and downlink must agree on sample rate, channel count, and sample format. Recommended: 48kHz / mono / s16le or 16kHz / mono / s16le (Opus performs better at 16kHz in terms of latency and bandwidth). Opus frame sizes support 2.5/5/10/20/40/60ms; 20ms is recommended as the balance between latency and packet-loss resilience.

2.4 Resource Budget on 4GB

Component Approx. RAM
Lightweight desktop (XFCE/LXQt) ~400–700 MB
VNC server (TigerVNC/x11vnc) ~50–150 MB
PulseAudio + virtual sink/source ~30–50 MB
Custom audio service ~30–80 MB
Interview software Remaining

Headroom is comfortable. Avoid GNOME/KDE on a 4GB host; prefer a lightweight desktop.

3. Custom Audio Protocol Design

3.1 Transport Choice

TCP with length-prefixed framing is the simplest reliable option. Each frame has a small header:

```
[2 bytes: payload length][1 byte: frame type][payload]
```

Frame types: 0x01 = Opus/PCM audio frame, 0x02 = heartbeat, 0x03 = control (e.g., format negotiation).

Alternatively, WebSocket can be used if you want easier Android-side integration and future browser compatibility. For a custom app, plain TCP is lighter.

3.2 Codec Choice

· Opus (recommended): 16–48kHz, 20ms frames, 16–32kbps for voice. Excellent latency and packet-loss resilience. Available on Android via MediaCodec or a native libopus build.
· Raw PCM: Simpler, no codec dependency, but higher bandwidth (~768kbps at 48kHz mono s16le) and no loss resilience. Acceptable on LAN.

Recommendation: Opus for both directions. Use MediaCodec on Android with the audio/opus MIME type; on Linux use libopus or opusenc/opusdec.

3.3 Handshake

On connect, the client sends a small JSON or binary handshake specifying:

· Protocol version
· Uplink format (codec, sample rate, channels, frame size)
· Downlink format request

The server replies with its accepted parameters. This makes the system robust to future format changes.

3.4 Clock Drift and Jitter

Android and Linux clocks are not synchronized. For the downlink, maintain a small jitter buffer (2–4 frames, i.e., 40–80ms at 20ms frames) on Android before playback. For the uplink, the Linux service should tolerate small timing variations; if using a null sink, PulseAudio handles the drift. If using pipe-source, implement a simple adaptive buffer or drop/duplicate frames when the pipe fills/drains.

4. Android Client Design (Android 8.0+)

4.1 Minimum SDK and Constraints

· minSdkVersion 26 (Android 8.0).
· Do not use AAudio (introduced in API 26 but with well-documented bugs on 8.0/8.1). Use Java AudioTrack and AudioRecord instead. These are stable on API 26 and adequate for voice-grade latency.
· For lower latency on AudioTrack, use AudioTrack.MODE_STREAM with a small buffer and the PERFORMANCE_MODE_LOW_LATENCY flag where available (API 26+). Note that actual low-latency support depends on the device's audio HAL; treat it as best-effort.

4.2 VNC Screen Component

Use an existing VNC client library or embed a proven open-source client (e.g., a fork of android-vnc-viewer or a modern libvncclient JNI wrapper). The screen component is independent of the audio pipeline and should be treated as a black box that handles RFB protocol, input events, and rendering.

4.3 Audio Playback (Downlink)

· Receive Opus frames over TCP.
· Decode via MediaCodec (audio/opus) or libopus JNI.
· Feed PCM into AudioTrack in MODE_STREAM.
· Use a jitter buffer of 2–4 frames before starting playback; drain/rebuffer if underrun occurs.
· Run the network receive loop on a dedicated thread; do not block the UI thread.

4.4 Audio Capture (Uplink)

· Use AudioRecord with MediaRecorder.AudioSource.VOICE_COMMUNICATION (preferred for interview use; applies echo cancellation/noise suppression where the device supports it) or MIC if you want raw audio.
· Sample rate: 16kHz or 48kHz mono, 16-bit PCM.
· Encode to Opus in 20ms frames; send over TCP.
· Request RECORD_AUDIO permission at runtime (mandatory since API 23).

4.5 Permissions and Manifest

Required permissions:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
```

For Android 8.0, if the app must keep audio running while backgrounded, use a foreground service with a persistent notification. Android 8.0 enforces background execution limits, so a plain background service will be killed.

4.6 Threading Model

· UI thread: VNC rendering surface + controls.
· VNC network thread: RFB protocol.
· Audio downlink thread: TCP receive → decode → jitter buffer → AudioTrack.
· Audio uplink thread: AudioRecord read → encode → TCP send.
· Heartbeat/keepalive thread (optional, can be merged).

Keep audio threads at THREAD_PRIORITY_URGENT_AUDIO where possible.

4.7 Android 8.0-Specific Pitfalls

· AAudio bugs: avoid entirely; use Java audio APIs.
· Background limits: use a foreground service for audio.
· Notification channels: required since API 26; create a channel for the foreground service notification.
· Runtime permissions: RECORD_AUDIO must be requested at runtime.
· Doze mode: audio in a foreground service is exempt, but network keepalives should be robust to brief interruptions.

5. End-to-End Latency Budget

Stage Approx. latency
Remote capture (monitor read) 10–20 ms
Opus encode 5–10 ms
Network (LAN) 1–5 ms
Android jitter buffer 40–80 ms
Opus decode 5–10 ms
AudioTrack output 20–50 ms
Total downlink ~80–175 ms
Total uplink (symmetric, minus jitter buffer on Linux) ~50–120 ms

This is acceptable for interview assistance. If lower latency is required, reduce the jitter buffer to 2 frames and use 10ms Opus frames (at the cost of packet-loss resilience).

6. Risks and Mitigations

Risk Mitigation
Android 8.0 audio glitches Avoid AAudio; use AudioTrack/AudioRecord; test on real API 26 devices
Background service killed Foreground service + notification channel
Clock drift between endpoints Jitter buffer (downlink); adaptive buffer or drop/duplicate (uplink)
PulseAudio virtual mic not detected by interview software Verify with pactl list sources; test with arecord -d 5 -f S16_LE -r 16000 test.wav
Network jitter/loss Opus FEC + jitter buffer; optionally add a simple retransmit for critical frames
VNC and audio competing for CPU on 4GB host Use lightweight desktop; cap VNC encoding quality; run audio service at normal priority

7. Implementation Roadmap

1. Linux virtual mic PoC: null sink + remap source; verify an interview app (or arecord) sees "Virtual_Microphone".
2. Linux audio service MVP: capture default monitor → raw PCM over TCP; receive uplink PCM → paplay into null sink. Test with a simple CLI client.
3. Android audio MVP: AudioTrack playback of raw PCM; AudioRecord capture → raw PCM over TCP. Verify round-trip audio on a real Android 8.0 device.
4. Add Opus: replace raw PCM with Opus on both ends; measure latency.
5. Integrate VNC client: embed VNC screen component alongside the audio pipeline.
6. Foreground service + permissions: make audio survive backgrounding on API 26.
7. Hardening: jitter buffer tuning, reconnect logic, error handling, heartbeat.

8. Summary

The design is sound for a 4GB Linux host running VNC plus bidirectional PulseAudio. The key decisions are:

· Decouple VNC from audio — VNC only carries the screen.
· Use null sink + remap source on Linux to present a virtual microphone to the interview software.
· Use a custom length-prefixed TCP protocol with Opus 20ms frames for both directions.
· On Android 8.0, avoid AAudio and build on Java AudioTrack / AudioRecord, wrapped in a foreground service.
· Maintain a small jitter buffer on the Android downlink and tolerate drift on the Linux uplink.

With these choices, the system is implementable with modest effort, stays within the 4GB memory budget, and delivers voice-grade latency suitable for interview assistance.
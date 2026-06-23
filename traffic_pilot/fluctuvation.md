# RTSP, GStreamer, and NVDEC Utilization Fluctuation

This note explains why NVDEC utilization can fluctuate while RTSP camera FPS is still correct.

## Short Answer

The fluctuation is not caused by the Linux TCP receive buffer filling up.

It is caused by how compressed RTSP video reaches the decoder:

```text
RTSP over TCP
  -> RTP packets
  -> GStreamer jitterbuffer / depayloader / parser
  -> complete H264/H265 access units
  -> NVDEC
```

NVDEC does not decode arbitrary TCP bytes or partial RTP packets. It decodes complete compressed
video units, usually one access unit / frame at a time. GStreamer must therefore collect enough
incoming packets to form a valid decodable unit before handing work to NVDEC.

That collection step is normal and required. It is not the same as waiting for the TCP socket
buffer to fill.

## What The TCP Buffer Does

The TCP receive buffer is only byte storage managed by the OS. It holds bytes that have arrived
from the network until the application reads them.

If the TCP buffer were the bottleneck, we would expect symptoms such as:

- RTSP stalls or reconnects
- packet retransmission issues
- decoded FPS dropping below camera FPS
- pipeline starvation

If `decfps` remains stable at the expected camera FPS, the TCP buffer is not the main problem.

## What GStreamer Must Do

The camera sends compressed video. A single video frame is split across many network packets.
GStreamer cannot send each partial packet directly to NVDEC as if it were a frame.

It must first reconstruct valid compressed video units:

```text
many TCP/RTP packets -> one H264/H265 access unit -> NVDEC decode job
```

That means NVDEC receives decode jobs in frame-shaped chunks, not as a perfectly smooth byte
stream.

## RTSP To NVDEC Pipeline

In the DeepStream daemon, the RTSP receive/decode path is hidden inside `nvurisrcbin`:

```text
IP camera
  -> RTSP over TCP socket
  -> rtspsrc
  -> RTP jitterbuffer
  -> RTP depayloader
  -> H264/H265 parser
  -> nvv4l2decoder
  -> NVDEC hardware
  -> decoded NV12/NVMM frame
```

The daemon creates the source in `cpp/ds/src/traffic_pilotd_ds.cpp`:

```cpp
GstElement *src = make("nvurisrcbin", ("src_" + std::to_string(i)).c_str());
g_object_set(src, "uri", uri.c_str(), NULL);
```

For RTSP sources, it configures TCP transport and jitterbuffer latency:

```cpp
g_object_set(src,
    "rtsp-reconnect-interval", 10,
    "select-rtp-protocol", 4,
    "latency", 500,
    NULL);
```

Conceptually, the internal stages behave like this:

1. **Camera encoder**

   The camera produces compressed H264/H265 frames. These frames are not equal size. Keyframes
   are much larger than normal P-frames.

2. **RTP packetization**

   One compressed frame is split across many RTP packets:

   ```text
   one compressed frame -> many RTP packets
   ```

3. **RTSP over TCP**

   RTP packets are carried over the RTSP TCP connection. TCP only transports bytes reliably; it
   does not understand video frames.

4. **OS TCP receive buffer**

   The kernel briefly holds arriving bytes until GStreamer reads them. This is not the deliberate
   video-timing buffer.

5. **`rtspsrc`**

   `rtspsrc` reads the TCP byte stream and reconstructs RTP packets.

6. **RTP jitterbuffer**

   This is the main timing buffer. It uses RTP sequence numbers and timestamps to absorb jitter,
   handle reordering, and release packets in timestamp order. The `latency` setting is the maximum
   timing cushion, not a requirement to fill a bucket before decoding.

7. **Depayloader**

   `rtph264depay` or `rtph265depay` removes the RTP wrapping and reconstructs H264/H265 payloads.
   It may need enough RTP payload to complete a fragmented video unit.

8. **Parser**

   `h264parse` or `h265parse` identifies valid compressed unit boundaries and prepares decoder
   input. This is where complete decoder-ready access units are formed.

9. **`nvv4l2decoder` / NVDEC**

   Once a valid compressed unit is available, the decoder submits it to NVDEC. NVDEC outputs raw
   NV12/NVMM frames.

10. **Post-decode drop**

    The daemon's `decode_drop_probe` runs after decode. In live mode it counts every decoded frame
    and passes only `infer_fps` frames toward `nvstreammux` and inference:

    ```text
    decode every incoming frame -> keep selected infer_fps frames -> drop the rest before inference
    ```

### Where Waiting Can Happen

The important wait points are:

```text
OS TCP receive buffer
  waits until the application reads bytes; not video-aware

RTP jitterbuffer
  waits within the latency window for packet timing, ordering, and late packets

Depayloader / parser
  waits until enough payload exists to form valid H264/H265 units

Decoder / downstream queues
  can wait if the decoder or downstream pipeline is backpressured
```

The key distinction:

```text
Wrong model:
  TCP bucket fills -> decode starts

Correct model:
  RTP packets arrive
    -> jitterbuffer orders/times them
    -> depay/parser forms complete H264/H265 access units
    -> NVDEC decodes those complete units
```

## Why The Work Is Bursty

Even at a constant 30 FPS, compressed video work is not identical every 33 ms.

Reasons:

- One frame consists of many packets.
- I-frames / keyframes are much larger than P-frames.
- The camera encoder may send packet groups close together.
- RTSP-over-TCP delivery and OS scheduling can bunch packets.
- GStreamer releases complete access units, not partial bytes.
- NVDEC can finish each decode job quickly, then wait for the next complete unit.

So the camera can be producing a steady 30 FPS while NVDEC utilization still appears bursty.

## Packet Size Is Not Fixed By This Code

The DeepStream daemon does not configure packet size, MTU, or TCP socket receive size.

The relevant code is in `cpp/ds/src/traffic_pilotd_ds.cpp`:

```cpp
g_object_set(src, "uri", uri.c_str(), NULL);

if (uri.rfind("rtsp://", 0) == 0)
  g_object_set(src,
      "rtsp-reconnect-interval", 10,
      "select-rtp-protocol", 4,
      "latency", 500,
      NULL);
```

This configures:

- RTSP over TCP
- reconnect behavior
- GStreamer RTSP latency / jitterbuffer delay

It does not configure a fixed RTP packet size.

Packet sizing mainly comes from:

- camera encoder / RTP packetizer
- network MTU
- TCP segmentation
- H264/H265 frame size

On normal Ethernet, packets are typically around MTU-sized chunks, but compressed frame sizes vary
heavily. A keyframe can be many times larger than a normal P-frame.

## What `latency` Means

The `latency` property is the GStreamer RTSP jitterbuffer delay in milliseconds.

It lets GStreamer hold a small timing cushion so packets can be reordered and released according
to RTP timestamps. Higher latency usually makes the stream more tolerant of network jitter but
adds live delay.

Tradeoff:

```text
higher latency -> more buffering tolerance, more delay
lower latency  -> lower delay, more jitter sensitivity
latency = 0    -> immediate forwarding, usually more bursty and fragile
```

Latency is not a TCP bucket size. It is a GStreamer timing buffer.

## Current Pipeline Behavior

In live mode, the intended behavior is:

```text
decode every camera frame
drop frames after decode to match infer_fps
send only selected frames to inference
```

That is handled by `decode_drop_probe` in `cpp/ds/src/traffic_pilotd_ds.cpp`.

So if the camera is sending 30 FPS and `infer_fps` is 8, NVDEC still decodes the incoming stream
first. The drop happens after decode, before the inference cascade.

## What Can Actually Improve Smoothness

If the goal is smoother NVDEC utilization, the useful levers are before or at the encoded stream:

- Use camera encoder CBR instead of highly variable bitrate if available.
- Reduce keyframe / I-frame size spikes.
- Use a regular GOP.
- Lower camera FPS, resolution, or bitrate.
- Use a camera substream.
- Keep RTSP latency at a practical value such as 100-500 ms instead of 0.

If the goal is lower inference GPU fluctuation, use inference-side levers:

- `PLATE_TOP_K`
- `PGIE_INTERVAL`
- disabling live view with `DS_LIVEVIEW=0`
- reducing plate/OCR workload

Those do not reduce NVDEC work, because NVDEC has already decoded the frames.

## Conclusion

The observed NVDEC fluctuation is best explained as:

```text
variable compressed video access-unit size and timing
+ GStreamer assembling valid decodable units
+ NVDEC processing complete units quickly
+ utilization sampling catching busy/idle periods
```

It is not caused by the TCP receive buffer filling up, and it cannot be fixed by pushing partial
packets into NVDEC. NVDEC requires valid complete compressed video units.

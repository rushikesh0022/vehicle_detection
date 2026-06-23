# DeepStream Stabilization Methods Tried

This note lists the methods we tried from the morning debugging session to make NVDEC / GR3D /
DeepStream FPS behavior steadier, and what each method proved.

The goal was not just to make utilization look flat. The real goal was to understand where
fluctuation enters the pipeline and which changes actually make the live path more stable.

## 1. Checked Whether TCP Buffer Size Was The Cause

Question:

```text
Is the TCP receive bucket filling and then releasing bursts into decode?
```

What we checked:

```text
tcp_rmem
rmem_default
rmem_max
tcp_moderate_rcvbuf
```

Observed:

```text
tcp_rmem = 4096 131072 6291456
tcp_moderate_rcvbuf = 1
```

Meaning:

```text
TCP receive autotuning is enabled.
Each RTSP/TCP socket can grow receive buffering up to about 6 MB.
```

Conclusion:

```text
TCP bucket size is not the main reason for the fluctuation.
```

Why:

```text
NVDEC does not decode TCP bytes directly.
GStreamer must first form complete H264/H265 access units.
```

Result:

```text
We did not tune tcp_rmem as the fix.
```

## 2. Tested RTSP Latency / Jitterbuffer Settings

Question:

```text
Can RTSP latency make packet delivery smoother?
```

Code path:

```text
cpp/ds/src/traffic_pilotd_ds.cpp
```

The live source uses `nvurisrcbin`, and for RTSP it sets:

```cpp
"select-rtp-protocol", 4
"latency", rtsp_latency
```

Meaning:

```text
select-rtp-protocol=4 -> RTSP/RTP over TCP
latency              -> GStreamer RTP jitterbuffer latency
```

Methods tried:

```text
RTSP_LATENCY_MS=0
RTSP_LATENCY_MS=100
RTSP_LATENCY_MS=500
RTSP_LATENCY_MS=1000
```

Observed:

```text
Changing latency changed the timing pattern.
Some values made NVDEC look steadier for a while.
latency=0 made the stream lower-delay but more timing-sensitive.
```

Conclusion:

```text
RTSP latency can shape the timing, but it is not a complete fix.
```

Why:

```text
The jitterbuffer can reorder/time packets, but it cannot make every compressed frame equal size or
make inference work constant.
```

## 3. Confirmed Where Packet Formation Happens

Question:

```text
Is GStreamer waiting for a packet bucket to fill before decode?
```

Conclusion:

```text
No. It waits only until valid compressed video units are available.
```

Correct path:

```text
RTSP/TCP packets
  -> RTP jitterbuffer
  -> depayloader
  -> H264/H265 parser
  -> complete access unit
  -> NVDEC
```

Important point:

```text
We cannot send partial RTP packets directly into NVDEC.
NVDEC requires valid compressed access units.
```

Result:

```text
We stopped treating the TCP socket as the main "bucket."
The real pre-NVDEC video buffer is the jitterbuffer/depay/parser path.
```

## 4. Measured Decode-Only Throughput

Question:

```text
Is NVDEC itself too slow?
```

Script created:

```text
/tmp/decode_fps_bench.sh
/tmp/decode_fps_bench.py
```

Command used:

```bash
/tmp/decode_fps_bench.sh /home/admin1/Downloads/1_1.mp4 --codec h265 --file-rtp
```

Pipeline:

```text
filesrc
  -> demux / parser
  -> RTP pay
  -> RTP depay
  -> parser
  -> NVDEC
  -> fakesink
```

Observed:

```text
decode_throughput_fps ~= 900 fps
decoded_frame_gap ~= 1.1 ms average
```

Conclusion:

```text
NVDEC has enough raw decode capacity.
Decode-only is not the bottleneck.
```

Result:

```text
We moved focus away from NVDEC capacity and toward live source timing and downstream inference.
```

## 5. Measured RTSP Access-Unit Timing

Question:

```text
Where is time spent before NVDEC on a real RTSP camera?
```

Script created:

```text
/tmp/nvdec_wait_probe.sh
/tmp/nvdec_wait_probe.py
```

Command used:

```bash
/tmp/nvdec_wait_probe.sh 'rtsp://192.168.1.221/video0.sdp' \
  --codec h264 \
  --rtsp-latency-ms 500 \
  --seconds 30
```

Measured:

```text
RTP packet interarrival
first RTP packet to complete access unit
parser input to unit ready
complete unit arrival spacing
decoder input/output FPS
```

Observed when the camera was configured near 8 fps:

```text
decoder_input_fps ~= 8.22
decoded_output_fps ~= 8.08
parser_input_to_unit_ready ~= 0.105 ms average
first_rtp_packet_to_unit_ready ~= 5 ms average
input_wait_between_complete_units ~= 119 ms average
```

Conclusion:

```text
Packet/access-unit formation was not the expensive part.
The long wait was mostly the camera frame period.
```

Result:

```text
We confirmed the camera feed timing, not parser cost, was controlling unit arrival.
```

## 6. Tested Decode + One YOLO Model On File

Question:

```text
How much throughput remains after adding vehicle YOLO inference?
```

Script created:

```text
/tmp/ds_infer_fps_bench.sh
/tmp/ds_infer_fps_bench.py
```

Pipeline:

```text
file decode
  -> nvstreammux
  -> nvinfer vehicle YOLO
  -> fakesink
```

Command used:

```bash
/tmp/ds_infer_fps_bench.sh /home/admin1/Downloads/1_1.mp4 --seconds 30
```

Observed:

```text
decode_fps ~= 211.87
mux_fps ~= 211.80
infer_out_fps ~= 211.57
```

Conclusion:

```text
One vehicle YOLO model can process about 210 fps.
```

Result:

```text
This is enough for 20 streams at 8 fps, because 20 * 8 = 160 fps.
```

## 7. Created A Live One-YOLO Pipeline

Question:

```text
Can the real RTSP live source path run with only one YOLO model?
```

Script created:

```text
/tmp/ds_yolo_live_fps.sh
/tmp/ds_yolo_live_fps.py
```

Pipeline:

```text
nvurisrcbin x N
  -> nvstreammux
  -> nvinfer vehicle YOLO only
  -> fakesink
```

This excludes:

```text
nvtracker
plate model
OCR model
Redis publish
analytics probe
```

Purpose:

```text
Separate RTSP + NVDEC + vehicle YOLO from the full 4-model ALPR chain.
```


# MediaCodec & Media Framework — Video/Audio Decode on Pi4

## 1. Tổng quan

```
App (MediaPlayer / ExoPlayer)
  │ MediaCodec API
  ▼
MediaCodec (Java/NDK)
  │ Binder
  ▼
MediaCodecService (mediaserver process)
  │ Codec2 / OMX interface
  ▼
Codec HAL implementation
  ├── SW codec (libavcodec / libstagefright)  ← CPU decode
  └── HW codec (Pi4: H264 via VideoCore VI)   ← GPU firmware
        │ VCHIQ mailbox → VideoCore firmware
        ▼
      Decoded frames (YUV buffers)
        │ DMA-BUF / gralloc
        ▼
      SurfaceFlinger → Display
```

---

## 2. MediaCodec API

```java
// Decode H264 video to Surface (zero-copy to GPU)
MediaCodec decoder = MediaCodec.createDecoderByType("video/avc");

MediaFormat format = MediaFormat.createVideoFormat("video/avc", 1920, 1080);
format.setByteBuffer("csd-0", spsBuffer);  // Sequence Parameter Set
format.setByteBuffer("csd-1", ppsBuffer);  // Picture Parameter Set

Surface surface = ...;  // SurfaceView or SurfaceTexture
decoder.configure(format, surface, null, 0);
decoder.start();

// Feed compressed data
int inputIndex = decoder.dequeueInputBuffer(10000);
ByteBuffer inputBuf = decoder.getInputBuffer(inputIndex);
inputBuf.put(h264NalUnit);
decoder.queueInputBuffer(inputIndex, 0, size, presentationTimeUs, 0);

// Get decoded frame
MediaCodec.BufferInfo info = new MediaCodec.BufferInfo();
int outputIndex = decoder.dequeueOutputBuffer(info, 10000);
if (outputIndex >= 0) {
    decoder.releaseOutputBuffer(outputIndex, true); // render to surface
}
```

---

## 3. Codec2 Framework (Android 10+)

```
Codec2 replaces OMX (OpenMAX):
  Cleaner API, better buffer management
  Supports flexible buffer types
  
Stack:
  MediaCodec (Java)
    │ MediaCodecList → find appropriate codec
    ▼
  CCodec (Codec2 wrapper for MediaCodec)
    │ IComponentStore AIDL
    ▼
  Codec2 HAL (android.hardware.media.c2)
    │
    ├── libcodec2_soft_*.so  (SW codecs)
    └── libcodec2_hw_*.so    (HW codecs via vendor)
```

---

## 4. Pi4 Hardware Codec

```
BCM2711 VideoCore VI firmware codec:
  H.264 decode: up to 1080p60 hardware
  H.265 decode: up to 4K30 hardware (Pi4 8GB)
  MJPEG decode: hardware
  H.264 encode: hardware (limited)
  
Access via:
  V4L2 M2M (memory-to-memory) interface
  /dev/video10 → H264 decoder
  /dev/video11 → H265 decoder (if license enabled)
  /dev/video12 → MJPEG decoder
  
Android HAL:
  v4l2_codec2 (VideoCodec2) → wraps V4L2 M2M
  Connects Codec2 framework to V4L2 M2M
  
Note: Pi4 H.265 requires separate $10 license from Raspberry Pi
```

---

## 5. V4L2 Memory-to-Memory (M2M)

```
V4L2 M2M = bidirectional codec device:
  OUTPUT queue  ← App sends compressed data (H264 NALs)
  CAPTURE queue ← App reads decoded frames (YUV)

Both are V4L2 buffer queues:
  REQBUFS on OUTPUT → for compressed input
  REQBUFS on CAPTURE → for decoded output
  
  QBUF on OUTPUT → queue compressed frames
  DQBUF on CAPTURE → get decoded frames
```

```c
/* V4L2 M2M codec setup */
int fd = open("/dev/video10", O_RDWR);  /* H264 decoder */

/* Set compressed format on OUTPUT */
struct v4l2_format fmt_out = {
    .type = V4L2_BUF_TYPE_VIDEO_OUTPUT_MPLANE,
    .fmt.pix_mp = {
        .width = 1920, .height = 1080,
        .pixelformat = V4L2_PIX_FMT_H264,
    }
};
ioctl(fd, VIDIOC_S_FMT, &fmt_out);

/* Set YUV format on CAPTURE */
struct v4l2_format fmt_cap = {
    .type = V4L2_BUF_TYPE_VIDEO_CAPTURE_MPLANE,
    .fmt.pix_mp = {
        .pixelformat = V4L2_PIX_FMT_NV12,  /* YUV 4:2:0 */
    }
};
ioctl(fd, VIDIOC_S_FMT, &fmt_cap);
```

---

## 6. MediaExtractor — Demux

```java
// Extract streams from container (MP4, MKV, TS, ...)
MediaExtractor extractor = new MediaExtractor();
extractor.setDataSource("/sdcard/video.mp4");

// Find video track
for (int i = 0; i < extractor.getTrackCount(); i++) {
    MediaFormat format = extractor.getTrackFormat(i);
    String mime = format.getString(MediaFormat.KEY_MIME);
    if (mime.startsWith("video/")) {
        extractor.selectTrack(i);
        break;
    }
}

// Read samples
ByteBuffer buf = ByteBuffer.allocate(1024 * 1024);
int size = extractor.readSampleData(buf, 0);
long pts  = extractor.getSampleTime();
int flags = extractor.getSampleFlags();
extractor.advance();  // move to next sample
```

---

## 7. MediaPlayer (High-level API)

```java
// Simple playback
MediaPlayer player = new MediaPlayer();
player.setDataSource(context, Uri.parse("file:///sdcard/video.mp4"));
player.setSurface(surface);
player.prepare();  // or prepareAsync()
player.start();

// State machine:
// IDLE → INITIALIZED → PREPARED → STARTED
//                              → PAUSED
//                              → STOPPED → PREPARED (reset)
//                    → ERROR
//                    → END (release())
```

---

## 8. ExoPlayer (Modern, recommended)

```java
// ExoPlayer = Jetpack Media3 library
// More flexible than MediaPlayer, uses MediaCodec internally

ExoPlayer player = new ExoPlayer.Builder(context).build();
playerView.setPlayer(player);

MediaItem mediaItem = MediaItem.fromUri(
    Uri.parse("https://example.com/video.mp4"));
player.setMediaItem(mediaItem);
player.prepare();
player.play();

// DASH/HLS adaptive streaming
MediaItem adaptiveItem = MediaItem.Builder()
    .setUri("https://example.com/manifest.mpd")  // MPEG-DASH
    .build();

// Format selection (force SW/HW codec)
DefaultRenderersFactory factory = new DefaultRenderersFactory(context)
    .setExtensionRendererMode(EXTENSION_RENDERER_MODE_OFF); // SW fallback
```

---

## 9. Camera → Encode → File

```java
// Camera2 + MediaCodec → save video
// Setup encoder
MediaCodec encoder = MediaCodec.createEncoderByType("video/avc");
MediaFormat encFmt = MediaFormat.createVideoFormat("video/avc", 1920, 1080);
encFmt.setInteger(MediaFormat.KEY_BIT_RATE, 4_000_000);   // 4 Mbps
encFmt.setInteger(MediaFormat.KEY_FRAME_RATE, 30);
encFmt.setInteger(MediaFormat.KEY_I_FRAME_INTERVAL, 1);   // 1 sec GOP
encoder.configure(encFmt, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE);

// Get input surface (zero-copy from camera)
Surface encoderSurface = encoder.createInputSurface();
encoder.start();

// Point Camera2 to encoder surface
captureRequest.addTarget(encoderSurface);
// Frames flow: Camera HAL → encoder surface → encoded NALs
```

---

## 10. Debug Media

```bash
# List available codecs
adb shell dumpsys media.player | grep -i codec
adb shell dumpsys media.codec

# Codec2 components
adb shell dumpsys android.hardware.media.c2@1.0::IComponentStore/software

# V4L2 M2M devices on Pi4
adb shell v4l2-ctl --list-devices
# bcm2835-codec-decode (platform:bcm2835-codec):
#   /dev/video10  (H264)
#   /dev/video11  (H265/HEVC - if licensed)
#   /dev/video12  (MJPEG)
# bcm2835-codec-encode (platform:bcm2835-codec):
#   /dev/video21  (H264 encode)

# Test HW decode
adb shell v4l2-ctl -d /dev/video10 \
    --set-fmt-output=width=1920,height=1080,pixelformat=H264 \
    --stream-mmap

# Media stats (playback performance)
adb shell dumpsys media.player | grep -A5 "Video"

# Check dropped frames during playback
adb shell dumpsys gfxinfo <player_app> | grep "Janky"
```

---

## 11. File quan trọng trong source

| File | Mô tả |
|------|-------|
| `frameworks/av/media/codec2/` | Codec2 framework |
| `frameworks/av/media/libmedia/MediaCodec.cpp` | MediaCodec native |
| `frameworks/av/media/libstagefright/` | Stagefright media engine |
| `hardware/interfaces/media/c2/aidl/` | Codec2 AIDL HAL |
| `kernel/common/drivers/media/platform/bcm2835/bcm2835-codec.c` | Pi4 V4L2 codec |
| `device/brcm/rpi4/` → codec config | Pi4 codec HAL config |

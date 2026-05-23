# Bluetooth A2DP Audio Flow — Android 16 on Raspberry Pi 4

## Tổng quan

A2DP (Advanced Audio Distribution Profile) stream audio chất lượng cao từ Android đến tai nghe/loa Bluetooth. Khác với audio thông thường, A2DP có thêm bước **encode** và **truyền qua Bluetooth radio**.

## Codec hỗ trợ (theo độ ưu tiên)

| Priority | Codec | Bitrate | Đặc điểm |
|----------|-------|---------|----------|
| 1 | **Opus** | ~192kbps | Mới nhất, latency thấp |
| 2 | **LDAC** | 330/660/990kbps | Sony, hi-res |
| 3 | **aptX HD** | 576kbps | Qualcomm, 24-bit |
| 4 | **aptX** | 352kbps | Qualcomm, CD quality |
| 5 | **AAC** | ~250kbps | Apple devices |
| 6 | **SBC** | 328kbps | Fallback bắt buộc |

---

## Luồng đầy đủ

```
┌──────────────────────────────────────────────────────────────────┐
│  APP PROCESS                                                     │
│  AudioTrack / MediaPlayer / ExoPlayer                            │
└──────────────────────────────┬───────────────────────────────────┘
                               │ Shared memory (mmap) + Binder IPC
┌──────────────────────────────▼───────────────────────────────────┐
│  MEDIASERVER (AudioFlinger)                                      │
│                                                                  │
│  MixerThread → mix PCM → route đến Bluetooth output             │
│  BluetoothAudioStreamOut::write(pcm_buffer)                      │
│  ← audio_bluetooth_hw.so (BT audio module)                       │
│    stream_apis.cc::out_write()                                   │
└──────────────────────────────┬───────────────────────────────────┘
                               │ FMQ — Fast Message Queue
                               │ (AIDL shared ring buffer, zero-copy)
                               │ android.hardware.bluetooth.audio
┌──────────────────────────────▼───────────────────────────────────┐
│  COM.ANDROID.BLUETOOTH PROCESS (BT stack)                        │
│                                                                  │
│  BluetoothAudioClientInterface::ReadAudioData(buf, len)          │
│   → đọc PCM từ FMQ (DataMQ)                                     │
│   ← audio_hal_interface/aidl/a2dp/client_interface_aidl.cc       │
│                                                                  │
│  btif_a2dp_source_read_callback()  ← encoder gọi khi cần data   │
│   ← btif/src/btif_a2dp_source.cc                                 │
│                                                                  │
│  ┌── ENCODER THREAD (REAL_TIME priority) ─────────────────────┐  │
│  │  encoder_interface->encode_audio_data()                    │  │
│  │                                                            │  │
│  │  SBC  ← stack/a2dp/a2dp_sbc_encoder.cc                    │  │
│  │  AAC  ← stack/a2dp/a2dp_aac_encoder.cc                    │  │
│  │  aptX ← embdrv/encoder_for_aptx/                          │  │
│  │  LDAC ← stack/a2dp/a2dp_vendor_ldac_encoder.cc            │  │
│  │  Opus ← stack/a2dp/a2dp_vendor_opus_encoder.cc            │  │
│  │                                                            │  │
│  │  PCM raw → compressed frames                               │  │
│  └─────────────────────────┬──────────────────────────────────┘  │
│                             ↓                                    │
│  btif_a2dp_source_enqueue_callback()                             │
│   → tx_audio_queue (fixed_queue)                                 │
│   → bta_av_ci_src_data_ready()  ← signal BTA_AV                 │
│                                                                  │
│  BTA AV (bta/av/) → AVDT_WriteReq()                             │
│                                                                  │
│  AVDTP (stack/avdt/)                                             │
│   → RTP header + codec payload                                   │
│   → avdt_scb_hdl_write_req()                                     │
│                                                                  │
│  L2CAP (stack/l2cap/)                                            │
│   → L2CA_DataWrite(lcid, packet)                                 │
│                                                                  │
│  HCI → H4Protocol::Send(PacketType::ACL_DATA, payload)          │
└──────────────────────────────┬───────────────────────────────────┘
                               │ AF_BLUETOOTH socket (HCI_CHANNEL_USER)
┌──────────────────────────────▼───────────────────────────────────┐
│  LINUX KERNEL                                                    │
│  hci_uart driver → ttyAMA0 (UART 3Mbps)                         │
└──────────────────────────────┬───────────────────────────────────┘
                               │ UART → CYW43455
┌──────────────────────────────▼───────────────────────────────────┐
│  CYW43455 → Bluetooth 2.4GHz radio                               │
│  → Tai nghe / Loa Bluetooth decode SBC/AAC/aptX/LDAC/Opus        │
└──────────────────────────────────────────────────────────────────┘
```

---

## Cơ chế FMQ (Fast Message Queue)

```
AudioFlinger                        BT Stack
    │                                   │
    │  out_write(pcm, len)              │
    ↓                                   │
 [FMQ DataMQ]  ←── shared memory ───►  ReadAudioData()
    │                                   │
    │  không cần Binder IPC             │
    │  không có data copy (zero-copy)   │
    └─── chỉ update read/write ptr ─────┘
```

Khởi tạo tại `client_interface_aidl.cc:353`:
```cpp
provider_->startSession(stack_if, audio_config, latency_modes, &mq_desc);
data_mq.reset(new DataMQ(mq_desc));  // bind vào shared memory
```

---

## Hai chế độ Encoding

| | Software Encoding | Hardware Offload |
|---|---|---|
| Encode tại | CPU (Android host) | BT chip firmware |
| SessionType | `A2DP_SOFTWARE_ENCODING_DATAPATH` | `A2DP_HARDWARE_OFFLOAD_ENCODING_DATAPATH` |
| Data qua FMQ | PCM raw | Compressed |
| RPi4 | **Dùng SW encoding** | Không hỗ trợ |

---

## Protocol stack layers

```
App PCM → AudioFlinger mix → [FMQ] → Encoder (SBC/AAC/…)
  → AVDTP media packet [RTP-like header + encoded payload]
  → L2CAP frame        [channel ID + length + data]
  → HCI ACL packet     [handle + flags + length]
  → H4 UART frame      [type=0x02 + HCI payload]
  → UART → CYW43455 radio → Tai nghe
```

---

## Negotiation codec khi kết nối

1. SDP discovery — thiết bị kia expose A2DP sink capabilities
2. AVDTP SETCONFIG — Android đề xuất codec tốt nhất cả 2 hỗ trợ
3. `A2dpCodecConfig::createCodec()` khởi tạo encoder tương ứng
4. Codec có thể thay đổi trong quá trình stream (AVDTP Reconfigure)

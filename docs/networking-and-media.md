# Networking & Media Streaming

Transport and video delivery choices that show up in chat, live, and VOD designs.

← [README](../README.md) · [Docs index](./README.md)

---

## TCP vs UDP

| | **TCP** | **UDP** |
|--|---------|---------|
| Connection | Reliable stream, ordered | Datagrams, no guarantee |
| Delivery | Retransmit, congestion control | App handles loss / order |
| Latency | Head-of-line blocking can hurt | Can be lower for real-time |
| Typical use | HTTP/1.1, TLS APIs, most backends | Gaming, VoIP, custom real-time, DNS |
| With QUIC/HTTP3 | — | QUIC runs over UDP, adds reliability selectively |

**Interview defaults**

- APIs, DB clients, most services → **TCP**
- Live media / WebRTC media path → often **UDP** (with FEC/jitter buffer)
- Don’t say “UDP is faster” without “accepts loss / app-level recovery”

WebSockets and HTTP still ride **TCP** (or QUIC).

---

## HLS vs DASH

Adaptive bitrate **VOD / live-to-many** over HTTP.

| | **HLS** (Apple) | **DASH** (MPEG-DASH) |
|--|-----------------|----------------------|
| Manifest | `.m3u8` playlists | MPD (XML) |
| Segments | `.ts` or fMP4 | Usually fMP4 |
| Adoption | Native on Apple; widely supported | Android / open ecosystem; Widevine common |
| Delivery | Plain HTTP(S) → CDN-friendly | Same |
| Latency | Traditional HLS seconds+; LL-HLS improves | Similar; CMAF + chunked helps both |

**When to mention:** YouTube-like / course video / live at scale → encode ladder → package HLS/DASH → **CDN** → player switches bitrate by bandwidth.

Not ideal for ultra-low-latency interactive calls (use WebRTC).

---

## RTMP vs SRT

**Ingest** (creator → platform) more than viewer playback.

| | **RTMP** | **SRT** |
|--|----------|---------|
| Era | Classic OBS → media server | Modern contribution protocol |
| Transport | TCP | UDP-based, ARQ for recovery |
| Firewall / NAT | Sometimes painful | Generally better for messy networks |
| Latency | Low-ish ingest | Designed for reliable low-latency over lossy links |
| Status | Still common; Flash player dead | Growing for broadcast contribution |

Typical live pipeline:

```text
OBS --RTMP/SRT--> Ingest → Transcode → Package HLS/DASH → CDN → Viewers
```

Interactive 1:1 or small group: **WebRTC** (UDP, ICE, SFU/MCU) instead of HLS.

---

## Related networking prerequisites

- **TLS termination** at LB / gateway
- **HTTP/2, HTTP/3** — multiplexing; reduces connection count
- **gRPC** — HTTP/2 + protobuf; great service-to-service
- **WebRTC SFU vs MCU** — forward vs mix media (cost/CPU trade-off)
- **Anycast CDN** — same IP, route to nearest PoP
- **Sticky sessions** — needed for in-memory sessions / some WS setups; prefer externalize state

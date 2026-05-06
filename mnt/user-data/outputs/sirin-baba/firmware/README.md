# Firmware — ESP32 (Rust)

> Embedded firmware running on the ESP32 inside the plush toy. Captures audio from an I2S microphone, streams it to the backend over UDP, receives synthesized speech, and plays it through an I2S amplifier.
>
> Peluş oyuncağın içindeki ESP32'de çalışan gömülü yazılım. I2S mikrofondan ses kaydeder, UDP üzerinden backend'e gönderir, sentezlenmiş konuşmayı alır ve I2S amplifikatörden çalar.

---

## 🇬🇧 English

### Stack

| Component | Choice | Why |
|-----------|--------|-----|
| Language | Rust (no_std) | Memory safety, no GC pauses, predictable real-time behavior |
| HAL | [`esp-hal`](https://github.com/esp-rs/esp-hal) | First-class ESP32 support, actively maintained |
| Async runtime | [`embassy-rs`](https://embassy.dev) | Zero-cost async, proven on embedded targets |
| WiFi stack | [`esp-wifi`](https://github.com/esp-rs/esp-wifi) | Native Rust WiFi for ESP32 |
| Networking | [`smoltcp`](https://github.com/smoltcp-rs/smoltcp) | Pure-Rust TCP/IP stack with UDP support |
| Logger | `defmt` + `defmt-rtt` | Compact binary logs over RTT |

### Why Rust over Arduino C++ or ESP-IDF?

- **Compile-time guarantees** — A buffer overflow in the audio path could brick the device. Rust catches this at build time.
- **`async`/`.await` for concurrent I/O** — Recording audio, streaming UDP, and reading the button happen concurrently without an RTOS scheduler.
- **Cargo ecosystem** — Dependencies, testing, and tooling that just work.
- **Modern type system** — Encoding pin states, peripheral ownership, and DMA buffers in the type system catches whole classes of bugs.

### Module layout

```
src/
├── main.rs       # Entry point, embassy executor, task spawning
├── audio.rs      # I2S mic + speaker drivers, ring buffers
├── network.rs    # WiFi connection + UDP socket management
├── button.rs     # GPIO interrupt handler, debouncing
└── protocol.rs   # Wire format for audio packets (header + PCM payload)
```

### Audio pipeline

```
Mic (INMP441)  ──I2S──►  DMA buffer  ──►  UDP packets  ──►  Backend
                                                              │
Speaker (MAX98357A) ◄──I2S──  DMA buffer  ◄──  UDP packets ◄──┘
```

- **Sample rate**: 16 kHz mono (sufficient for speech, halves bandwidth vs 44.1 kHz)
- **Bit depth**: 16-bit signed PCM
- **Packet size**: 512 samples (~32ms of audio) per UDP datagram
- **Recording duration**: 3 seconds max per question (configurable)

### Pin assignments

| Function | GPIO | Notes |
|----------|------|-------|
| I2S Mic — SCK (BCLK) | 26 | INMP441 clock |
| I2S Mic — WS (LRCLK) | 25 | INMP441 word select |
| I2S Mic — SD (DATA) | 34 | Input only pin |
| I2S Speaker — BCLK | 27 | MAX98357A clock |
| I2S Speaker — LRC | 14 | MAX98357A word select |
| I2S Speaker — DIN | 22 | Audio data out |
| Button | 32 | Pull-up, falling edge interrupt |
| Status LED | 2 | Onboard LED, optional |

### Build & flash

Prerequisites:
- Rust 1.75+ with `rustup`
- `espup` for ESP toolchain — `cargo install espup && espup install`
- `cargo-espflash` — `cargo install cargo-espflash`

```bash
cd firmware

# Source the ESP environment (do this in every new shell)
. ~/export-esp.sh

# Configure WiFi credentials
cp config.example.toml config.toml
# Edit config.toml with WIFI_SSID, WIFI_PASS, BACKEND_IP

# Build (release mode is required — debug is too slow for 16kHz audio)
cargo build --release

# Flash & monitor
cargo espflash flash --release --monitor
```

### Configuration

Sensitive values live in `config.toml` (gitignored). Build script reads it via `build.rs` and bakes constants into the binary — no runtime parsing.

```toml
[wifi]
ssid = "YourNetwork"
password = "YourPassword"

[backend]
ip   = "192.168.1.125"   # Proxmox VM with backend
port = 9000

[audio]
recording_seconds = 3
```

### Memory & performance

| Resource | Budget | Notes |
|----------|--------|-------|
| Flash | ~600 KB | Comfortably under 4MB partition |
| RAM (heap) | ~80 KB | embassy + smoltcp + audio buffers |
| Peak CPU | ~40% | I2S DMA does the heavy lifting |
| Latency (mic → UDP send) | ~50 ms | Dominated by packet batching |

### Roadmap

- [ ] Bring up I2S mic with DMA, verify in serial
- [ ] Bring up I2S speaker, play a test tone
- [ ] WiFi association + UDP socket
- [ ] Full duplex audio loopback test (mic → backend → speaker)
- [ ] Button interrupt + state machine (idle → recording → waiting → playing)
- [ ] Power management (deep sleep when idle)
- [ ] OTA updates over WiFi

---

## 🇹🇷 Türkçe

### Yığın

| Bileşen | Tercih | Gerekçe |
|---------|--------|---------|
| Dil | Rust (no_std) | Bellek güvenliği, GC duraksamaları yok, öngörülebilir gerçek zamanlı davranış |
| HAL | [`esp-hal`](https://github.com/esp-rs/esp-hal) | Birinci sınıf ESP32 desteği, aktif geliştirilen |
| Async runtime | [`embassy-rs`](https://embassy.dev) | Sıfır maliyetli async, gömülü hedeflerde kanıtlanmış |
| WiFi stack | [`esp-wifi`](https://github.com/esp-rs/esp-wifi) | ESP32 için native Rust WiFi |
| Ağ | [`smoltcp`](https://github.com/smoltcp-rs/smoltcp) | UDP destekli saf Rust TCP/IP stack |
| Logger | `defmt` + `defmt-rtt` | RTT üzerinden kompakt binary loglar |

### Neden Arduino C++ veya ESP-IDF değil de Rust?

- **Derleme zamanı garantileri** — Ses yolunda bir buffer overflow cihazı bricklemeye yeter. Rust bunu derlerken yakalar.
- **Eşzamanlı I/O için `async`/`.await`** — Ses kaydı, UDP streaming ve buton okuma; RTOS scheduler'a ihtiyaç olmadan eşzamanlı çalışır.
- **Cargo ekosistemi** — Çalışan bağımlılıklar, test ve araçlar.
- **Modern tip sistemi** — Pin durumları, çevre birimi sahipliği ve DMA buffer'ları tip sisteminde kodlamak, koca bir bug sınıfını yakalar.

### Modül yapısı

```
src/
├── main.rs       # Giriş noktası, embassy executor, task spawn
├── audio.rs      # I2S mikrofon + hoparlör sürücüleri, ring buffer
├── network.rs    # WiFi bağlantısı + UDP soket yönetimi
├── button.rs     # GPIO interrupt handler, debounce
└── protocol.rs   # Ses paket formatı (header + PCM payload)
```

### Ses pipeline

```
Mikrofon (INMP441)  ──I2S──►  DMA buffer  ──►  UDP paket  ──►  Backend
                                                                  │
Hoparlör (MAX98357A) ◄──I2S──  DMA buffer  ◄──  UDP paket ◄───────┘
```

- **Örnekleme hızı**: 16 kHz mono (konuşma için yeterli, 44.1 kHz'a göre bant genişliğini yarıya indirir)
- **Bit derinliği**: 16-bit işaretli PCM
- **Paket boyutu**: UDP datagram başına 512 sample (~32ms ses)
- **Kayıt süresi**: Soru başına maksimum 3 saniye (yapılandırılabilir)

### Pin atamaları

| İşlev | GPIO | Notlar |
|-------|------|--------|
| I2S Mik — SCK (BCLK) | 26 | INMP441 clock |
| I2S Mik — WS (LRCLK) | 25 | INMP441 word select |
| I2S Mik — SD (DATA) | 34 | Sadece input pin |
| I2S Hoparlör — BCLK | 27 | MAX98357A clock |
| I2S Hoparlör — LRC | 14 | MAX98357A word select |
| I2S Hoparlör — DIN | 22 | Ses çıkış data |
| Buton | 32 | Pull-up, falling edge interrupt |
| Durum LED'i | 2 | Onboard LED, opsiyonel |

### Derleme & yükleme

Önkoşullar:
- `rustup` ile Rust 1.75+
- ESP toolchain için `espup` — `cargo install espup && espup install`
- `cargo-espflash` — `cargo install cargo-espflash`

```bash
cd firmware

# ESP ortamını yükle (her yeni shell'de bunu yap)
. ~/export-esp.sh

# WiFi bilgilerini ayarla
cp config.example.toml config.toml
# config.toml içine WIFI_SSID, WIFI_PASS, BACKEND_IP yaz

# Derle (release modu zorunlu — debug 16kHz ses için çok yavaş)
cargo build --release

# Yükle ve monitor et
cargo espflash flash --release --monitor
```

### Yapılandırma

Hassas değerler `config.toml` içinde (gitignore'da). Build script `build.rs` üzerinden okur ve sabitleri binary'ye gömer — runtime parsing yok.

```toml
[wifi]
ssid = "AgInizinAdi"
password = "Sifreniz"

[backend]
ip   = "192.168.1.125"   # Backend'in çalıştığı Proxmox VM
port = 9000

[audio]
recording_seconds = 3
```

### Bellek & performans

| Kaynak | Bütçe | Notlar |
|--------|-------|--------|
| Flash | ~600 KB | 4MB partition'da rahat |
| RAM (heap) | ~80 KB | embassy + smoltcp + ses buffer'ları |
| Tepe CPU | ~40% | Ağır işi I2S DMA yapıyor |
| Gecikme (mik → UDP) | ~50 ms | Çoğunluğu paket batchleme |

### Yol haritası

- [ ] DMA ile I2S mikrofon ayağa kaldır, seri portta doğrula
- [ ] I2S hoparlörü ayağa kaldır, test tonu çal
- [ ] WiFi bağlantısı + UDP soket
- [ ] Tam duplex ses loopback testi (mik → backend → hoparlör)
- [ ] Buton interrupt + state machine (boşta → kayıt → bekleme → çalma)
- [ ] Güç yönetimi (boştayken deep sleep)
- [ ] WiFi üzerinden OTA güncellemeler

# 🍄 Şirin Baba — Conversational AI Toy for Toddlers

> A voice-interactive Papa Smurf plush toy that answers a 3-year-old's questions with pedagogically-tuned responses, powered by a local LLM. Built with Rust (ESP32 firmware) and Python (AI backend).
>
> 3 yaşındaki bir çocuğun sorularına pedagojik olarak ayarlanmış cevaplar veren, sesle etkileşimli Şirin Baba peluş oyuncağı. Yerel LLM ile çalışır. Rust (ESP32 firmware) ve Python (AI backend) ile geliştirilmiştir.

[![Status](https://img.shields.io/badge/status-in_development-orange)]()
[![License](https://img.shields.io/badge/license-MIT-blue.svg)]()
[![Rust](https://img.shields.io/badge/Rust-embassy--rs-orange?logo=rust)]()
[![Python](https://img.shields.io/badge/Python-3.11+-blue?logo=python)]()
[![Ollama](https://img.shields.io/badge/Ollama-Qwen2.5-green)]()

---

<details open>
<summary><b>🇬🇧 English</b></summary>

## Overview

Şirin Baba ("Papa Smurf" in Turkish) is an open-source hardware project that turns a plush toy into a child-friendly conversational AI companion. The system listens through a microphone embedded in the plush, processes the child's question through a locally-hosted LLM with a carefully designed pedagogical prompt, and responds in a warm Turkish voice — all without sending any data to the cloud.

This project is built for my own child, with privacy and developmental appropriateness as first-class design goals.

### Why this project?

Most "smart toys" on the market either rely on cloud APIs (privacy concerns), give wildly inappropriate responses to children (no pedagogical filter), or require a screen (which I want to avoid for a 3-year-old). Şirin Baba addresses all three:

- **Fully offline AI** — Local Whisper STT + local LLM + local Piper TTS. No data leaves the home network.
- **Pedagogical system prompt** — The LLM is constrained to respond at an A1-Turkish vocabulary level, with 2-sentence maximum, ending with curiosity-promoting questions.
- **Screen-free** — The interface is the plush toy itself. A button on the belly, a microphone in the hat, a speaker in the body.

### Architecture

```
┌─────────────────┐         UDP/PCM          ┌──────────────────┐
│   ESP32 (Rust)  │ ◄──────────────────────► │  Python Backend  │
│                 │                           │                  │
│  • I2S Mic      │   1. Audio in            │  • Whisper STT   │
│  • I2S Speaker  │   2. STT → Text          │  • Ollama LLM    │
│  • WiFi (UDP)   │   3. LLM → Response      │  • Piper TTS     │
│  • Push Button  │   4. TTS → Audio         │  • FastAPI       │
└─────────────────┘                           └──────────────────┘
       │                                              │
   Plush toy                              Proxmox VM (RTX A2000)
```

### Tech Stack

| Layer | Technology | Reason |
|-------|------------|--------|
| Firmware | Rust + `embassy-rs` + `esp-hal` | Memory safety, async I/O, low-level I2S control |
| Communication | UDP over WiFi | Low-latency audio streaming, no TCP overhead |
| Speech-to-Text | `faster-whisper` (Turkish) | High accuracy, GPU-accelerated, runs locally |
| LLM | Qwen2.5 3B via Ollama | Fast inference on 6GB VRAM, good Turkish support |
| Text-to-Speech | Piper TTS (Turkish voice) | Fully offline, sub-second synthesis |
| Backend | Python 3.11 + FastAPI + asyncio | Async pipeline orchestration |
| Hosting | Proxmox VM (Ubuntu) | Existing home lab infrastructure |

### Hardware

- ESP32-WROOM (38-pin)
- INMP441 I2S MEMS microphone
- MAX98357A I2S audio amplifier
- 4Ω 3W speaker
- Push button + LiPo battery + charge module
- Existing Papa Smurf plush toy (the soul of the project)

### Pedagogical Design

The system prompt enforces principles drawn from early childhood development research:

- **Vocabulary ceiling** — Turkish A1 level (~500 most common words)
- **Length constraint** — Maximum 2 short sentences per response
- **Curiosity loops** — Every answer ends with an open-ended question to the child
- **Specific praise** — "You figured that out yourself!" instead of generic "Good job!"
- **Concrete metaphors** — Abstract concepts grounded in familiar objects
- **Emotional validation** — Recognizes feelings before redirecting

### Project Roadmap

- [x] System architecture design
- [x] Component selection
- [ ] **Week 1** — Rust firmware: I2S audio capture + UDP streaming
- [ ] **Week 2** — Python backend: STT pipeline + Ollama integration
- [ ] **Week 3** — Piper TTS + pedagogical prompt fine-tuning
- [ ] **Week 4** — Physical assembly + battery integration + field testing
- [ ] **Future** — Conversation memory, multiple voice personas, parental dashboard

### Project Status

🚧 **Development starts June 2026.** This repository is currently a planning artifact — code, schematics, and assembly guides will land progressively. Star the repo to follow along.

</details>

---

<details open>
<summary><b>🇹🇷 Türkçe</b></summary>

## Genel Bakış

Şirin Baba, peluş oyuncağı çocuk dostu bir konuşan yapay zeka arkadaşına dönüştüren açık kaynaklı bir donanım projesidir. Sistem, peluşa gömülü mikrofon aracılığıyla çocuğu dinler, sorusunu pedagojik olarak özenle hazırlanmış bir prompt ile yerel olarak çalışan bir LLM'den geçirir ve sıcak bir Türkçe sesle yanıt verir — hiçbir veri buluta gönderilmez.

Bu projeyi kendi çocuğum için geliştiriyorum; gizlilik ve gelişimsel uygunluk en önemli tasarım hedefleri.

### Neden bu proje?

Piyasadaki "akıllı oyuncakların" çoğu ya bulut API'lerine bağımlı (gizlilik sorunu), ya çocuklara uygunsuz cevaplar veriyor (pedagojik filtre yok), ya da ekran gerektiriyor (3 yaşındaki bir çocuk için istemediğim bir şey). Şirin Baba bu üç sorunu da çözer:

- **Tamamen offline AI** — Yerel Whisper STT + yerel LLM + yerel Piper TTS. Hiçbir veri ev ağının dışına çıkmaz.
- **Pedagojik sistem promptu** — LLM, A1 seviye Türkçe kelime dağarcığıyla, en fazla 2 cümle ile, çocuğu meraka yönlendiren sorularla bitiren cevaplar vermeye zorlanır.
- **Ekransız** — Arayüz, peluş oyuncağın kendisi. Karında bir buton, şapkada bir mikrofon, gövdede bir hoparlör.

### Mimari

```
┌─────────────────┐         UDP/PCM          ┌──────────────────┐
│   ESP32 (Rust)  │ ◄──────────────────────► │  Python Backend  │
│                 │                           │                  │
│  • I2S Mikrofon │   1. Ses girişi          │  • Whisper STT   │
│  • I2S Hoparlör │   2. STT → Metin         │  • Ollama LLM    │
│  • WiFi (UDP)   │   3. LLM → Cevap         │  • Piper TTS     │
│  • Buton        │   4. TTS → Ses           │  • FastAPI       │
└─────────────────┘                           └──────────────────┘
       │                                              │
  Peluş oyuncak                          Proxmox VM (RTX A2000)
```

### Teknoloji Yığını

| Katman | Teknoloji | Gerekçe |
|--------|-----------|---------|
| Firmware | Rust + `embassy-rs` + `esp-hal` | Bellek güvenliği, async I/O, düşük seviye I2S kontrolü |
| İletişim | WiFi üzerinden UDP | Düşük gecikmeli ses akışı, TCP overhead yok |
| Konuşma → Metin | `faster-whisper` (Türkçe) | Yüksek doğruluk, GPU ivmeli, yerel çalışır |
| LLM | Ollama üzerinden Qwen2.5 3B | 6GB VRAM'da hızlı çıkarım, iyi Türkçe desteği |
| Metin → Konuşma | Piper TTS (Türkçe ses) | Tamamen offline, saniye altı sentez |
| Backend | Python 3.11 + FastAPI + asyncio | Async pipeline orkestrasyon |
| Barındırma | Proxmox VM (Ubuntu) | Mevcut ev lab altyapısı |

### Donanım

- ESP32-WROOM (38-pin)
- INMP441 I2S MEMS mikrofon
- MAX98357A I2S ses amplifikatörü
- 4Ω 3W hoparlör
- Buton + LiPo pil + şarj modülü
- Mevcut Şirin Baba peluş oyuncak (projenin ruhu)

### Pedagojik Tasarım

Sistem promptu, erken çocukluk gelişimi araştırmalarından alınan ilkeleri uygular:

- **Kelime tavanı** — Türkçe A1 seviyesi (~500 en yaygın kelime)
- **Uzunluk kısıtı** — Cevap başına en fazla 2 kısa cümle
- **Merak döngüleri** — Her cevap, çocuğa açık uçlu bir soruyla biter
- **Spesifik övgü** — Genel "Aferin!" yerine "Bunu kendin buldun!"
- **Somut metaforlar** — Soyut kavramlar tanıdık nesnelerle açıklanır
- **Duygusal doğrulama** — Yönlendirmeden önce duygular tanınır

### Proje Yol Haritası

- [x] Sistem mimarisi tasarımı
- [x] Bileşen seçimi
- [ ] **Hafta 1** — Rust firmware: I2S ses kaydı + UDP streaming
- [ ] **Hafta 2** — Python backend: STT pipeline + Ollama entegrasyonu
- [ ] **Hafta 3** — Piper TTS + pedagojik prompt ince ayarı
- [ ] **Hafta 4** — Fiziksel montaj + pil entegrasyonu + saha testi
- [ ] **Gelecek** — Konuşma hafızası, çoklu ses personaları, ebeveyn paneli

### Proje Durumu

🚧 **Geliştirme Haziran 2026'da başlıyor.** Bu repo şu anda bir planlama eseri — kod, şemalar ve montaj rehberleri zamanla yüklenecek. Takip etmek için repoya yıldız ver.

</details>

---

## Repository Structure

```
sirin-baba/
├── firmware/              # Rust firmware for ESP32
│   ├── Cargo.toml
│   ├── src/
│   │   ├── main.rs        # embassy async runtime entry
│   │   ├── audio.rs       # I2S mic + speaker drivers
│   │   ├── network.rs     # WiFi + UDP streaming
│   │   └── button.rs      # GPIO interrupt handler
│   └── README.md
│
├── backend/               # Python AI backend
│   ├── pyproject.toml
│   ├── src/
│   │   ├── main.py        # FastAPI + UDP server
│   │   ├── stt.py         # faster-whisper wrapper
│   │   ├── llm.py         # Ollama client + prompt
│   │   ├── tts.py         # Piper TTS wrapper
│   │   └── pedagogy.py    # System prompt + safety filters
│   ├── prompts/
│   │   └── sirin_baba.md  # The pedagogical prompt
│   └── README.md
│
├── hardware/              # Schematics, BOM, assembly
│   ├── wiring-diagram.png
│   ├── bom.md             # Bill of materials
│   └── assembly-guide.md
│
├── docs/                  # Project documentation
│   ├── architecture.md
│   ├── pedagogical-design.md
│   └── demo.gif
│
└── README.md              # You are here
```

## Contributing

This is a personal hands-on project, but ideas, pedagogy critiques, and Turkish language reviews are very welcome — open an issue! / Bu kişisel bir proje, ancak fikirler, pedagoji eleştirileri ve Türkçe dil incelemeleri memnuniyetle karşılanır — issue açabilirsiniz!

## License

MIT — see [LICENSE](LICENSE).

## Author

**Fatih Bayram** — Workshop Service Technician → MLOps Engineer in transition.
[GitHub](https://github.com/fatihhbayramm) · [LinkedIn](https://linkedin.com/in/fatihhbayramm) · [Medium](https://medium.com/@fatihhbayramm)

---

<sub>Built with ☕ in Istanbul · Made for my kid · Inspired by every parent who's ever been asked "but why?" 47 times in a row.</sub>

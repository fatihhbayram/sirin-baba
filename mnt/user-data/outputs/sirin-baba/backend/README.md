# Backend — Python AI Pipeline

> The brain. Receives audio packets from the ESP32, transcribes them, generates a pedagogically appropriate response with a local LLM, synthesizes Turkish speech, and streams it back to the toy.
>
> Beyin. ESP32'den ses paketleri alır, metne çevirir, yerel LLM ile pedagojik olarak uygun bir cevap üretir, Türkçe konuşma sentezler ve oyuncağa geri gönderir.

---

## 🇬🇧 English

### Stack

| Component | Choice | Why |
|-----------|--------|-----|
| Runtime | Python 3.11+ asyncio | Pipeline is mostly I/O-bound; async is a natural fit |
| HTTP / WS | FastAPI | Fast, typed, ergonomic, great docs |
| UDP server | `asyncio.DatagramProtocol` | Standard library, no extra deps |
| STT | [`faster-whisper`](https://github.com/SYSTRAN/faster-whisper) | 4× faster than openai-whisper, GPU accelerated |
| LLM | [Ollama](https://ollama.com) (Qwen2.5 3B) | Local, no API keys, ~2GB VRAM |
| TTS | [Piper](https://github.com/rhasspy/piper) (Turkish voice) | Fully offline, sub-second synthesis |
| Validation | Pydantic v2 | Config + request/response models |
| Logging | `structlog` | Structured JSON logs for observability |
| Testing | pytest + pytest-asyncio | Standard async test setup |

### Architecture

```
                  ┌─────────────────────────────────────────────┐
                  │             Python Backend                   │
                  │                                              │
   UDP audio in   │   ┌─────────┐    ┌──────────┐    ┌──────┐   │   UDP audio out
  ─────────────►  │──►│ Whisper │───►│  Ollama  │───►│Piper │──►│──────────────►
                  │   │   STT   │    │   LLM    │    │ TTS  │   │
                  │   └─────────┘    └──────────┘    └──────┘   │
                  │        │              │                       │
                  │        └──────────────┴────► structured logs  │
                  │                                                │
                  │   ┌──────────────────────────────────────────┐│
                  │   │  Pedagogical layer: prompt + safety      ││
                  │   │  filters + child-appropriateness check   ││
                  │   └──────────────────────────────────────────┘│
                  └─────────────────────────────────────────────┘
```

### Module layout

```
src/
├── main.py          # FastAPI app + UDP server entrypoint
├── config.py        # Pydantic settings, env vars
├── audio.py         # PCM buffer assembly, WAV encoding
├── stt.py           # faster-whisper async wrapper
├── llm.py           # Ollama client, retry logic
├── tts.py           # Piper subprocess wrapper, audio streaming
├── pedagogy.py      # System prompt builder, safety filters
├── pipeline.py      # End-to-end orchestration
└── server.py        # UDP DatagramProtocol implementation

prompts/
└── sirin_baba.md    # The pedagogical system prompt (versioned!)

tests/
├── test_pedagogy.py     # Prompt evaluation tests
├── test_pipeline.py     # End-to-end mocked pipeline
└── fixtures/
    └── child_voice/     # Sample audio for STT regression tests
```

### Setup

Prerequisites:
- Python 3.11+
- [Ollama installed and running](https://ollama.com/download)
- NVIDIA GPU recommended (RTX A2000 6GB tested)

```bash
cd backend

# Create virtual environment
python -m venv .venv
source .venv/bin/activate

# Install dependencies (using uv for speed)
pip install uv
uv pip install -e ".[dev]"

# Download the LLM (one-time)
ollama pull qwen2.5:3b

# Download Piper Turkish voice (one-time)
mkdir -p models/piper
cd models/piper
wget https://huggingface.co/rhasspy/piper-voices/resolve/main/tr/tr_TR/dfki-medium/tr_TR-dfki-medium.onnx
wget https://huggingface.co/rhasspy/piper-voices/resolve/main/tr/tr_TR/dfki-medium/tr_TR-dfki-medium.onnx.json
cd ../..

# Configure
cp .env.example .env
# Edit .env with model paths and ports

# Run
python -m src.main
```

### Pipeline performance budget

End-to-end latency target: **< 3 seconds** from button release to first audio sample. A 3-year-old won't wait longer.

| Stage | Target | Notes |
|-------|--------|-------|
| Audio receive | ~100 ms | UDP packet assembly |
| STT (Whisper) | ~500 ms | int8 quant, GPU |
| LLM (Qwen2.5 3B) | ~1500 ms | ~80 token output, streaming |
| TTS (Piper) | ~400 ms | Streamable, first chunk fast |
| Network return | ~100 ms | UDP packets |
| **Total** | **~2.6 s** | |

### Configuration

`.env` file (gitignored, never commit):

```bash
# Server
HOST=0.0.0.0
HTTP_PORT=8000
UDP_PORT=9000

# STT
WHISPER_MODEL=small             # tiny | base | small | medium
WHISPER_DEVICE=cuda              # cuda | cpu
WHISPER_COMPUTE_TYPE=int8_float16

# LLM
OLLAMA_HOST=http://localhost:11434
OLLAMA_MODEL=qwen2.5:3b

# TTS
PIPER_MODEL_PATH=./models/piper/tr_TR-dfki-medium.onnx
PIPER_SAMPLE_RATE=22050

# Logging
LOG_LEVEL=INFO
LOG_FORMAT=json
```

### Testing the pedagogical prompt

The prompt isn't code — it's the heart of the project. We treat it as a versioned artifact and write evaluation tests for it.

```bash
# Run prompt evals against a fixture set of expected child questions
pytest tests/test_pedagogy.py -v

# Sample test
def test_response_length_max_two_sentences():
    response = llm_query("Gökyüzü neden mavi?")
    sentence_count = len(re.split(r'[.!?]+', response.strip())) - 1
    assert sentence_count <= 2, f"Too long: {response}"

def test_response_ends_with_question():
    response = llm_query("At nedir?")
    assert response.strip().endswith('?'), f"Missing curiosity hook: {response}"
```

### Privacy & data handling

- **No persistent audio storage** — Recordings are processed in memory and discarded
- **No conversation logs by default** — Only enabled via `DEBUG_LOG_CONVERSATIONS=true` for development
- **No external API calls** — Everything runs locally, the backend has no internet egress requirement after model download
- **Logs scrub PII** — Child's name (if mentioned) is hashed before logging

### Roadmap

- [ ] UDP server skeleton, audio packet reassembly
- [ ] Whisper STT integration, Turkish-only mode
- [ ] Ollama client + first version of pedagogy prompt
- [ ] Piper TTS integration, audio streaming back to ESP32
- [ ] End-to-end latency profiling
- [ ] Prompt eval suite with 30+ child question fixtures
- [ ] Conversation memory (last N turns) — opt-in
- [ ] Parental dashboard (FastAPI + simple HTML) — see daily summaries

---

## 🇹🇷 Türkçe

### Yığın

| Bileşen | Tercih | Gerekçe |
|---------|--------|---------|
| Runtime | Python 3.11+ asyncio | Pipeline çoğunlukla I/O-bound; async doğal uyum |
| HTTP / WS | FastAPI | Hızlı, tipli, ergonomik, harika dokümanlar |
| UDP server | `asyncio.DatagramProtocol` | Standart kütüphane, ek bağımlılık yok |
| STT | [`faster-whisper`](https://github.com/SYSTRAN/faster-whisper) | openai-whisper'dan 4× hızlı, GPU ivmeli |
| LLM | [Ollama](https://ollama.com) (Qwen2.5 3B) | Yerel, API key yok, ~2GB VRAM |
| TTS | [Piper](https://github.com/rhasspy/piper) (Türkçe ses) | Tamamen offline, saniye altı sentez |
| Doğrulama | Pydantic v2 | Config + request/response modelleri |
| Loglama | `structlog` | Observability için yapılandırılmış JSON loglar |
| Test | pytest + pytest-asyncio | Standart async test kurulumu |

### Mimari

```
                  ┌─────────────────────────────────────────────┐
                  │             Python Backend                    │
                  │                                              │
   UDP ses gelir  │   ┌─────────┐    ┌──────────┐    ┌──────┐   │   UDP ses çıkar
  ─────────────►  │──►│ Whisper │───►│  Ollama  │───►│Piper │──►│──────────────►
                  │   │   STT   │    │   LLM    │    │ TTS  │   │
                  │   └─────────┘    └──────────┘    └──────┘   │
                  │        │              │                       │
                  │        └──────────────┴────► yapılandırılmış  │
                  │                                  loglar        │
                  │   ┌──────────────────────────────────────────┐│
                  │   │  Pedagojik katman: prompt + güvenlik     ││
                  │   │  filtreleri + çocuğa uygunluk kontrolü   ││
                  │   └──────────────────────────────────────────┘│
                  └─────────────────────────────────────────────┘
```

### Modül yapısı

```
src/
├── main.py          # FastAPI app + UDP server giriş noktası
├── config.py        # Pydantic ayarları, env değişkenleri
├── audio.py         # PCM buffer birleştirme, WAV encoding
├── stt.py           # faster-whisper async wrapper
├── llm.py           # Ollama client, retry mantığı
├── tts.py           # Piper subprocess wrapper, ses streaming
├── pedagogy.py      # Sistem prompt builder, güvenlik filtreleri
├── pipeline.py      # Uçtan uca orkestrasyon
└── server.py        # UDP DatagramProtocol implementasyonu

prompts/
└── sirin_baba.md    # Pedagojik sistem prompt (versiyonlu!)

tests/
├── test_pedagogy.py     # Prompt değerlendirme testleri
├── test_pipeline.py     # Mock'lanmış uçtan uca pipeline
└── fixtures/
    └── child_voice/     # STT regresyon testleri için örnek ses
```

### Kurulum

Önkoşullar:
- Python 3.11+
- [Ollama kurulu ve çalışır](https://ollama.com/download)
- NVIDIA GPU önerilir (RTX A2000 6GB test edildi)

```bash
cd backend

# Sanal ortam oluştur
python -m venv .venv
source .venv/bin/activate

# Bağımlılıkları kur (hız için uv ile)
pip install uv
uv pip install -e ".[dev]"

# LLM'i indir (tek seferlik)
ollama pull qwen2.5:3b

# Piper Türkçe sesini indir (tek seferlik)
mkdir -p models/piper
cd models/piper
wget https://huggingface.co/rhasspy/piper-voices/resolve/main/tr/tr_TR/dfki-medium/tr_TR-dfki-medium.onnx
wget https://huggingface.co/rhasspy/piper-voices/resolve/main/tr/tr_TR/dfki-medium/tr_TR-dfki-medium.onnx.json
cd ../..

# Yapılandır
cp .env.example .env
# .env'yi model yolları ve portlarla düzenle

# Çalıştır
python -m src.main
```

### Pipeline performans bütçesi

Uçtan uca gecikme hedefi: **< 3 saniye** (buton bırakıldığından ilk ses sample'ına kadar). 3 yaşındaki bir çocuk daha fazla beklemez.

| Aşama | Hedef | Notlar |
|-------|-------|--------|
| Ses alımı | ~100 ms | UDP paket birleştirme |
| STT (Whisper) | ~500 ms | int8 quant, GPU |
| LLM (Qwen2.5 3B) | ~1500 ms | ~80 token çıktı, streaming |
| TTS (Piper) | ~400 ms | Streamable, ilk chunk hızlı |
| Geri dönüş ağı | ~100 ms | UDP paketler |
| **Toplam** | **~2.6 sn** | |

### Yapılandırma

`.env` dosyası (gitignore'da, asla commit etme):

```bash
# Server
HOST=0.0.0.0
HTTP_PORT=8000
UDP_PORT=9000

# STT
WHISPER_MODEL=small             # tiny | base | small | medium
WHISPER_DEVICE=cuda              # cuda | cpu
WHISPER_COMPUTE_TYPE=int8_float16

# LLM
OLLAMA_HOST=http://localhost:11434
OLLAMA_MODEL=qwen2.5:3b

# TTS
PIPER_MODEL_PATH=./models/piper/tr_TR-dfki-medium.onnx
PIPER_SAMPLE_RATE=22050

# Loglama
LOG_LEVEL=INFO
LOG_FORMAT=json
```

### Pedagojik prompt'u test etmek

Prompt sadece kod değil — projenin kalbi. Onu versiyonlu bir artifact olarak ele alıyor ve değerlendirme testleri yazıyoruz.

```bash
# Beklenen çocuk sorularından oluşan fixture seti üzerinde prompt eval'leri çalıştır
pytest tests/test_pedagogy.py -v

# Örnek test
def test_cevap_uzunlugu_en_fazla_iki_cumle():
    cevap = llm_sorgu("Gökyüzü neden mavi?")
    cumle_sayisi = len(re.split(r'[.!?]+', cevap.strip())) - 1
    assert cumle_sayisi <= 2, f"Çok uzun: {cevap}"

def test_cevap_soruyla_biter():
    cevap = llm_sorgu("At nedir?")
    assert cevap.strip().endswith('?'), f"Merak kancası eksik: {cevap}"
```

### Gizlilik & veri yönetimi

- **Kalıcı ses depolama yok** — Kayıtlar bellekte işlenir ve silinir
- **Varsayılan olarak konuşma logu yok** — Sadece geliştirme için `DEBUG_LOG_CONVERSATIONS=true` ile açılabilir
- **Dış API çağrısı yok** — Her şey yerel çalışır; model indirildikten sonra backend'in internete çıkması gerekmez
- **Loglar PII'yi temizler** — Çocuğun adı (geçerse) loglamadan önce hash'lenir

### Yol haritası

- [ ] UDP server iskeleti, ses paket birleştirme
- [ ] Whisper STT entegrasyonu, Türkçe-only mod
- [ ] Ollama client + pedagoji prompt'unun ilk versiyonu
- [ ] Piper TTS entegrasyonu, ESP32'ye ses streaming
- [ ] Uçtan uca gecikme profili
- [ ] 30+ çocuk sorusu fixture'lı prompt eval suite
- [ ] Konuşma hafızası (son N tur) — opt-in
- [ ] Ebeveyn paneli (FastAPI + sade HTML) — günlük özet

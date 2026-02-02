# 🔐 TimedBox

**Zero-Knowledge Encrypted File Transfer**

[![License: AGPL-3.0](https://img.shields.io/badge/License-AGPL--3.0-blue.svg)](https://www.gnu.org/licenses/agpl-3.0)
[![Node.js](https://img.shields.io/badge/Node.js-18%2B-green.svg)](https://nodejs.org/)
[![Docker](https://img.shields.io/badge/Docker-Ready-blue.svg)](https://www.docker.com/)

Send files securely with true end-to-end encryption. Files are encrypted in your browser before transmission — the server never sees your data.

**🌐 Live Demo:** [timedbox.app](https://timedbox.app)

---

## ✨ Features

| Feature | Description |
|---------|-------------|
| 🔐 **E2E Encryption** | AES-256-GCM encryption happens in your browser |
| 🛡️ **Zero Knowledge** | Server relays encrypted bytes only — cannot decrypt |
| 📦 **Zero Storage** | Files are relayed, never stored on disk |
| 🔑 **Split Code** | 6-digit session ID + 4-char key = server can't derive encryption key |
| ⏱️ **Auto-Destruct** | Sessions expire in 5 minutes (extendable once) |
| 🚫 **No Accounts** | No signup, no email, no tracking, no cookies |
| 📱 **Mobile Ready** | Responsive design, works on any device |
| 🌐 **Open Source** | Audit the code yourself, or self-host |

---

## 🔒 How the Encryption Works

### The Problem with "Zero Storage" Claims

Most "secure" file transfer services claim they don't store your files, but the server still **sees your data in cleartext** during transmission. You're trusting them not to peek.

### TimedBox's Solution: True Zero-Knowledge

```
┌─────────────────────────────────────────────────────────────────────┐
│  SENDER'S BROWSER                                                   │
│  ┌─────────────┐    ┌──────────────┐    ┌─────────────────────┐    │
│  │ Your File   │ -> │ AES-256-GCM  │ -> │ Encrypted Bytes     │    │
│  │ (cleartext) │    │ Encryption   │    │ (unreadable)        │    │
│  └─────────────┘    └──────────────┘    └─────────────────────┘    │
│                            ↑                       │                │
│                     Encryption Key                 │                │
│                     (from full code)               │                │
└────────────────────────────┼───────────────────────┼────────────────┘
                             │                       │
                             │                       ▼
┌────────────────────────────┼───────────────────────────────────────┐
│  SERVER (Zero Knowledge)   │                                        │
│                            │    ┌─────────────────────┐            │
│  Server knows: 123-456     │    │ Encrypted Bytes     │            │
│  Server doesn't know: ABCD │    │ ??????????????????  │            │
│  Server CANNOT decrypt     │    │ (server can't read) │            │
│                            │    └─────────────────────┘            │
└────────────────────────────┼───────────────────────┼───────────────┘
                             │                       │
                             │                       ▼
┌────────────────────────────┼───────────────────────────────────────┐
│  RECEIVER'S BROWSER        │                                        │
│                     Encryption Key                 │                │
│                     (from full code)               │                │
│                            ↓                       ▼                │
│  ┌─────────────────────┐    ┌──────────────┐    ┌─────────────┐    │
│  │ Encrypted Bytes     │ -> │ AES-256-GCM  │ -> │ Your File   │    │
│  │ (unreadable)        │    │ Decryption   │    │ (cleartext) │    │
│  └─────────────────────┘    └──────────────┘    └─────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### The Split Code Mechanism

When you send a file, you get a **10-character code**:

```
123-456-ABCD
└──┬──┘ └─┬─┘
   │      │
   │      └── CLIENT-GENERATED (4 chars)
   │          • Generated in your browser
   │          • Part of encryption key derivation
   │          • Server NEVER sees this
   │
   └── SERVER-GENERATED (6 digits)
       • Used for session routing
       • Server knows this
       • NOT sufficient to decrypt
```

**The encryption key is derived from the FULL code:**
```
PBKDF2-SHA256(fullCode + salt, 100000 iterations) → AES-256 Key
```

Since the server only knows `123-456` but the key requires `123456ABCD`, the server **mathematically cannot** derive the decryption key.

---

## 🚀 Quick Start

### Using Docker (Recommended)

```bash
# Clone the repository
git clone https://github.com/rosszc91/timedbox.git
cd timedbox

# Build and run
docker build -t timedbox .
docker run -d -p 3000:3000 --name timedbox timedbox

# Access at http://localhost:3000
```

### Using Node.js

```bash
# Clone the repository
git clone https://github.com/rosszc91/timedbox.git
cd timedbox

# Install dependencies
npm install

# Start the server
npm start

# Access at http://localhost:3000
```

### Using Docker Compose

```yaml
version: '3.8'
services:
  timedbox:
    build: .
    ports:
      - "3000:3000"
    environment:
      - PORT=3000
      - ALERTS_WEBHOOK=https://discord.com/api/webhooks/...
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

---

## 📊 API Endpoints

### Health Check
```
GET /api/health
```

Response:
```json
{
  "status": "ok",
  "version": "5.0.0",
  "encryption": "AES-256-GCM",
  "keyDerivation": "PBKDF2-SHA256",
  "zeroKnowledge": true,
  "activeSessions": 0,
  "activeTransfers": 0,
  "totalTransfers": 42,
  "e2eTransfers": 42,
  "bytesRelayed": 1073741824,
  "uptime": 86400,
  "activeConnections": 2
}
```

---

## 🔧 Configuration

Environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3000` | Server port |
| `ALERTS_WEBHOOK` | — | Discord webhook URL for alerts |

---

## 🔐 Cryptographic Details

| Component | Specification |
|-----------|---------------|
| **Encryption Algorithm** | AES-256-GCM |
| **Key Derivation** | PBKDF2-SHA256 |
| **PBKDF2 Iterations** | 100,000 |
| **Salt** | 16 bytes (random, per session) |
| **IV/Nonce** | 12 bytes (random base + counter) |
| **Auth Tag** | 16 bytes (GCM default) |
| **Chunk Size** | 64 KB |

### Why These Choices?

- **AES-256-GCM**: Industry standard authenticated encryption. Provides both confidentiality and integrity.
- **PBKDF2-SHA256**: Proven key derivation function, resistant to brute force with high iteration count.
- **100k iterations**: Balances security with performance on modern devices.
- **Counter-based IV**: Each chunk gets a unique IV derived from base IV XOR chunk index, ensuring no IV reuse.

---

## 📁 Project Structure

```
timedbox/
├── server.js          # WebSocket relay server
├── public/
│   └── index.html     # Single-page application with crypto
├── package.json       # Node.js dependencies
├── Dockerfile         # Container build
├── README.md          # This file
├── LICENSE            # AGPL-3.0
├── SECURITY.md        # Security policy
└── .github/
    └── workflows/
        └── docker.yml # CI/CD pipeline
```

---

## 🛡️ Security Considerations

### What TimedBox Protects Against

✅ **Server-side data theft** — Server only sees encrypted bytes  
✅ **Man-in-the-middle** — TLS + E2E encryption  
✅ **Data persistence** — Files never touch disk  
✅ **Metadata leakage** — Filenames encrypted, only size visible  
✅ **Session hijacking** — Random codes, short expiry  

### What TimedBox Does NOT Protect Against

⚠️ **Compromised sender device** — If malware is on the sender's computer  
⚠️ **Compromised receiver device** — If malware is on the receiver's computer  
⚠️ **Code interception** — If someone intercepts the 10-char code  
⚠️ **Traffic analysis** — File sizes are visible (but not contents)  

### Threat Model

TimedBox assumes:
- Your device is secure
- You share the code through a separate secure channel
- The receiver is who you intend

---

## 📜 License

**AGPL-3.0** — Free to use, modify, and self-host. If you run a modified version as a service, you must open-source your changes.

See [LICENSE](LICENSE) for full text.

---

## 🤝 Contributing

Contributions welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing`)
5. Open a Pull Request

---

## 💖 Support

If TimedBox is useful to you, consider:

- ⭐ **Starring** the repository
- 💳 **Donating** via [PayPal](https://www.paypal.com/paypalme/DATAROSSIT)
- 🐛 **Reporting** bugs or suggesting features
- 📢 **Sharing** with others who value privacy

---

## 📞 Contact

- **Website:** [dataross.com](https://dataross.com)
- **GitHub:** [@rosszc91](https://github.com/rosszc91)
- **Location:** Washington DC Metro

---

<p align="center">
  <strong>🔐 Your files. Encrypted before they leave.</strong><br>
  <em>© 2026 Zachary Ross / DATAROSS</em>
</p>

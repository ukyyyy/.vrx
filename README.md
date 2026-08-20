# Virex Vault (.vrx)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Security: CSP Secure](https://img.shields.io/badge/Security-CSP_connect--src_'none'-success.svg)](#security-guarantees)
[![Platform: Web / Offline](https://img.shields.io/badge/Platform-Web_%2F_Offline-blue.svg)](#deployment)

A single-file, zero-trust web tool for local file and text encryption utilizing the native **WebCrypto API**. 

Vault is designed to solve a simple problem: **How to encrypt files before sending them over insecure platforms (Slack, Discord, Email) without installing software or trusting a third-party server.**

---

## Key Features

- **100% Client-Side:** All encryption and decryption happens in your browser RAM. Your files never touch a server.
- **Zero-Network Policy:** A strict Content Security Policy (CSP) physically blocks the browser from making any outgoing network requests.
- **Two Lock Modes:** Secure your data with a cryptographically strong 256-bit random key or a user-defined password.
- **Bundling Support:** Select and compress multiple files into a single encrypted `.vrx` archive.
- **Detached Previews:** Preview images and text files directly in the browser after decryption—no disk-write required.
- **Zero Traces:** No accounts, no cookies, no tracking, and no telemetry.
- **Single-File App:** Download the `virex-vault.html` and use it completely offline (`file:///`).

---

## Security & Cryptographic Specifications

Vault uses the native **WebCrypto API** provided by modern browsers (no external CDNs, no custom JavaScript crypt-engines).

### 1. Encryption (`AES-256-GCM`)
All payloads are encrypted using **AES-GCM** (Galois/Counter Mode) with a 256-bit key length. This provides *authenticated encryption*, meaning the file's integrity is mathematically verified during decryption. If even a single bit of the `.vrx` file is altered, decryption will fail.
* **Nonce/IV:** 96-bit (12 bytes), cryptographically random (`crypto.getRandomValues`) generated uniquely for every encryption session.

### 2. Key Derivation (`PBKDF2-SHA256`)
When using Password Mode, the key is derived using PBKDF2:
* **Hash Function:** SHA-256
* **Iterations:** `600,000` (aligned with current OWASP recommendations to prevent GPU/ASIC brute-forcing)
* **Salt:** 128-bit (16 bytes) CSPRNG-generated salt per file.

### 3. Network Isolation (CSP)
To prove that Vault does not exfiltrate keys or data, the page enforces a Content Security Policy via meta tags and headers:
```http
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; connect-src 'none'; img-src 'self' data: blob:;
```
The critical directive is `connect-src 'none'`, which completely disables `fetch`, `XMLHttpRequest`, `WebSocket`, and `sendBeacon`.

---

## File Format Specification (`.vrx` v2)

The `.vrx` format is open and documented, ensuring you can write custom scripts (e.g., in Python or Go) to decrypt your files.

| Offset (Bytes) | Field Name | Description |
|---|---|---|
| `0 - 3` | **MAGIC** | `56 52 58 02` (ASCII `"VRX\x02"`) |
| `4` | **VERSION** | `02` (Format version) |
| `5` | **MODE** | `01` = Random Key Mode, `02` = Password Mode |
| `6` | **KDF** | `00` = None (Key), `01` = PBKDF2-SHA256 (Password) |
| `7` | **Reserved** | `00` (Padding/future expansion) |
| `8 - 23` | **SALT** | 16-byte random salt (used only in Password Mode) |
| `24 - 27` | **ITERS** | 4-byte Big-Endian uint32 (e.g., `600,000` iterations) |
| `28 - 39` | **IV** | 12-byte random initialization vector/nonce |
| `40 - ...` | **CIPHERTEXT** | Raw encrypted payload + 16-byte GCM authentication tag |

### Decrypted Payload Structure
Once decrypted, the payload contains:
1. `[1 Byte]` Payload Type (`0x01` = Single File, `0x02` = Multi-Bundle, `0x03` = Text)
2. `[4 Bytes]` Metadata length (Big-Endian uint32)
3. `[N Bytes]` UTF-8 JSON Metadata (filenames, sizes, MIME types)
4. `[Data Bytes]` Concatenated raw file data

---

## How to Verify & Run Locally

To ensure the integrity of the tool, you can audit the source code and run it in a sandbox:

1. Clone the repository:
   ```bash
   git clone https://github.com/ukyyyy/.vrx.git
   ```
2. Calculate the SHA-256 hash of the HTML file to match release notes:
   ```bash
   shasum -a 256 virex-vault.html
   ```
3. Open `virex-vault.html` in your browser:
   * Double-click the file to run it via `file:///` path.
   * Or run a local lightweight server:
     ```bash
     python3 -m http.server 8000
     ```
4. Verify isolation:
   * Open Developer Tools (F12) -> **Network Tab**.
   * Drag a file in, encrypt it, and check the network log. You will see **zero network traffic**.
   * Turn off your Wi-Fi/Internet completely—the application remains fully functional.

---

## Deployment & Self-Hosting

You can host this file on any static hosting provider (GitHub Pages, Vercel, Cloudflare Pages) or your own Nginx/Apache server.

For maximum security on production servers, configure the following HTTP response headers:

```nginx
# Nginx Configuration Example
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
add_header X-Frame-Options "DENY" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "no-referrer" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; connect-src 'none'; img-src 'self' data: blob:; frame-ancestors 'none'; form-action 'none';" always;
```

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

*Created by **Virex**.*

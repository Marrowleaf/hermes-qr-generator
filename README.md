# QR Generator

> Generate styled QR codes for WiFi, URLs, vCards, and more — save to Obsidian or send via Telegram with batch support.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://github.com/Marrowleaf/hermes-qr-generator)

## Features

- WiFi QR codes (WPA/WEP/open networks)
- URL, vCard, plain text, email, and SMS QR codes
- Custom styling: colors, rounded corners, logo overlays
- PNG and SVG output formats
- Batch generation from CSV files
- Send directly to Telegram or save to Obsidian vault
- High error correction level for scannable logo overlays
- Natural language command parsing

## Installation

```bash
hermes skills install productivity/qr-generator
```

Or manually clone into `~/.hermes/skills/productivity/qr-generator/`.

## Usage

```bash
# WiFi QR code
python3 generate.py --data "WIFI:T:WPA;S:MyNetwork;P:MyPassword;;" --output telegram

# URL QR code saved to Obsidian
python3 generate.py --data "https://example.com" --output obsidian --format PNG

# vCard QR code
qr-vcard name:"John Doe" phone:"+447700900123" email:"john@example.com"

# Batch from CSV
python3 batch.py --input input.csv --output-dir ~/obsidian-vault/3-Resources/QR\ Codes/

# Custom styled QR
python3 generate.py --data "https://example.com" --fill "#CC0000" --rounded
```

## Configuration

- `obsidian_vault`: Default `~/obsidian-vault`
- `qr_output_path`: Default `~/obsidian-vault/3-Resources/QR Codes`
- `default_output`: `telegram` or `obsidian` (default: telegram)
- `default_format`: PNG or SVG (default: PNG)
- Custom colors: `--fill` and `--back` flags (hex format)

## Requirements

- Python 3.8+
- `qrcode[pil]` and `Pillow` (`pip3 install 'qrcode[pil]' Pillow`)

## License

MIT
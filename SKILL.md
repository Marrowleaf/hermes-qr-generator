---
name: qr-generator
description: Generate styled QR codes for WiFi, URLs, vCards, and more — save to Obsidian or send via Telegram
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  tags:
    - qr-code
    - wifi
    - vcard
    - url
    - generator
    - obsidian
    - telegram
    - batch
  related_skills:
    - price-tracker
    - telegram-channels
---

# QR Generator

Generate QR codes for WiFi passwords, URLs, vCards, plain text, and more. Output as PNG or SVG with custom colours, logo overlays, and rounded corners. Supports batch generation from CSV data, saves to Obsidian attachments, or sends directly via Telegram.

## Overview

This skill wraps the Python `qrcode` library with `Pillow` for image manipulation. It produces styled QR codes from natural language requests and can deliver them as files in your Obsidian vault or as photos in Telegram chat.

## Commands

```yaml
commands:
  - name: qr
    description: "Generate a QR code"
    usage: "qr wifi:MyNetwork:WPA:MyPassword | qr url:https://example.com | qr text:Hello World"
    args:
      - name: type
        description: "Type: wifi, url, vcard, text, email, phone, sms"
        required: true
      - name: data
        description: "Data to encode (type-prefixed or freeform)"
        required: true
      - name: output
        description: "Output: obsidian, telegram, or file path (default: telegram)"
        required: false
      - name: format
        description: "PNG or SVG (default: PNG)"
        required: false

  - name: qr-wifi
    description: "Generate a WiFi QR code"
    usage: "qr-wifi ssid:MyNetwork password:MyPassword encryption:WPA"
    args:
      - name: ssid
        description: "WiFi network name"
        required: true
      - name: password
        description: "WiFi password"
        required: true
      - name: encryption
        description: "WPA, WEP, or nopass (default: WPA)"
        required: false
      - name: hidden
        description: "Hidden network? true/false (default: false)"
        required: false

  - name: qr-vcard
    description: "Generate a vCard QR code"
    usage: "qr-vcard name:John Doe phone:+447700900123 email:john@example.com"
    args:
      - name: name
        description: "Full name"
        required: true
      - name: phone
        description: "Phone number"
        required: false
      - name: email
        description: "Email address"
        required: false
      - name: org
        description: "Organisation"
        required: false
      - name: url
        description: "Website URL"
        required: false

  - name: qr-batch
    description: "Batch generate QR codes from a CSV file"
    usage: "qr-batch input.csv"
    args:
      - name: file
        description: "Path to CSV file"
        required: true
      - name: output_dir
        description: "Output directory (default: ~/obsidian-vault/3-Resources/QR Codes)"
        required: false

  - name: qr-style
    description: "Configure default QR styling"
    usage: "qr-style fill:#000000 back:#FFFFFF box_size:10"
    args:
      - name: fill
        description: "Fill colour hex (default: #000000)"
        required: false
      - name: back
        description: "Background colour hex (default: #FFFFFF)"
        required: false
      - name: box_size
        description: "Pixel size per QR box (default: 10)"
        required: false
      - name: border
        description: "Border boxes (default: 4)"
        required: false
```

## Configuration

```yaml
configuration:
  obsidian_vault: ~/obsidian-vault
  qr_output_path: "~/obsidian-vault/3-Resources/QR Codes"
  default_format: PNG
  default_output: telegram
  default_fill_color: "#000000"
  default_back_color: "#FFFFFF"
  default_box_size: 10
  default_border: 4
  telegram_chat_id: "6653762536"
  python_packages:
    - qrcode[pil]
    - Pillow
```

## How It Works

### 1. Quick QR Generation

Generate QR codes from natural language:

```bash
# WiFi QR code
python3 ~/.hermes/skills/qr-generator/generate.py \
  --data "WIFI:T:WPA;S:MyNetwork;P:MyPassword;;" \
  --output telegram \
  --fill "#000000" \
  --back "#FFFFFF"
```

```bash
# URL QR code
python3 ~/.hermes/skills/qr-generator/generate.py \
  --data "https://example.com" \
  --output obsidian \
  --format PNG
```

### 2. QR Generation Script

```python
#!/usr/bin/env python3
"""Generate styled QR codes."""
import argparse
import qrcode
from qrcode.image.styledpil import StyledPilImage
from qrcode.image.styles.moduledrawers import RoundedModuleDrawer
from qrcode.image.styles.colormasks import SolidFillColorMask
import os
import requests

def generate_qr(data, output="telegram", format="PNG",
                fill="#000000", back="#FFFFFF",
                box_size=10, border=4,
                logo=None, rounded=False,
                filename=None, telegram_token=None, telegram_chat_id=None):
    """Generate a QR code and output to specified destination."""

    qr = qrcode.QRCode(
        version=None,
        error_correction=qrcode.constants.ERROR_CORRECT_H,
        box_size=box_size,
        border=border,
    )
    qr.add_data(data)
    qr.make(fit=True)

    # Style options
    if rounded:
        img = qr.make_image(
            image_factory=StyledPilImage,
            module_drawer=RoundedModuleDrawer(),
            color_mask=SolidFillColorMask(
                front_color=hex_to_rgb(fill),
                back_color=hex_to_rgb(back),
            ),
        )
    else:
        img = qr.make_image(
            fill_color=fill,
            back_color=back,
        )

    # Logo overlay
    if logo and os.path.exists(logo):
        logo_img = Image.open(logo)
        logo_size = img.size[0] // 5
        logo_img = logo_img.resize((logo_size, logo_size), Image.LANCZOS)
        pos = ((img.size[0] - logo_size) // 2, (img.size[1] - logo_size) // 2)
        img.paste(logo_img, pos, logo_img if logo_img.mode == 'RGBA' else None)

    # Determine filename
    if not filename:
        safe_data = data[:30].replace('/', '_').replace(':', '_')
        filename = f"qr_{safe_data}"
    
    out_path = f"/tmp/{filename}.{format.lower()}"

    if format.upper() == "SVG":
        import qrcode.image.svg
        qr_img = qrcode.make(data, image_factory=qrcode.image.svg.SvgImage)
        with open(out_path, "w") as f:
            qr_img.save(f)
    else:
        img.save(out_path)

    # Output destination
    if output == "telegram":
        send_telegram(out_path, telegram_token, telegram_chat_id)
    elif output == "obsidian":
        obs_dir = os.path.expanduser("~/obsidian-vault/3-Resources/QR Codes")
        os.makedirs(obs_dir, exist_ok=True)
        obs_path = os.path.join(obs_dir, os.path.basename(out_path))
        os.rename(out_path, obs_path)
        print(f"Saved to: {obs_path}")
    else:
        print(f"Saved to: {out_path}")

    return out_path

def hex_to_rgb(hex_color):
    hex_color = hex_color.lstrip('#')
    return tuple(int(hex_color[i:i+2], 16) for i in (0, 2, 4))

def send_telegram(file_path, token, chat_id):
    url = f"https://api.telegram.org/bot{token}/sendPhoto"
    with open(file_path, 'rb') as f:
        requests.post(url, data={'chat_id': chat_id}, files={'photo': f})

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Generate QR codes")
    parser.add_argument("--data", required=True)
    parser.add_argument("--output", default="telegram")
    parser.add_argument("--format", default="PNG")
    parser.add_argument("--fill", default="#000000")
    parser.add_argument("--back", default="#FFFFFF")
    parser.add_argument("--box-size", type=int, default=10)
    parser.add_argument("--border", type=int, default=4)
    parser.add_argument("--logo", default=None)
    parser.add_argument("--rounded", action="store_true")
    args = parser.parse_args()
    
    generate_qr(
        data=args.data,
        output=args.output,
        format=args.format,
        fill=args.fill,
        back=args.back,
        box_size=args.box_size,
        border=args.border,
        logo=args.logo,
        rounded=args.rounded,
    )
```

### 3. QR Data Formats

**WiFi:**
```
WIFI:T:WPA;S:<SSID>;P:<PASSWORD>;;
```

**URL:**
```
https://example.com
```

**vCard:**
```
BEGIN:VCARD
VERSION:3.0
N:Doe;John
FN:John Doe
TEL:+447700900123
EMAIL:john@example.com
ORG:Acme Corp
URL:https://example.com
END:VCARD
```

**Email:**
```
mailto:hello@example.com?subject=Hello
```

**SMS:**
```
smsto:+447700900123:Hello there
```

### 4. Natural Language Parsing

Recognised patterns:
- `"qr wifi:SSID:PASSWORD"` — WiFi QR
- `"qr url:https://..."` — URL QR
- `"qr vcard:name:John phone:+44..."` — vCard QR
- `"qr text:Hello World"` — plain text QR
- `"qr batch:input.csv"` — batch from CSV
- `"generate qr code for https://..."` — auto-detect URL
- `"create wifi qr for network X with password Y"` — WiFi QR

### 5. Batch Generation from CSV

CSV format (`input.csv`):
```csv
type,data,label
wifi,MyNetwork:WPA:MyPassword,office-wifi
url,https://example.com,example-website
text,Hello World,hello-text
email,john@example.com,john-email
```

```bash
python3 ~/.hermes/skills/qr-generator/batch.py \
  --input input.csv \
  --output-dir ~/obsidian-vault/3-Resources/QR\ Codes/
```

Each row generates a separate PNG file named by label.

### 6. Custom Styling Examples

```bash
# Red QR on white background
python3 generate.py --data "https://example.com" --fill "#CC0000" --back "#FFFFFF"

# Rounded corners
python3 generate.py --data "https://example.com" --rounded

# With logo overlay
python3 generate.py --data "https://example.com" --logo ~/logo.png

# SVG output
python3 generate.py --data "https://example.com" --format svg --output obsidian
```

## Pitfalls

- **WiFi QR format**: Must follow `WIFI:T:<encryption>;S:<SSID>;P:<PASSWORD>;;` exactly. Missing semicolons will produce invalid QR codes that phones can't scan.
- **Error correction level**: Using `ERROR_CORRECT_H` (highest) is intentional — it allows logo overlays while remaining scannable. Don't lower this if using logos.
- **Logo size**: Logo overlay must not exceed ~20% of the QR code area, or it becomes unscannable. The script uses 1/5th of the image size.
- **SVG limitations**: SVG output does not support logo overlays or rounded corners (uses different image factory). Stick to PNG for styled QR codes.
- **Pillow dependency**: PNG/QR generation requires `Pillow`. SVG output requires `qrcode[pil]` as well for the base library. Install with: `pip3 install 'qrcode[pil]' Pillow`.
- **Special characters**: vCard data with commas or colons must be properly escaped. The script handles this, but manual data entry might break.
- **Telegram file size**: Telegram allows photos up to 10MB. QR codes are typically <500KB, so this is not an issue.
- **Obsidian attachment path**: Files saved to `3-Resources/QR Codes/` must be referenced with `![[qr_name.png]]` syntax in Obsidian notes, not markdown image links.
- **Colour format**: Always use hex format (`#RRGGBB`) for colours. Named colours (`red`, `blue`) are not supported by the script.

## Verification Steps

1. **Install Python dependencies:**
   ```bash
   pip3 install 'qrcode[pil]' Pillow
   python3 -c "import qrcode; print('qrcode OK')"
   python3 -c "from PIL import Image; print('Pillow OK')"
   ```

2. **Generate a test QR code (local file):**
   ```bash
   python3 ~/.hermes/skills/productivity/qr-generator/generate.py \
     --data "https://example.com" \
     --output file \
     --format PNG
   # Verify the file exists
   ls -la /tmp/qr_https___example_com.png
   ```

3. **Send a test QR via Telegram:**
   ```bash
   python3 ~/.hermes/skills/productivity/qr-generator/generate.py \
     --data "WIFI:T:WPA;S:TestNetwork;P:TestPassword;;" \
     --output telegram
   # Check Telegram chat for the QR code photo
   ```

4. **Verify Obsidian output path:**
   ```bash
   mkdir -p ~/obsidian-vault/3-Resources/QR\ Codes/
   ls ~/obsidian-vault/3-Resources/QR\ Codes/ && echo "Path OK"
   ```

5. **Test batch generation:**
   ```bash
   echo 'type,data,label
   url,https://example.com,test-url
   text,Hello,test-text' > /tmp/test-qr.csv
   python3 ~/.hermes/skills/productivity/qr-generator/batch.py \
     --input /tmp/test-qr.csv \
     --output-dir /tmp/qr-test/
   ls /tmp/qr-test/
   ```

6. **Test custom styling:**
   ```bash
   python3 ~/.hermes/skills/productivity/qr-generator/generate.py \
     --data "Styled test" \
     --fill "#0066CC" \
     --back "#F0F0F0" \
     --rounded \
     --output file
   ```
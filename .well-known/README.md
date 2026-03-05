# Hướng dẫn assetlinks.json

## File: .well-known/assetlinks.json

Domain: **link.sirobabycloud.io.vn**

### Bước 1: Lấy SHA-256 Certificate Fingerprint

**Debug build:**

```bash
keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android
```

**Release build:**

```bash
keytool -list -v -keystore /path/to/your/release.keystore -alias your-alias
```

Tìm dòng SHA256, copy chuỗi (format: `AB:CD:EF:12:34:...`)

### Bước 2: Update file assetlinks.json

Thay `XX:XX:XX:...` bằng SHA-256 fingerprint thật

### Bước 3: Validate

Test sau khi deploy:

```
https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://link.sirobabycloud.io.vn
```

### Bước 4: Test trên device

```bash
adb shell pm get-app-links vn.loyalty.vn
```

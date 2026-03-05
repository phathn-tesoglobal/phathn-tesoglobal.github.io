# 🚀 Setup Guide - Universal Links & App Links cho anhanh.io.vn

## 📋 Tổng quan

Domain: **https://anhanh.io.vn/**  
Mục tiêu: Setup Universal Links (iOS) và App Links (Android) để deeplink hoạt động hiệu quả

---

## 1️⃣ Setup Server (HTTPS bắt buộc!)

### Yêu cầu:

- ✅ Domain phải có SSL certificate (HTTPS)
- ✅ Các file cấu hình phải accessible qua HTTPS
- ✅ Content-Type phải đúng

### Cấu trúc file cần thiết:

```
demoweb/
├── index.html                                    # Web chính
├── .well-known/
│   ├── apple-app-site-association              # iOS Universal Links (NO extension)
│   └── assetlinks.json                         # Android App Links
```

---

## 2️⃣ Setup cho iOS (Universal Links)

### Bước 1: Cập nhật file `apple-app-site-association`

File tại: `.well-known/apple-app-site-association`

**Lấy Team ID và Bundle ID:**

1. Vào Apple Developer Portal
2. Team ID: Trong Account settings
3. Bundle ID: Trong app's Identifier

**Cập nhật file:**

```json
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "ABCD123456.vn.loyalty.app",
        "paths": ["*"]
      }
    ]
  }
}
```

### Bước 2: Cấu hình trong Xcode

1. Mở project trong Xcode
2. Target → Signing & Capabilities
3. Thêm "Associated Domains"
4. Thêm domain: `applinks:anhanh.io.vn`

### Bước 3: Code trong iOS App

```swift
// AppDelegate.swift hoặc SceneDelegate.swift
func application(_ application: UIApplication,
                 continue userActivity: NSUserActivity,
                 restorationHandler: @escaping ([UIUserActivityRestoring]?) -> Void) -> Bool {

    guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
          let url = userActivity.webpageURL else {
        return false
    }

    // Parse URL components
    let components = URLComponents(url: url, resolvingAgainstBaseURL: true)

    // Lấy UTM parameters
    let utmCampaign = components?.queryItems?.first(where: { $0.name == "utm_campaign" })?.value
    let utmSource = components?.queryItems?.first(where: { $0.name == "utm_source" })?.value
    let utmMedium = components?.queryItems?.first(where: { $0.name == "utm_medium" })?.value
    let utmContent = components?.queryItems?.first(where: { $0.name == "utm_content" })?.value

    print("UTM Campaign: \(utmCampaign ?? "")")
    print("UTM Source: \(utmSource ?? "")")
    print("UTM Medium: \(utmMedium ?? "")")
    print("UTM Content: \(utmContent ?? "")")

    // Xử lý logic với các tham số UTM
    handleDeeplink(campaign: utmCampaign, source: utmSource, medium: utmMedium, content: utmContent)

    return true
}
```

### Bước 4: Test trên iOS

```bash
# Test Universal Links validation
xcrun simctl openurl booted "https://anhanh.io.vn?utm_campaign=tet2026&utm_source=fb"
```

**Validate Universal Links:**
https://branch.io/resources/aasa-validator/

---

## 3️⃣ Setup cho Android (App Links)

### Bước 1: Lấy SHA-256 Certificate Fingerprint

**Debug keystore:**

```bash
keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android
```

**Release keystore:**

```bash
keytool -list -v -keystore /path/to/your/release.keystore -alias your-alias
```

Copy chuỗi SHA256 (định dạng: `AB:CD:EF:...`)

### Bước 2: Cập nhật file `assetlinks.json`

File tại: `.well-known/assetlinks.json`

```json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "vn.loyalty.vn",
      "sha256_cert_fingerprints": [
        "AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56:78:90"
      ]
    }
  }
]
```

### Bước 3: Cấu hình AndroidManifest.xml

```xml
<activity
    android:name=".MainActivity"
    android:launchMode="singleTask">

    <!-- App Links -->
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />

        <data
            android:scheme="https"
            android:host="anhanh.io.vn" />
    </intent-filter>

    <!-- Deep Links (fallback) -->
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />

        <data
            android:scheme="vn.loyalty.vn"
            android:host="" />
    </intent-filter>
</activity>
```

### Bước 4: Code trong Android App

**Kotlin:**

```kotlin
// MainActivity.kt
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    handleIntent(intent)
}

override fun onNewIntent(intent: Intent?) {
    super.onNewIntent(intent)
    intent?.let { handleIntent(it) }
}

private fun handleIntent(intent: Intent) {
    val appLinkAction = intent.action
    val appLinkData: Uri? = intent.data

    if (Intent.ACTION_VIEW == appLinkAction && appLinkData != null) {
        // Parse UTM parameters
        val utmCampaign = appLinkData.getQueryParameter("utm_campaign")
        val utmSource = appLinkData.getQueryParameter("utm_source")
        val utmMedium = appLinkData.getQueryParameter("utm_medium")
        val utmContent = appLinkData.getQueryParameter("utm_content")

        Log.d("DeepLink", "UTM Campaign: $utmCampaign")
        Log.d("DeepLink", "UTM Source: $utmSource")
        Log.d("DeepLink", "UTM Medium: $utmMedium")
        Log.d("DeepLink", "UTM Content: $utmContent")

        // Xử lý logic với các tham số UTM
        handleDeeplink(utmCampaign, utmSource, utmMedium, utmContent)
    }
}
```

### Bước 5: Test trên Android

```bash
# Test App Links
adb shell am start -W -a android.intent.action.VIEW -d "https://anhanh.io.vn?utm_campaign=tet2026&utm_source=fb" vn.loyalty.vn

# Kiểm tra verification status
adb shell pm get-app-links vn.loyalty.vn
```

**Validate App Links:**
https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://anhanh.io.vn

---

## 4️⃣ Setup Server Web

### Option 1: Python (Development)

```bash
cd d:\Phat\Job\TESO\demoweb
python -m http.server 8080
```

### Option 2: Nginx (Production)

```nginx
server {
    listen 443 ssl http2;
    server_name anhanh.io.vn;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    root /var/www/demoweb;
    index index.html;

    # Universal Links (iOS) - NO content-type header
    location = /.well-known/apple-app-site-association {
        default_type application/pkcs7-mime;
        add_header Content-Type application/json;
    }

    # App Links (Android)
    location = /.well-known/assetlinks.json {
        default_type application/json;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### Option 3: Node.js Express

```javascript
const express = require("express");
const path = require("path");
const app = express();

// Serve static files
app.use(express.static("public"));

// Apple App Site Association (iOS)
app.get("/.well-known/apple-app-site-association", (req, res) => {
  res.type("application/json");
  res.sendFile(path.join(__dirname, ".well-known/apple-app-site-association"));
});

// Asset Links (Android)
app.get("/.well-known/assetlinks.json", (req, res) => {
  res.type("application/json");
  res.sendFile(path.join(__dirname, ".well-known/assetlinks.json"));
});

app.listen(443);
```

---

## 5️⃣ Checklist Deploy

### Trước khi deploy:

- [ ] Cập nhật iOS Team ID và Bundle ID trong `apple-app-site-association`
- [ ] Cập nhật Android SHA-256 fingerprint trong `assetlinks.json`
- [ ] Cập nhật package name trong `assetlinks.json`
- [ ] Test file accessibility qua HTTPS
- [ ] Verify SSL certificate hoạt động

### Sau khi deploy:

- [ ] Test URL: `https://anhanh.io.vn`
- [ ] Test iOS Universal Links: `https://anhanh.io.vn?utm_campaign=test&utm_source=test`
- [ ] Test Android App Links: `https://anhanh.io.vn?utm_campaign=test&utm_source=test`
- [ ] Validate với Branch.io AASA validator
- [ ] Validate với Google Digital Asset Links validator

### URLs để test:

```
https://anhanh.io.vn
https://anhanh.io.vn?utm_campaign=tet2026&utm_source=fb
https://anhanh.io.vn?utm_campaign=tet2026&utm_source=fb&utm_medium=banner&utm_content=promo_image
https://anhanh.io.vn/.well-known/apple-app-site-association
https://anhanh.io.vn/.well-known/assetlinks.json
```

---

## 6️⃣ Troubleshooting

### iOS Universal Links không hoạt động:

1. Kiểm tra SSL certificate
2. Verify appID format: `TEAMID.BUNDLEID`
3. Clear Safari cache và restart device
4. Validate AASA file: https://branch.io/resources/aasa-validator/
5. Check Xcode console logs

### Android App Links không hoạt động:

1. Verify SHA-256 fingerprint chính xác
2. Check `android:autoVerify="true"` trong manifest
3. Clear app data và reinstall
4. Check logcat: `adb logcat | grep -i "AppLinks"`
5. Validate assetlinks: `adb shell pm get-app-links vn.loyalty.vn`

### Common Issues:

- **301/302 Redirects**: Không được redirect khi access AASA file
- **Wrong Content-Type**: iOS yêu cầu `application/json` hoặc no content-type
- **HTTPS required**: Cả iOS và Android đều yêu cầu HTTPS
- **CDN caching**: Clear CDN cache sau khi update config files

---

## 7️⃣ Testing Tools

### Validators:

- **iOS AASA**: https://branch.io/resources/aasa-validator/
- **Android Asset Links**: https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://anhanh.io.vn

### Debug Commands:

```bash
# iOS - Test trong simulator
xcrun simctl openurl booted "https://anhanh.io.vn?utm_campaign=tet2026&utm_source=fb"

# Android - Test trên device
adb shell am start -W -a android.intent.action.VIEW -d "https://anhanh.io.vn?utm_campaign=tet2026&utm_source=fb"

# Android - Check verification
adb shell pm get-app-links vn.loyalty.vn
```

---

## 📞 Support

Nếu gặp vấn đề, check:

1. Server logs
2. Mobile app logs (Xcode Console / Logcat)
3. Network requests (Charles Proxy / Proxyman)
4. SSL certificate validity

Good luck! 🚀

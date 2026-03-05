# 🚀 Setup Subdomain cho Android App Links

## 📋 Tổng quan

- **Domain chính**: sirobabycloud.io.vn (đang dùng cho mục đích khác)
- **Subdomain cho app**: link.sirobabycloud.io.vn
- **Alternative subdomain**: app.sirobabycloud.io.vn
- **Mục tiêu**: Setup Android App Links để deeplink hoạt động

---

## 1️⃣ Tạo Subdomain

### Option 1: link.sirobabycloud.io.vn (Recommended)

```
Subdomain: link.sirobabycloud.io.vn
Mục đích: Dành cho deeplink, tracking, và redirect
```

### Option 2: app.sirobabycloud.io.vn

```
Subdomain: app.sirobabycloud.io.vn
Mục đích: Dành cho app-related features
```

### Lưu ý quan trọng:

- ✅ **Chỉ cần 1 subdomain** để deeplink hoạt động
- ✅ Subdomain phải có **SSL certificate** (HTTPS bắt buộc!)
- ✅ Subdomain phải trỏ về server hosting file `assetlinks.json`

---

## 2️⃣ Cấu hình DNS

### Tại nhà cung cấp domain (VD: CloudFlare, GoDaddy, etc.)

#### Thêm DNS Record:

```
Type: A hoặc CNAME
Name: link
Host/Target: IP server của bạn (hoặc CNAME đến server)
TTL: Auto hoặc 3600
```

#### Ví dụ cụ thể:

**Option A - Record type A (IP trực tiếp):**

```
Type: A
Name: link
Value: 103.15.51.97  (IP server của bạn)
TTL: 3600
```

**Option B - Record type CNAME (alias):**

```
Type: CNAME
Name: link
Value: your-server.example.com
TTL: 3600
```

#### Kết quả:

```
link.sirobabycloud.io.vn → Server hosting files
```

---

## 3️⃣ Cấu trúc Server

### Server phải host các file sau:

```
/var/www/link.sirobabycloud.io.vn/     (hoặc thư mục khác)
├── index.html                          # Web test deeplink
├── .well-known/
│   └── assetlinks.json                 # Android App Links config
```

### File paths phải accessible:

- ✅ `https://link.sirobabycloud.io.vn/`
- ✅ `https://link.sirobabycloud.io.vn/.well-known/assetlinks.json`

---

## 4️⃣ Cấu hình Server Web

### Option 1: Nginx (Production - Recommended)

#### File: `/etc/nginx/sites-available/link.sirobabycloud.io.vn`

```nginx
server {
    listen 80;
    server_name link.sirobabycloud.io.vn;

    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name link.sirobabycloud.io.vn;

    # SSL Certificate (QUAN TRỌNG!)
    ssl_certificate /etc/letsencrypt/live/link.sirobabycloud.io.vn/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/link.sirobabycloud.io.vn/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Document root
    root /var/www/link.sirobabycloud.io.vn;
    index index.html;

    # CORS headers (nếu cần)
    add_header 'Access-Control-Allow-Origin' '*' always;

    # Android App Links
    location = /.well-known/assetlinks.json {
        default_type application/json;
        add_header 'Access-Control-Allow-Origin' '*' always;
        try_files $uri =404;
    }

    # Main site
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Logging
    access_log /var/log/nginx/link.sirobabycloud.io.vn_access.log;
    error_log /var/log/nginx/link.sirobabycloud.io.vn_error.log;
}
```

#### Enable site:

```bash
sudo ln -s /etc/nginx/sites-available/link.sirobabycloud.io.vn /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### Option 2: Apache

#### File: `/etc/apache2/sites-available/link.sirobabycloud.io.vn.conf`

```apache
<VirtualHost *:80>
    ServerName link.sirobabycloud.io.vn
    Redirect permanent / https://link.sirobabycloud.io.vn/
</VirtualHost>

<VirtualHost *:443>
    ServerName link.sirobabycloud.io.vn

    # SSL Configuration
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/link.sirobabycloud.io.vn/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/link.sirobabycloud.io.vn/privkey.pem

    DocumentRoot /var/www/link.sirobabycloud.io.vn

    <Directory /var/www/link.sirobabycloud.io.vn>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    # Android App Links
    <Location /.well-known/assetlinks.json>
        Header set Content-Type "application/json"
        Header set Access-Control-Allow-Origin "*"
    </Location>

    ErrorLog ${APACHE_LOG_DIR}/link.sirobabycloud.io.vn_error.log
    CustomLog ${APACHE_LOG_DIR}/link.sirobabycloud.io.vn_access.log combined
</VirtualHost>
```

#### Enable site:

```bash
sudo a2ensite link.sirobabycloud.io.vn.conf
sudo a2enmod ssl headers
sudo systemctl reload apache2
```

### Option 3: Node.js/Express

```javascript
const express = require("express");
const https = require("https");
const fs = require("fs");
const path = require("path");

const app = express();

// Serve static files
app.use(express.static(path.join(__dirname, "public")));

// Android App Links
app.get("/.well-known/assetlinks.json", (req, res) => {
  res.type("application/json");
  res.header("Access-Control-Allow-Origin", "*");
  res.sendFile(path.join(__dirname, ".well-known/assetlinks.json"));
});

// Main page
app.get("*", (req, res) => {
  res.sendFile(path.join(__dirname, "public/index.html"));
});

// HTTPS server
const options = {
  key: fs.readFileSync(
    "/etc/letsencrypt/live/link.sirobabycloud.io.vn/privkey.pem",
  ),
  cert: fs.readFileSync(
    "/etc/letsencrypt/live/link.sirobabycloud.io.vn/fullchain.pem",
  ),
};

https.createServer(options, app).listen(443, () => {
  console.log("Server running on https://link.sirobabycloud.io.vn");
});
```

---

## 5️⃣ SSL Certificate (HTTPS) - BẮT BUỘC!

### Sử dụng Let's Encrypt (Free):

```bash
# Install certbot
sudo apt update
sudo apt install certbot python3-certbot-nginx  # cho Nginx
# hoặc
sudo apt install certbot python3-certbot-apache  # cho Apache

# Tạo certificate cho subdomain
sudo certbot --nginx -d link.sirobabycloud.io.vn
# hoặc
sudo certbot --apache -d link.sirobabycloud.io.vn

# Auto renew
sudo certbot renew --dry-run
```

### Kiểm tra SSL:

```bash
curl -I https://link.sirobabycloud.io.vn
```

---

## 6️⃣ Setup Android App

### Bước 1: Lấy SHA-256 Certificate Fingerprint

```bash
# Debug keystore
keytool -list -v -keystore ~/.android/debug.keystore -alias androiddebugkey -storepass android -keypass android | grep SHA256

# Release keystore
keytool -list -v -keystore /path/to/release.keystore -alias your-alias | grep SHA256
```

Copy chuỗi SHA256 (format: `AB:CD:EF:12:34:...`)

### Bước 2: Update assetlinks.json

File: `.well-known/assetlinks.json`

```json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "vn.loyalty.vn",
      "sha256_cert_fingerprints": [
        "AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56:78:90:AB:CD:EF:12:34:56"
      ]
    }
  }
]
```

### Bước 3: AndroidManifest.xml

```xml
<activity
    android:name=".MainActivity"
    android:launchMode="singleTask">

    <!-- App Links cho subdomain -->
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />

        <data
            android:scheme="https"
            android:host="link.sirobabycloud.io.vn" />
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

### Bước 4: Code xử lý Intent

**Kotlin:**

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    handleIntent(intent)
}

override fun onNewIntent(intent: Intent?) {
    super.onNewIntent(intent)
    intent?.let { handleIntent(it) }
}

private fun handleIntent(intent: Intent) {
    val appLinkData: Uri? = intent.data

    if (Intent.ACTION_VIEW == intent.action && appLinkData != null) {
        // Parse UTM parameters
        val utmCampaign = appLinkData.getQueryParameter("utm_campaign")
        val utmSource = appLinkData.getQueryParameter("utm_source")
        val utmMedium = appLinkData.getQueryParameter("utm_medium")
        val utmContent = appLinkData.getQueryParameter("utm_content")

        Log.d("DeepLink", "URL: ${appLinkData.toString()}")
        Log.d("DeepLink", "UTM Campaign: $utmCampaign")
        Log.d("DeepLink", "UTM Source: $utmSource")
        Log.d("DeepLink", "UTM Medium: $utmMedium")
        Log.d("DeepLink", "UTM Content: $utmContent")

        // Xử lý logic business
        when {
            utmCampaign == "tet2026" -> {
                // Điều hướng đến màn hình TET campaign
                navigateToTetCampaign(utmSource, utmMedium, utmContent)
            }
            utmSource == "fb" -> {
                // Track Facebook traffic
                trackFacebookTraffic(utmCampaign, utmMedium)
            }
            else -> {
                // Default handling
                handleGenericDeeplink(appLinkData)
            }
        }
    }
}

private fun navigateToTetCampaign(source: String?, medium: String?, content: String?) {
    // Analytics tracking
    FirebaseAnalytics.getInstance(this).logEvent("tet_campaign_opened") {
        param("utm_source", source ?: "unknown")
        param("utm_medium", medium ?: "unknown")
        param("utm_content", content ?: "unknown")
    }

    // Navigate to campaign screen
    val intent = Intent(this, TetCampaignActivity::class.java).apply {
        putExtra("source", source)
        putExtra("medium", medium)
        putExtra("content", content)
    }
    startActivity(intent)
}
```

---

## 7️⃣ Deploy Files

### Upload files lên server:

```bash
# Via SCP
scp -r demoweb/* user@your-server:/var/www/link.sirobabycloud.io.vn/

# Via FTP/SFTP
# Sử dụng FileZilla hoặc WinSCP
# Upload tất cả files vào thư mục web root
```

### Set permissions:

```bash
sudo chown -R www-data:www-data /var/www/link.sirobabycloud.io.vn
sudo chmod -R 755 /var/www/link.sirobabycloud.io.vn
```

---

## 8️⃣ Testing

### Test 1: Check DNS Resolution

```bash
nslookup link.sirobabycloud.io.vn
ping link.sirobabycloud.io.vn
```

### Test 2: Check HTTPS và SSL

```bash
curl -I https://link.sirobabycloud.io.vn
curl https://link.sirobabycloud.io.vn/.well-known/assetlinks.json
```

### Test 3: Validate App Links

```
https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://link.sirobabycloud.io.vn
```

### Test 4: Test trên Android Device

```bash
# Install app với debug/release build
adb install app-debug.apk

# Test deeplink
adb shell am start -W -a android.intent.action.VIEW \
  -d "https://link.sirobabycloud.io.vn?utm_campaign=tet2026&utm_source=fb" \
  vn.loyalty.vn

# Check verification status
adb shell pm get-app-links vn.loyalty.vn

# Expected output:
# link.sirobabycloud.io.vn: verified
```

### Test 5: Test trong Browser

```
https://link.sirobabycloud.io.vn?utm_campaign=tet2026&utm_source=fb&utm_medium=banner
```

---

## 9️⃣ Troubleshooting

### Issue: DNS không resolve

```bash
# Clear DNS cache
sudo systemd-resolve --flush-caches  # Linux
ipconfig /flushdns  # Windows

# Wait for DNS propagation (có thể mất 1-24h)
# Check: https://dnschecker.org/#A/link.sirobabycloud.io.vn
```

### Issue: SSL certificate error

```bash
# Verify certificate
openssl s_client -connect link.sirobabycloud.io.vn:443 -servername link.sirobabycloud.io.vn

# Renew certificate
sudo certbot renew
```

### Issue: assetlinks.json not found

```bash
# Check file exists
ls -la /var/www/link.sirobabycloud.io.vn/.well-known/assetlinks.json

# Check nginx/apache config
sudo nginx -t
sudo apache2ctl -t

# Check permissions
sudo chmod 644 /var/www/link.sirobabycloud.io.vn/.well-known/assetlinks.json
```

### Issue: Android App Links verification failed

```bash
# Check logcat during app install
adb logcat | grep -i "IntentFilter"

# Manually verify
adb shell pm verify-app-links --re-verify vn.loyalty.vn

# Check status
adb shell pm get-app-links vn.loyalty.vn
```

### Issue: Deeplink không mở app

1. Reinstall app (để trigger verification lại)
2. Clear app data: Settings → Apps → Your App → Clear Data
3. Check manifest có `android:autoVerify="true"`
4. Verify SHA-256 fingerprint đúng
5. Check logcat: `adb logcat | grep -i "deeplink"`

---

## 🎯 Summary - Tóm tắt

### Subdomain setup:

```
Domain chính: sirobabycloud.io.vn (dùng cho mục đích khác)
Subdomain:    link.sirobabycloud.io.vn (dùng cho deeplink)
```

### Subdomain trỏ về đâu?

```
link.sirobabycloud.io.vn → IP Server hosting web + assetlinks.json
```

### Files cần thiết trên server:

```
/var/www/link.sirobabycloud.io.vn/
├── index.html                          # Web test
├── .well-known/
│   └── assetlinks.json                 # Android App Links config
```

### URLs cần hoạt động:

```
✅ https://link.sirobabycloud.io.vn/
✅ https://link.sirobabycloud.io.vn/.well-known/assetlinks.json
✅ https://link.sirobabycloud.io.vn?utm_campaign=tet2026&utm_source=fb
```

### Checklist:

- [ ] Tạo subdomain DNS record (A hoặc CNAME)
- [ ] Setup SSL certificate (Let's Encrypt)
- [ ] Deploy files lên server
- [ ] Config Nginx/Apache
- [ ] Update assetlinks.json với SHA-256 fingerprint
- [ ] Update AndroidManifest.xml với subdomain
- [ ] Test với adb command
- [ ] Validate với Google API

---

## 📞 Next Steps

1. **Tạo subdomain** tại DNS provider (CloudFlare/GoDaddy/etc.)
2. **Point subdomain** đến IP server của bạn
3. **Setup SSL** bằng Let's Encrypt
4. **Deploy files** từ folder `demoweb`
5. **Get SHA-256** từ keystore
6. **Update assetlinks.json**
7. **Test!**

Good luck! 🚀

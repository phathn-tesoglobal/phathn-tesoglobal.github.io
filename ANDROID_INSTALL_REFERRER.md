# 📱 Hướng dẫn nhận Install Referrer trong Android App

## Khi user install app từ Play Store với referrer, app cần code để nhận UTM data

---

## 1️⃣ Thêm dependency vào build.gradle

```gradle
dependencies {
    // Play Install Referrer Library
    implementation 'com.android.installreferrer:installreferrer:2.2'
}
```

---

## 2️⃣ Code nhận Install Referrer

### Kotlin:

```kotlin
import com.android.installreferrer.api.InstallReferrerClient
import com.android.installreferrer.api.InstallReferrerStateListener
import android.net.Uri
import android.util.Log

class MainActivity : AppCompatActivity() {

    private lateinit var referrerClient: InstallReferrerClient

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        // Lấy install referrer
        getInstallReferrer()
    }

    private fun getInstallReferrer() {
        referrerClient = InstallReferrerClient.newBuilder(this).build()
        referrerClient.startConnection(object : InstallReferrerStateListener {

            override fun onInstallReferrerSetupFinished(responseCode: Int) {
                when (responseCode) {
                    InstallReferrerClient.InstallReferrerResponse.OK -> {
                        try {
                            val response = referrerClient.installReferrer
                            val referrerUrl = response.installReferrer

                            Log.d("InstallReferrer", "Raw referrer: $referrerUrl")

                            // Parse UTM parameters
                            val utmData = parseReferrer(referrerUrl)

                            Log.d("InstallReferrer", "UTM Campaign: ${utmData["utm_campaign"]}")
                            Log.d("InstallReferrer", "UTM Source: ${utmData["utm_source"]}")
                            Log.d("InstallReferrer", "UTM Medium: ${utmData["utm_medium"]}")
                            Log.d("InstallReferrer", "UTM Content: ${utmData["utm_content"]}")

                            // Lưu vào SharedPreferences hoặc gửi lên server
                            saveUtmData(utmData)

                            // Gửi lên analytics
                            trackInstallSource(utmData)

                        } catch (e: Exception) {
                            Log.e("InstallReferrer", "Error: ${e.message}")
                        } finally {
                            referrerClient.endConnection()
                        }
                    }

                    InstallReferrerClient.InstallReferrerResponse.FEATURE_NOT_SUPPORTED -> {
                        Log.w("InstallReferrer", "Feature not supported")
                    }

                    InstallReferrerClient.InstallReferrerResponse.SERVICE_UNAVAILABLE -> {
                        Log.w("InstallReferrer", "Service unavailable")
                    }
                }
            }

            override fun onInstallReferrerServiceDisconnected() {
                Log.w("InstallReferrer", "Service disconnected")
                // Có thể retry kết nối
            }
        })
    }

    private fun parseReferrer(referrer: String): Map<String, String> {
        val utmData = mutableMapOf<String, String>()

        try {
            // Parse query string
            val params = referrer.split("&")
            for (param in params) {
                val keyValue = param.split("=")
                if (keyValue.size == 2) {
                    val key = Uri.decode(keyValue[0])
                    val value = Uri.decode(keyValue[1])
                    utmData[key] = value
                }
            }
        } catch (e: Exception) {
            Log.e("InstallReferrer", "Parse error: ${e.message}")
        }

        return utmData
    }

    private fun saveUtmData(utmData: Map<String, String>) {
        val prefs = getSharedPreferences("app_prefs", Context.MODE_PRIVATE)
        prefs.edit().apply {
            utmData.forEach { (key, value) ->
                putString(key, value)
            }
            apply()
        }
    }

    private fun trackInstallSource(utmData: Map<String, String>) {
        // Firebase Analytics
        FirebaseAnalytics.getInstance(this).logEvent("app_install") {
            utmData.forEach { (key, value) ->
                param(key, value)
            }
        }

        // Hoặc gửi lên server của bạn
        // apiService.trackInstall(utmData)
    }
}
```

---

## 3️⃣ Luồng hoạt động hoàn chỉnh

### Case 1: User chưa cài app

```
1. User click: https://lc2jf9bn-8080.asse.devtunnels.ms?utm_campaign=tet2026&utm_source=fb
2. Web hiển thị → User click "Mở App"
3. Try deeplink: vn.loyalty.vn://?utm_campaign=tet2026&utm_source=fb (FAIL - chưa cài)
4. Redirect Play Store:
   https://play.google.com/store/apps/details?id=vn.loyalty.vn&referrer=utm_source%3Dfb%26utm_campaign%3Dtet2026
5. User install app
6. App mở lần đầu → getInstallReferrer() nhận được:
   utm_source=fb
   utm_campaign=tet2026
7. App lưu UTM data → analytics → server
```

### Case 2: User đã cài app

```
1. User click: https://lc2jf9bn-8080.asse.devtunnels.ms?utm_campaign=tet2026&utm_source=fb
2. Web hiển thị → User click "Mở App"
3. Open deeplink: vn.loyalty.vn://?utm_campaign=tet2026&utm_source=fb (SUCCESS)
4. App nhận Intent → handleIntent() parse UTM params:
   utm_campaign=tet2026
   utm_source=fb
5. App xử lý logic business với UTM data
```

---

## 4️⃣ Test Install Referrer

### Test trên device:

```bash
# Simulate install referrer (trước khi install app)
adb shell am broadcast \
  -a com.android.vending.INSTALL_REFERRER \
  -n vn.loyalty.vn/com.android.installreferrer.InstallReferrerReceiver \
  --es "referrer" "utm_source=fb&utm_campaign=tet2026&utm_medium=banner"

# Sau đó install app và check logs
adb logcat | grep InstallReferrer
```

### Debug output:

```
InstallReferrer: Raw referrer: utm_source=fb&utm_campaign=tet2026&utm_medium=banner
InstallReferrer: UTM Campaign: tet2026
InstallReferrer: UTM Source: fb
InstallReferrer: UTM Medium: banner
```

---

## 5️⃣ Lưu ý quan trọng

### ⏰ Timing:

- Install referrer chỉ available **trong 90 ngày** sau khi install
- Nên lấy ngay trong **lần mở app đầu tiên**
- Lưu vào storage để dùng sau

### 🔒 Security:

- Referrer có thể bị fake → validate trước khi dùng
- Không lưu sensitive data trong referrer

### 📊 Analytics:

- Gửi install attribution lên Firebase/Adjust/AppsFlyer
- Track conversion: Install → First Open → Action

---

## 6️⃣ URL Examples

### Play Store URL với referrer:

```
https://play.google.com/store/apps/details?id=vn.loyalty.vn&referrer=utm_source%3Dfb%26utm_campaign%3Dtet2026%26utm_medium%3Dbanner%26utm_content%3Dpromo_image
```

### Referrer sẽ được decode thành:

```
utm_source=fb
utm_campaign=tet2026
utm_medium=banner
utm_content=promo_image
```

---

## 7️⃣ Best Practices

1. **Check null/empty**: Referrer có thể null nếu user install từ APK file
2. **Retry logic**: Service có thể unavailable, implement retry
3. **Cache results**: Lưu referrer ngay sau khi get được
4. **Error handling**: Handle tất cả error cases
5. **Privacy**: Tuân thủ GDPR/privacy laws khi track user

---

## 📞 Summary

✅ Web đã setup xong: Tự động thêm referrer vào Play Store URL  
✅ App cần code: Install Referrer Library để nhận UTM data  
✅ Analytics: Track install attribution để measure campaign effectiveness

Good luck! 🚀

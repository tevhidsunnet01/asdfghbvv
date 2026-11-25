```bash
#!/usr/bin/env bash
set -e

mkdir -p ".github/workflows"
cat << 'EOF_FILE' > ".github/workflows/build-apk.yml"
name: Build Android APK

on:
  push:
    branches: [ main, master ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.10.2'

      - name: Build Debug APK
        run: gradle assembleDebug

      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: NamazVaktiWidget-APK
          path: app/build/outputs/apk/debug/app-debug.apk
EOF_FILE

cat << 'EOF_FILE' > "KURULUM.md"
# Namaz Vakitleri Android Widget — Kurulum ve APK Alma

Bu proje, Diyanet İşleri Başkanlığı verilerini [Ezan Vakti API](https://ezanvakti.emushaf.net/) üzerinden çeken, sade koyu temalı bir Android ana ekran aracı (widget) uygulamasıdır.

## Özellikler
- Gönderdiğiniz ekran görüntüsündeki koyu kart tasarımına uygun görünüm (`Namaz vakitleri`, `İl / İlçe · Tarih`, sıradaki vakit, büyük saat ve saniyelik canlı geri sayım)
- Widget'a dokunulduğunda açılan sade **Şehir ve İlçe Seçimi** ekranı
- Seçilen konumu `İl / İlçe` biçiminde widget üstünde gösterme
- 30 günlük Diyanet vakitlerini önbellekte saklama ve vakit geçince otomatik yenileme
- Ek kütüphane bağımlılığı olmadan küçük boyutlu, pil dostu çalışma (Android 7.0+)

## APK Nasıl Üretilir?

### Yöntem 1: Android Studio Kurmadan (GitHub Actions — Önerilen)
1. [GitHub](https://github.com) üzerinde yeni bir depo (repository) oluşturun.
2. Bu ZIP dosyasının içindeki tüm dosya ve klasörleri (`.github` klasörü dahil) depoya yükleyin.
3. Depoda **Actions** sekmesine girin; **Build Android APK** iş akışı otomatik çalışacaktır (yaklaşık 2 dakika sürer).
4. İşlem bitince **Artifacts** bölümünden `NamazVaktiWidget-APK` dosyasını indirin ve içindeki `app-debug.apk` dosyasını telefonunuza kurun.

### Yöntem 2: Android Studio ile
1. ZIP dosyasını bir klasöre çıkarın ve Android Studio'da **Open** ile açın.
2. Gradle senkronizasyonu bittikten sonra üst menüden **Build → Build Bundle(s) / APK(s) → Build APK(s)** seçin.
3. Oluşan APK dosyasını (`app/build/outputs/apk/debug/app-debug.apk`) telefonunuza aktarıp kurun.

## Ana Ekrana Araç (Widget) Olarak Ekleme
1. Telefonunuzun ana ekranında boş bir alana uzun basın ve **Araçlar (Widgets)** menüsünü açın.
2. Listeden **Namaz Vakitleri** aracını ana ekranınıza sürükleyip bırakın.
3. Açılan ekranda **İl** ve **İlçe** seçip **Kaydet**'e dokunun. Daha sonra konumu değiştirmek istediğinizde widget'ın üzerine dokunmanız yeterlidir.
EOF_FILE

mkdir -p "app"
cat << 'EOF_FILE' > "app/build.gradle.kts"
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.namazvakti.widget"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.namazvakti.widget"
        minSdk = 24
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }
}

dependencies {
    // Saf Android API kullanıldığı için ek kütüphane bağımlılığı yoktur;
    // APK boyutu çok küçük kalır ve derleme hızlı tamamlanır.
}
EOF_FILE

mkdir -p "app"
cat << 'EOF_FILE' > "app/proguard-rules.pro"
# NamazVaktiWidget ProGuard rules
EOF_FILE

mkdir -p "app/src/main"
cat << 'EOF_FILE' > "app/src/main/AndroidManifest.xml"
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher"
        android:supportsRtl="true"
        android:theme="@style/Theme.NamazVaktiWidget">

        <activity
            android:name=".CitySelectActivity"
            android:exported="true"
            android:launchMode="singleTask">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
            <intent-filter>
                <action android:name="android.appwidget.action.APPWIDGET_CONFIGURE" />
            </intent-filter>
        </activity>

        <receiver
            android:name=".NamazWidgetProvider"
            android:exported="true"
            android:label="@string/app_name">
            <intent-filter>
                <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
                <action android:name="android.intent.action.BOOT_COMPLETED" />
                <action android:name="android.intent.action.TIME_SET" />
                <action android:name="android.intent.action.TIMEZONE_CHANGED" />
                <action android:name="com.namazvakti.widget.ACTION_TICK_UPDATE" />
            </intent-filter>
            <meta-data
                android:name="android.appwidget.provider"
                android:resource="@xml/namaz_widget_info" />
        </receiver>

    </application>
</manifest>
EOF_FILE

mkdir -p "app/src/main/java/com/namazvakti/widget"
cat << 'EOF_FILE' > "app/src/main/java/com/namazvakti/widget/CitySelectActivity.kt"
package com.namazvakti.widget

import android.app.Activity
import android.appwidget.AppWidgetManager
import android.content.Intent
import android.os.Bundle
import android.view.View
import android.widget.AdapterView
import android.widget.ArrayAdapter
import android.widget.Button
import android.widget.ProgressBar
import android.widget.Spinner
import android.widget.TextView
import android.widget.Toast
import java.util.concurrent.Executors

class CitySelectActivity : Activity() {

    private lateinit var spinnerCity: Spinner
    private lateinit var spinnerDistrict: Spinner
    private lateinit var progressBar: ProgressBar
    private lateinit var tvStatus: TextView
    private lateinit var btnSave: Button

    private val executor = Executors.newSingleThreadExecutor()
    private var cities: List<CityItem> = emptyList()
    private var districts: List<DistrictItem> = emptyList()
    private var appWidgetId: Int = AppWidgetManager.INVALID_APPWIDGET_ID
    private var savedLocation: SavedLocation? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_city_select)

        appWidgetId = intent?.extras?.getInt(
            AppWidgetManager.EXTRA_APPWIDGET_ID,
            AppWidgetManager.INVALID_APPWIDGET_ID
        ) ?: AppWidgetManager.INVALID_APPWIDGET_ID

        if (appWidgetId != AppWidgetManager.INVALID_APPWIDGET_ID) {
            setResult(RESULT_CANCELED, Intent().putExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, appWidgetId))
        }

        spinnerCity = findViewById(R.id.spinner_city)
        spinnerDistrict = findViewById(R.id.spinner_district)
        progressBar = findViewById(R.id.progress_bar)
        tvStatus = findViewById(R.id.tv_status)
        btnSave = findViewById(R.id.btn_save)

        savedLocation = PrayerPrefs.getLocation(this)

        spinnerCity.onItemSelectedListener = object : AdapterView.OnItemSelectedListener {
            override fun onItemSelected(parent: AdapterView<*>?, view: View?, position: Int, id: Long) {
                val selectedCity = cities.getOrNull(position) ?: return
                loadDistricts(selectedCity.id)
            }

            override fun onNothingSelected(parent: AdapterView<*>?) = Unit
        }

        btnSave.setOnClickListener { saveSelection() }
        loadCities()
    }

    private fun loadCities() {
        setLoading(true, "Şehirler yükleniyor...")
        executor.execute {
            try {
                val loaded = DiyanetApi.fetchCities()
                runOnUiThread {
                    cities = loaded
                    val adapter = ArrayAdapter(
                        this,
                        R.layout.item_spinner,
                        cities
                    ).apply {
                        setDropDownViewResource(R.layout.item_spinner_dropdown)
                    }
                    spinnerCity.adapter = adapter

                    val preselectIndex = savedLocation?.let { saved ->
                        cities.indexOfFirst { it.id == saved.cityId }
                    }?.takeIf { it >= 0 } ?: cities.indexOfFirst { it.name.equals("İSTANBUL", ignoreCase = true) }
                        .takeIf { it >= 0 } ?: 0

                    spinnerCity.setSelection(preselectIndex)
                    setLoading(false, null)
                }
            } catch (e: Exception) {
                runOnUiThread {
                    setLoading(false, "Şehir listesi alınamadı. İnternet bağlantınızı kontrol edin.")
                }
            }
        }
    }

    private fun loadDistricts(cityId: String) {
        setLoading(true, "İlçeler yükleniyor...")
        executor.execute {
            try {
                val loaded = DiyanetApi.fetchDistricts(cityId)
                runOnUiThread {
                    districts = loaded
                    val adapter = ArrayAdapter(
                        this,
                        R.layout.item_spinner,
                        districts
                    ).apply {
                        setDropDownViewResource(R.layout.item_spinner_dropdown)
                    }
                    spinnerDistrict.adapter = adapter

                    val preselectIndex = savedLocation?.takeIf { it.cityId == cityId }?.let { saved ->
                        districts.indexOfFirst { it.id == saved.districtId }
                    }?.takeIf { it >= 0 } ?: 0

                    spinnerDistrict.setSelection(preselectIndex)
                    setLoading(false, null)
                }
            } catch (e: Exception) {
                runOnUiThread {
                    setLoading(false, "İlçe listesi alınamadı.")
                }
            }
        }
    }

    private fun saveSelection() {
        val selectedCity = cities.getOrNull(spinnerCity.selectedItemPosition)
        val selectedDistrict = districts.getOrNull(spinnerDistrict.selectedItemPosition)
        if (selectedCity == null || selectedDistrict == null) {
            Toast.makeText(this, "Lütfen önce il ve ilçe seçin.", Toast.LENGTH_SHORT).show()
            return
        }

        setLoading(true, "Diyanet vakitleri alınıyor...")
        executor.execute {
            try {
                val rawJson = DiyanetApi.fetchPrayerTimesRawJson(selectedDistrict.id)
                val location = SavedLocation(
                    cityId = selectedCity.id,
                    cityName = selectedCity.name,
                    districtId = selectedDistrict.id,
                    districtName = selectedDistrict.name
                )
                PrayerPrefs.saveLocation(this, location)
                PrayerPrefs.saveCachedSchedule(this, rawJson)
                NamazWidgetProvider.updateAllWidgets(this, forceFetch = false)

                runOnUiThread {
                    setLoading(false, null)
                    Toast.makeText(this, "${location.displayLabel()} kaydedildi", Toast.LENGTH_SHORT).show()
                    if (appWidgetId != AppWidgetManager.INVALID_APPWIDGET_ID) {
                        val result = Intent().putExtra(AppWidgetManager.EXTRA_APPWIDGET_ID, appWidgetId)
                        setResult(RESULT_OK, result)
                    }
                    finish()
                }
            } catch (e: Exception) {
                runOnUiThread {
                    setLoading(false, "Vakitler alınamadı. Tekrar deneyin.")
                }
            }
        }
    }

    private fun setLoading(loading: Boolean, message: String?) {
        progressBar.visibility = if (loading) View.VISIBLE else View.GONE
        btnSave.isEnabled = !loading
        if (message.isNullOrBlank()) {
            tvStatus.visibility = View.GONE
        } else {
            tvStatus.text = message
            tvStatus.visibility = View.VISIBLE
        }
    }
}
EOF_FILE

mkdir -p "app/src/main/java/com/namazvakti/widget"
cat << 'EOF_FILE' > "app/src/main/java/com/namazvakti/widget/DiyanetApi.kt"
package com.namazvakti.widget

import org.json.JSONArray
import java.io.BufferedReader
import java.io.InputStreamReader
import java.net.HttpURLConnection
import java.net.URL

data class CityItem(
    val id: String,
    val name: String
) {
    override fun toString(): String = PrayerPrefs.toTurkishTitleCase(name)
}

data class DistrictItem(
    val id: String,
    val name: String
) {
    override fun toString(): String = PrayerPrefs.toTurkishTitleCase(name)
}

object DiyanetApi {
    private const val BASE_URL = "https://ezanvakti.emushaf.net"
    private const val TURKEY_COUNTRY_ID = "2"

    fun fetchCities(): List<CityItem> {
        val body = httpGet("$BASE_URL/sehirler/$TURKEY_COUNTRY_ID")
        val arr = JSONArray(body)
        val result = ArrayList<CityItem>(arr.length())
        for (i in 0 until arr.length()) {
            val obj = arr.getJSONObject(i)
            result.add(
                CityItem(
                    id = obj.getString("SehirID"),
                    name = obj.getString("SehirAdi")
                )
            )
        }
        return result
    }

    fun fetchDistricts(cityId: String): List<DistrictItem> {
        val body = httpGet("$BASE_URL/ilceler/$cityId")
        val arr = JSONArray(body)
        val result = ArrayList<DistrictItem>(arr.length())
        for (i in 0 until arr.length()) {
            val obj = arr.getJSONObject(i)
            result.add(
                DistrictItem(
                    id = obj.getString("IlceID"),
                    name = obj.getString("IlceAdi")
                )
            )
        }
        return result
    }

    fun fetchPrayerTimesRawJson(districtId: String): String {
        return httpGet("$BASE_URL/vakitler/$districtId")
    }

    private fun httpGet(urlString: String): String {
        val connection = (URL(urlString).openConnection() as HttpURLConnection).apply {
            requestMethod = "GET"
            connectTimeout = 10_000
            readTimeout = 10_000
            setRequestProperty("Accept", "application/json")
            setRequestProperty("User-Agent", "NamazVaktiWidget/1.0")
        }
        return try {
            val code = connection.responseCode
            if (code !in 200..299) {
                throw IllegalStateException("Sunucu hatası: HTTP $code")
            }
            BufferedReader(InputStreamReader(connection.inputStream, Charsets.UTF_8)).use { reader ->
                reader.readText()
            }
        } finally {
            connection.disconnect()
        }
    }
}
EOF_FILE

mkdir -p "app/src/main/java/com/namazvakti/widget"
cat << 'EOF_FILE' > "app/src/main/java/com/namazvakti/widget/NamazWidgetProvider.kt"
package com.namazvakti.widget

import android.app.AlarmManager
import android.app.PendingIntent
import android.appwidget.AppWidgetManager
import android.appwidget.AppWidgetProvider
import android.content.ComponentName
import android.content.Context
import android.content.Intent
import android.os.Build
import android.os.SystemClock
import android.widget.RemoteViews
import java.util.concurrent.Executors

class NamazWidgetProvider : AppWidgetProvider() {

    override fun onUpdate(
        context: Context,
        appWidgetManager: AppWidgetManager,
        appWidgetIds: IntArray
    ) {
        updateAllWidgets(context, forceFetch = false)
    }

    override fun onReceive(context: Context, intent: Intent) {
        super.onReceive(context, intent)
        when (intent.action) {
            ACTION_TICK_UPDATE,
            Intent.ACTION_BOOT_COMPLETED,
            Intent.ACTION_TIME_SET,
            Intent.ACTION_TIMEZONE_CHANGED -> {
                updateAllWidgets(context, forceFetch = false)
            }
        }
    }

    companion object {
        const val ACTION_TICK_UPDATE = "com.namazvakti.widget.ACTION_TICK_UPDATE"
        private val executor = Executors.newSingleThreadExecutor()

        fun updateAllWidgets(context: Context, forceFetch: Boolean) {
            val appContext = context.applicationContext
            val manager = AppWidgetManager.getInstance(appContext)
            val component = ComponentName(appContext, NamazWidgetProvider::class.java)
            val widgetIds = manager.getAppWidgetIds(component)
            if (widgetIds.isEmpty()) return

            val location = PrayerPrefs.getLocation(appContext)
            if (location == null) {
                renderUnconfigured(appContext, manager, widgetIds)
                return
            }

            val cachedJson = PrayerPrefs.getCachedSchedule(appContext)
            val cachedNext = cachedJson?.let { runCatching { PrayerCalculator.computeNextPrayer(it) }.getOrNull() }

            if (cachedNext != null) {
                renderPrayerInfo(appContext, manager, widgetIds, location, cachedNext)
                scheduleNextUpdate(appContext, cachedNext.targetEpochMs + 1_500L)
            }

            if (forceFetch || cachedNext == null || PrayerPrefs.isCacheStale(appContext)) {
                executor.execute {
                    try {
                        val freshJson = DiyanetApi.fetchPrayerTimesRawJson(location.districtId)
                        PrayerPrefs.saveCachedSchedule(appContext, freshJson)
                        val freshNext = PrayerCalculator.computeNextPrayer(freshJson)
                        if (freshNext != null) {
                            renderPrayerInfo(appContext, manager, widgetIds, location, freshNext)
                            scheduleNextUpdate(appContext, freshNext.targetEpochMs + 1_500L)
                        }
                    } catch (_: Exception) {
                        if (cachedNext == null) {
                            renderError(appContext, manager, widgetIds, location)
                        }
                    }
                }
            }
        }

        private fun renderUnconfigured(
            context: Context,
            manager: AppWidgetManager,
            widgetIds: IntArray
        ) {
            val views = RemoteViews(context.packageName, R.layout.widget_namaz)
            views.setTextViewText(R.id.tv_location_date, context.getString(R.string.select_city_hint))
            views.setTextViewText(R.id.tv_prayer_name, "Konum seçilmedi")
            views.setTextViewText(R.id.tv_prayer_time, "--:--")
            views.setChronometer(R.id.ch_countdown, SystemClock.elapsedRealtime(), "Dokunup şehir seçin", false)
            views.setOnClickPendingIntent(R.id.widget_root, createConfigPendingIntent(context))
            widgetIds.forEach { manager.updateAppWidget(it, views) }
        }

        private fun renderError(
            context: Context,
            manager: AppWidgetManager,
            widgetIds: IntArray,
            location: SavedLocation
        ) {
            val views = RemoteViews(context.packageName, R.layout.widget_namaz)
            views.setTextViewText(R.id.tv_location_date, location.displayLabel())
            views.setTextViewText(R.id.tv_prayer_name, "Bağlantı bekleniyor")
            views.setTextViewText(R.id.tv_prayer_time, "--:--")
            views.setChronometer(R.id.ch_countdown, SystemClock.elapsedRealtime(), "Güncellemek için dokunun", false)
            views.setOnClickPendingIntent(R.id.widget_root, createConfigPendingIntent(context))
            widgetIds.forEach { manager.updateAppWidget(it, views) }
        }

        private fun renderPrayerInfo(
            context: Context,
            manager: AppWidgetManager,
            widgetIds: IntArray,
            location: SavedLocation,
            info: NextPrayerInfo
        ) {
            val views = RemoteViews(context.packageName, R.layout.widget_namaz)
            views.setTextViewText(R.id.tv_location_date, "${location.displayLabel()} · ${info.formattedDate}")
            views.setTextViewText(R.id.tv_prayer_name, info.prayerName)
            views.setTextViewText(R.id.tv_prayer_time, info.prayerTimeText)

            val countDownBase = SystemClock.elapsedRealtime() + info.remainingMs
            views.setChronometerCountDown(R.id.ch_countdown, true)
            views.setChronometer(R.id.ch_countdown, countDownBase, "%s sonra başlıyor", true)

            views.setOnClickPendingIntent(R.id.widget_root, createConfigPendingIntent(context))
            widgetIds.forEach { manager.updateAppWidget(it, views) }
        }

        private fun createConfigPendingIntent(context: Context): PendingIntent {
            val intent = Intent(context, CitySelectActivity::class.java).apply {
                flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP
            }
            val flags = PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
            return PendingIntent.getActivity(context, 1001, intent, flags)
        }

        private fun scheduleNextUpdate(context: Context, triggerAtEpochMs: Long) {
            val alarmManager = context.getSystemService(Context.ALARM_SERVICE) as? AlarmManager ?: return
            val intent = Intent(context, NamazWidgetProvider::class.java).apply {
                action = ACTION_TICK_UPDATE
            }
            val flags = PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
            val pendingIntent = PendingIntent.getBroadcast(context, 2001, intent, flags)

            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                alarmManager.setAndAllowWhileIdle(AlarmManager.RTC, triggerAtEpochMs, pendingIntent)
            } else {
                alarmManager.set(AlarmManager.RTC, triggerAtEpochMs, pendingIntent)
            }
        }
    }
}
EOF_FILE

mkdir -p "app/src/main/java/com/namazvakti/widget"
cat << 'EOF_FILE' > "app/src/main/java/com/namazvakti/widget/PrayerCalculator.kt"
package com.namazvakti.widget

import org.json.JSONArray
import org.json.JSONObject
import java.text.SimpleDateFormat
import java.util.Calendar
import java.util.Date
import java.util.Locale

data class NextPrayerInfo(
    val prayerName: String,
    val prayerTimeText: String,
    val targetEpochMs: Long,
    val remainingMs: Long,
    val formattedDate: String
)

object PrayerCalculator {
    private val TR_LOCALE = Locale("tr", "TR")
    private val SHORT_DATE_FMT = SimpleDateFormat("dd.MM.yyyy", TR_LOCALE)
    private val DISPLAY_DATE_FMT = SimpleDateFormat("d MMMM yyyy", TR_LOCALE)

    private val PRAYER_KEYS = listOf(
        "Imsak" to "İmsak",
        "Gunes" to "Güneş",
        "Ogle" to "Öğle",
        "Ikindi" to "İkindi",
        "Aksam" to "Akşam",
        "Yatsi" to "Yatsı"
    )

    fun computeNextPrayer(rawJson: String, nowMs: Long = System.currentTimeMillis()): NextPrayerInfo? {
        val array = JSONArray(rawJson)
        val nowCal = Calendar.getInstance().apply { timeInMillis = nowMs }
        val todayKey = SHORT_DATE_FMT.format(nowCal.time)

        val tomorrowCal = (nowCal.clone() as Calendar).apply { add(Calendar.DAY_OF_YEAR, 1) }
        val tomorrowKey = SHORT_DATE_FMT.format(tomorrowCal.time)

        var todayObj: JSONObject? = null
        var tomorrowObj: JSONObject? = null

        for (i in 0 until array.length()) {
            val item = array.getJSONObject(i)
            when (item.optString("MiladiTarihKisa")) {
                todayKey -> todayObj = item
                tomorrowKey -> tomorrowObj = item
            }
        }

        val displayDate = DISPLAY_DATE_FMT.format(Date(nowMs))

        if (todayObj != null) {
            for ((jsonKey, label) in PRAYER_KEYS) {
                val hhmm = todayObj.optString(jsonKey, "")
                val targetMs = parseTimeOnCalendar(nowCal, hhmm) ?: continue
                if (targetMs > nowMs) {
                    return NextPrayerInfo(
                        prayerName = label,
                        prayerTimeText = hhmm,
                        targetEpochMs = targetMs,
                        remainingMs = targetMs - nowMs,
                        formattedDate = displayDate
                    )
                }
            }
        }

        if (tomorrowObj != null) {
            val imsak = tomorrowObj.optString("Imsak", "")
            val targetMs = parseTimeOnCalendar(tomorrowCal, imsak)
            if (targetMs != null && targetMs > nowMs) {
                return NextPrayerInfo(
                    prayerName = "İmsak",
                    prayerTimeText = imsak,
                    targetEpochMs = targetMs,
                    remainingMs = targetMs - nowMs,
                    formattedDate = displayDate
                )
            }
        }

        return null
    }

    private fun parseTimeOnCalendar(dayCal: Calendar, hhmm: String): Long? {
        val parts = hhmm.trim().split(":")
        if (parts.size != 2) return null
        val hour = parts[0].toIntOrNull() ?: return null
        val minute = parts[1].toIntOrNull() ?: return null

        val cal = dayCal.clone() as Calendar
        cal.set(Calendar.HOUR_OF_DAY, hour)
        cal.set(Calendar.MINUTE, minute)
        cal.set(Calendar.SECOND, 0)
        cal.set(Calendar.MILLISECOND, 0)
        return cal.timeInMillis
    }
}
EOF_FILE

mkdir -p "app/src/main/java/com/namazvakti/widget"
cat << 'EOF_FILE' > "app/src/main/java/com/namazvakti/widget/PrayerPrefs.kt"
package com.namazvakti.widget

import android.content.Context
import java.util.Locale

data class SavedLocation(
    val cityId: String,
    val cityName: String,
    val districtId: String,
    val districtName: String
) {
    fun displayLabel(): String {
        val city = PrayerPrefs.toTurkishTitleCase(cityName)
        val district = PrayerPrefs.toTurkishTitleCase(districtName)
        return "$city / $district"
    }
}

object PrayerPrefs {
    private const val PREFS_NAME = "namaz_vakti_prefs"
    private const val KEY_CITY_ID = "city_id"
    private const val KEY_CITY_NAME = "city_name"
    private const val KEY_DISTRICT_ID = "district_id"
    private const val KEY_DISTRICT_NAME = "district_name"
    private const val KEY_CACHED_JSON = "cached_json"
    private const val KEY_LAST_FETCH_MS = "last_fetch_ms"

    private val TR_LOCALE = Locale("tr", "TR")

    fun saveLocation(context: Context, location: SavedLocation) {
        context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE)
            .edit()
            .putString(KEY_CITY_ID, location.cityId)
            .putString(KEY_CITY_NAME, location.cityName)
            .putString(KEY_DISTRICT_ID, location.districtId)
            .putString(KEY_DISTRICT_NAME, location.districtName)
            .apply()
    }

    fun getLocation(context: Context): SavedLocation? {
        val prefs = context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE)
        val cityId = prefs.getString(KEY_CITY_ID, null) ?: return null
        val cityName = prefs.getString(KEY_CITY_NAME, null) ?: return null
        val districtId = prefs.getString(KEY_DISTRICT_ID, null) ?: return null
        val districtName = prefs.getString(KEY_DISTRICT_NAME, null) ?: return null
        return SavedLocation(cityId, cityName, districtId, districtName)
    }

    fun saveCachedSchedule(context: Context, rawJson: String) {
        context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE)
            .edit()
            .putString(KEY_CACHED_JSON, rawJson)
            .putLong(KEY_LAST_FETCH_MS, System.currentTimeMillis())
            .apply()
    }

    fun getCachedSchedule(context: Context): String? {
        return context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE)
            .getString(KEY_CACHED_JSON, null)
    }

    fun isCacheStale(context: Context): Boolean {
        val last = context.getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE)
            .getLong(KEY_LAST_FETCH_MS, 0L)
        val twelveHoursMs = 12L * 60L * 60L * 1000L
        return System.currentTimeMillis() - last > twelveHoursMs
    }

    fun toTurkishTitleCase(input: String): String {
        return input.trim()
            .lowercase(TR_LOCALE)
            .split(" ")
            .filter { it.isNotEmpty() }
            .joinToString(" ") { word ->
                word.replaceFirstChar { ch -> ch.titlecase(TR_LOCALE) }
            }
    }
}
EOF_FILE

mkdir -p "app/src/main/res/drawable"
cat << 'EOF_FILE' > "app/src/main/res/drawable/button_bg.xml"
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle">
    <solid android:color="@color/accent" />
    <corners android:radius="14dp" />
</shape>
EOF_FILE

mkdir -p "app/src/main/res/drawable"
cat << 'EOF_FILE' > "app/src/main/res/drawable/ic_launcher_background.xml"
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle">
    <solid android:color="#1E1F22" />
</shape>
EOF_FILE

mkdir -p "app/src/main/res/drawable"
cat << 'EOF_FILE' > "app/src/main/res/drawable/ic_launcher_foreground.xml"
<?xml version="1.0" encoding="utf-8"?>
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="108dp"
    android:height="108dp"
    android:viewportWidth="108"
    android:viewportHeight="108">
    <path
        android:fillColor="#E8EAED"
        android:pathData="M59,28C45.7,28 35,38.7 35,52C35,65.3 45.7,76 59,76C65.5,76 71.4,73.4 75.7,69.2C73.3,69.8 70.8,70.2 68.2,70.2C56.6,70.2 47.2,60.8 47.2,49.2C47.2,40.1 53,32.3 61.1,29.4C60.4,28.5 59.7,28 59,28Z" />
</vector>
EOF_FILE

mkdir -p "app/src/main/res/drawable"
cat << 'EOF_FILE' > "app/src/main/res/drawable/spinner_bg.xml"
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle">
    <solid android:color="#27292E" />
    <corners android:radius="12dp" />
    <stroke
        android:width="1dp"
        android:color="#3A3D45" />
</shape>
EOF_FILE

mkdir -p "app/src/main/res/drawable"
cat << 'EOF_FILE' > "app/src/main/res/drawable/widget_bg.xml"
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle">
    <solid android:color="@color/widget_bg" />
    <corners android:radius="22dp" />
    <stroke
        android:width="1dp"
        android:color="@color/widget_border" />
</shape>
EOF_FILE

mkdir -p "app/src/main/res/layout"
cat << 'EOF_FILE' > "app/src/main/res/layout/activity_city_select.xml"
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@color/screen_bg"
    android:gravity="center_horizontal"
    android:orientation="vertical"
    android:padding="24dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="32dp"
        android:background="@drawable/widget_bg"
        android:orientation="vertical"
        android:padding="22dp">

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:fontFamily="sans-serif-medium"
            android:text="Şehir ve İlçe Seçimi"
            android:textColor="@color/text_primary"
            android:textSize="22sp" />

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="6dp"
            android:text="Diyanet namaz vakitleri seçtiğiniz il ve ilçe için ana ekran aracında (widget) gösterilir."
            android:textColor="@color/text_secondary"
            android:textSize="14sp" />

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="22dp"
            android:text="Şehir (İl)"
            android:textColor="@color/text_secondary"
            android:textSize="13sp" />

        <Spinner
            android:id="@+id/spinner_city"
            android:layout_width="match_parent"
            android:layout_height="50dp"
            android:layout_marginTop="6dp"
            android:background="@drawable/spinner_bg"
            android:paddingStart="12dp"
            android:paddingEnd="12dp"
            android:popupBackground="@color/card_bg" />

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="16dp"
            android:text="İlçe"
            android:textColor="@color/text_secondary"
            android:textSize="13sp" />

        <Spinner
            android:id="@+id/spinner_district"
            android:layout_width="match_parent"
            android:layout_height="50dp"
            android:layout_marginTop="6dp"
            android:background="@drawable/spinner_bg"
            android:paddingStart="12dp"
            android:paddingEnd="12dp"
            android:popupBackground="@color/card_bg" />

        <ProgressBar
            android:id="@+id/progress_bar"
            style="?android:attr/progressBarStyleSmall"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center_horizontal"
            android:layout_marginTop="16dp"
            android:visibility="gone" />

        <TextView
            android:id="@+id/tv_status"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="10dp"
            android:gravity="center"
            android:textColor="@color/text_secondary"
            android:textSize="13sp"
            android:visibility="gone" />

        <Button
            android:id="@+id/btn_save"
            android:layout_width="match_parent"
            android:layout_height="50dp"
            android:layout_marginTop="20dp"
            android:background="@drawable/button_bg"
            android:fontFamily="sans-serif-medium"
            android:text="Kaydet"
            android:textAllCaps="false"
            android:textColor="@color/button_text"
            android:textSize="16sp" />

    </LinearLayout>

</LinearLayout>
EOF_FILE

mkdir -p "app/src/main/res/layout"
cat << 'EOF_FILE' > "app/src/main/res/layout/item_spinner.xml"
<?xml version="1.0" encoding="utf-8"?>
<TextView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:ellipsize="end"
    android:maxLines="1"
    android:paddingStart="6dp"
    android:paddingTop="12dp"
    android:paddingEnd="6dp"
    android:paddingBottom="12dp"
    android:textColor="@color/text_primary"
    android:textSize="16sp" />
EOF_FILE

mkdir -p "app/src/main/res/layout"
cat << 'EOF_FILE' > "app/src/main/res/layout/item_spinner_dropdown.xml"
<?xml version="1.0" encoding="utf-8"?>
<TextView xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:background="@color/card_bg"
    android:ellipsize="end"
    android:maxLines="1"
    android:paddingStart="16dp"
    android:paddingTop="14dp"
    android:paddingEnd="16dp"
    android:paddingBottom="14dp"
    android:textColor="@color/text_primary"
    android:textSize="16sp" />
EOF_FILE

mkdir -p "app/src/main/res/layout"
cat << 'EOF_FILE' > "app/src/main/res/layout/widget_namaz.xml"
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/widget_root"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@drawable/widget_bg"
    android:orientation="vertical"
    android:paddingStart="20dp"
    android:paddingTop="16dp"
    android:paddingEnd="20dp"
    android:paddingBottom="16dp">

    <TextView
        android:id="@+id/tv_title"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:fontFamily="sans-serif-medium"
        android:maxLines="1"
        android:text="@string/widget_title"
        android:textColor="@color/text_primary"
        android:textSize="22sp" />

    <TextView
        android:id="@+id/tv_location_date"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="4dp"
        android:ellipsize="end"
        android:fontFamily="sans-serif"
        android:maxLines="1"
        android:text="@string/select_city_hint"
        android:textColor="@color/text_secondary"
        android:textSize="14sp" />

    <ImageView
        android:layout_width="match_parent"
        android:layout_height="1dp"
        android:layout_marginTop="14dp"
        android:layout_marginBottom="14dp"
        android:background="@color/widget_divider"
        android:contentDescription="@null" />

    <TextView
        android:id="@+id/tv_prayer_name"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:fontFamily="sans-serif"
        android:maxLines="1"
        android:text="--"
        android:textColor="@color/text_secondary"
        android:textSize="16sp" />

    <TextView
        android:id="@+id/tv_prayer_time"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="2dp"
        android:fontFamily="sans-serif-light"
        android:includeFontPadding="false"
        android:maxLines="1"
        android:text="--:--"
        android:textColor="@color/text_primary"
        android:textSize="48sp" />

    <Chronometer
        android:id="@+id/ch_countdown"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="4dp"
        android:countDown="true"
        android:fontFamily="sans-serif"
        android:maxLines="1"
        android:textColor="@color/text_secondary"
        android:textSize="16sp" />

</LinearLayout>
EOF_FILE

mkdir -p "app/src/main/res/mipmap-anydpi-v26"
cat << 'EOF_FILE' > "app/src/main/res/mipmap-anydpi-v26/ic_launcher.xml"
<?xml version="1.0" encoding="utf-8"?>
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@drawable/ic_launcher_background" />
    <foreground android:drawable="@drawable/ic_launcher_foreground" />
</adaptive-icon>
EOF_FILE

mkdir -p "app/src/main/res/mipmap-xxhdpi"
base64 -d << 'EOF_B64' > "app/src/main/res/mipmap-xxhdpi/ic_launcher.png"
iVBORw0KGgoAAAANSUhEUgAAAJAAAACQCAYAAADnRuK4AAADKUlEQVR4nO3dQU7kMBRF0U+r1wESe6D3P6YWRY/SiqKChnqx/d7PPSOGTnz5doEQT88vrx8FPOjX6gUgGwFBQkCQEBAkBAQJAUFCQJAQECQEBAkBQUJAkBAQJAQECQFBQkCQEBAkBAQJAUFCQJAQECQEBAkBQUJAkBAQJAQECQFBQkCQEBAkBAQJAUFCQJAQECS/Vy9glffb7d/Xf97eFq4k26UC2kdTRThnaB/QMZoN8ZyjbUCfhVNFPGdqF9BX4VQRz9lafQojnvlaTKD/hVNFPKPET6DvxINxogP6bjxMn3Eij7CfTB3iGStuAhGPl6iAuO/4iQnop/EwfeaICQieIgJi+viyD4h7jzfrgB6Jh+kzl3VA8GcbEEdXBsuAHo2H42s+y4CQwy4gjq4sdgEhi1VATJ88VgEpuECv0SYgrGETEMdXJpuAkImAILEIiOMrl0VAyEVAkBAQJMsD4v6TbXlAyNYmICbZGm0CwhoEBAkBQUJAkLQKiIv0fK0CwnwEBEm7gDjG5moXEOZqGRBTaJ7lAfHXFNmWBzQKU2iOtgFhjtYBMYXGswho5D2IiMayCGg0IhrnEgFhHJuARn+cZwqNYRPQDER0vksFVEVEZ7MKaNZPpYnoPFYBzURE53h6fnn9WL2Io9mby+/jHnfZCbSXMo0c12kZ0IqJ4Lg5e++3m+WktDzCNqs21WmjtnfgtKa9yH+6O5rDprlPxI31BKryeJEzQ7r3vK7TpyogoCqPiDYjNvOr53OOpyokoCqviI5+ssnd/m05d6ATOMc9muXH+HsSvhvPlPK8MQFV5bxUVdJzRgVUlfVyH5H2fDGX6Hs63T3SwtnETaC91Jd+lPwc0QFVZb/8qvz1Rx9hR0lHWno4m/gJtJeyKSnr/I5WE2jPcRp1CmfTNqCNQ0gdw9m0D2hvZkydo9m7VEB7I2K6SjR7lw3o6JGgrhjMEQFB0upjPOYjIEgICBICgoSAICEgSAgIEgKChIAgISBICAgSAoKEgCAhIEgICBICgoSAICEgSAgIEgKChIAgISBICAgSAoKEgCAhIEgICBICgoSAICEgSP4CgSi/qRuAnSIAAAAASUVORK5CYII=
EOF_B64

mkdir -p "app/src/main/res/values"
cat << 'EOF_FILE' > "app/src/main/res/values/colors.xml"
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <color name="widget_bg">#1E1F22</color>
    <color name="widget_border">#2E3036</color>
    <color name="widget_divider">#35383F</color>
    <color name="text_primary">#E8EAED</color>
    <color name="text_secondary">#B0B3B8</color>
    <color name="accent">#8AB4F8</color>
    <color name="button_text">#141619</color>
    <color name="screen_bg">#141518</color>
    <color name="card_bg">#1E1F22</color>
</resources>
EOF_FILE

mkdir -p "app/src/main/res/values"
cat << 'EOF_FILE' > "app/src/main/res/values/strings.xml"
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">Namaz Vakitleri</string>
    <string name="widget_title">Namaz vakitleri</string>
    <string name="widget_description">Diyanet namaz vakitlerini anlık geri sayımla gösterir.</string>
    <string name="select_city_hint">İl ve ilçe seçmek için dokunun</string>
</resources>
EOF_FILE

mkdir -p "app/src/main/res/values"
cat << 'EOF_FILE' > "app/src/main/res/values/themes.xml"
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <style name="Theme.NamazVaktiWidget" parent="android:Theme.DeviceDefault.NoActionBar">
        <item name="android:windowBackground">@color/screen_bg</item>
        <item name="android:statusBarColor">@color/screen_bg</item>
        <item name="android:navigationBarColor">@color/screen_bg</item>
        <item name="android:textColorPrimary">@color/text_primary</item>
        <item name="android:textColorSecondary">@color/text_secondary</item>
    </style>
</resources>
EOF_FILE

mkdir -p "app/src/main/res/xml"
cat << 'EOF_FILE' > "app/src/main/res/xml/namaz_widget_info.xml"
<?xml version="1.0" encoding="utf-8"?>
<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
    android:configure="com.namazvakti.widget.CitySelectActivity"
    android:description="@string/widget_description"
    android:initialLayout="@layout/widget_namaz"
    android:minHeight="130dp"
    android:minResizeHeight="110dp"
    android:minResizeWidth="180dp"
    android:minWidth="220dp"
    android:resizeMode="horizontal|vertical"
    android:targetCellHeight="2"
    android:targetCellWidth="4"
    android:updatePeriodMillis="1800000"
    android:widgetCategory="home_screen" />
EOF_FILE

cat << 'EOF_FILE' > "build.gradle.kts"
plugins {
    id("com.android.application") version "8.7.3" apply false
    id("org.jetbrains.kotlin.android") version "2.0.21" apply false
}
EOF_FILE

cat << 'EOF_FILE' > "gradle.properties"
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
kotlin.code.style=official
android.nonTransitiveRClass=true
EOF_FILE

cat << 'EOF_FILE' > "settings.gradle.kts"
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}
rootProject.name = "NamazVaktiWidget"
include(":app")
EOF_FILE

git add .
git commit -m "Namaz Vakitleri Widget"
git push
echo "Tamamlandi! Simdi GitHub Actions sekmesinden APK dosyasini indirebilirsiniz."
```
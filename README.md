build.gradle.kts ===
plugins {
    id("com.android.application") version "8.5.2" apply false
    id("org.jetbrains.kotlin.android") version "1.9.24" apply false
}gradle.properties ===
-e android.useAndroidX=true
org.gradle.jvmargs=-Xmx2g

app/build.gradle.kts ===
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}
android {
    namespace = "com.jarvis.app"
    compileSdk = 34
    defaultConfig {
        applicationId = "com.jarvis.app"
        minSdk = 26
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"
    }
    compileOptions { sourceCompatibility = JavaVersion.VERSION_17; targetCompatibility = JavaVersion.VERSION_17 }
    kotlinOptions { jvmTarget = "17" }
}app/src/main/AndroidManifest.xml ===
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.READ_CONTACTS"/>
    <queries>
        <intent><action android:name="android.intent.action.MAIN"/><category android:name="android.intent.category.LAUNCHER"/></intent>
        <intent><action android:name="android.speech.action.RECOGNIZE_SPEECH"/></intent>
    </queries>
    <application android:label="J.A.R.V.I.S." android:theme="@android:style/Theme.Material.NoActionBar">
        <activity android:name=".MainActivity" android:exported="true" android:windowSoftInputMode="adjustResize">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
</manifest>

app/src/main/java/com/jarvis/app/MainActivity.kt ===
package com.jarvis.app

import android.Manifest
import android.app.AlertDialog
import android.app.SearchManager
import android.content.*
import android.content.pm.PackageManager
import android.graphics.Color
import android.hardware.camera2.CameraManager
import android.media.AudioManager
import android.os.BatteryManager
import android.os.Bundle
import android.os.Environment
import android.os.StatFs
import android.provider.AlarmClock
import android.provider.ContactsContract
import android.provider.Settings
import android.speech.RecognizerIntent
import android.speech.tts.TextToSpeech
import android.view.Gravity
import android.view.inputmethod.EditorInfo
import android.widget.*
import androidx.core.app.ActivityCompat
import org.json.JSONArray
import org.json.JSONObject
import java.net.HttpURLConnection
import java.net.URL
import java.util.*
import kotlin.concurrent.thread

class MainActivity : android.app.Activity(), TextToSpeech.OnInitListener {
    private lateinit var tts: TextToSpeech
    private lateinit var log: TextView
    private lateinit var scroll: ScrollView
    private lateinit var input: EditText
    private var torch = false
    private val hist = JSONArray()
    private val prefs by lazy { getSharedPreferences("jv", MODE_PRIVATE) }

    override fun onCreate(b: Bundle?) {
        super.onCreate(b)
        tts = TextToSpeech(this, this)
        val root = LinearLayout(this).apply { orientation = LinearLayout.VERTICAL; setBackgroundColor(Color.parseColor("#06121c")); setPadding(24, 48, 24, 24) }
        val title = TextView(this).apply { text = "J.A.R.V.I.S."; textSize = 26f; setTextColor(Color.parseColor("#4fdcff")); gravity = Gravity.CENTER }
        log = TextView(this).apply { textSize = 16f; setTextColor(Color.parseColor("#e6f4fb")); setPadding(8, 16, 8, 16) }
        scroll = ScrollView(this).apply { addView(log) }
        input = EditText(this).apply { hint = "Komut yaz..."; setHintTextColor(Color.GRAY); setTextColor(Color.WHITE); imeOptions = EditorInfo.IME_ACTION_SEND; maxLines = 1; inputType = android.text.InputType.TYPE_CLASS_TEXT
            setOnEditorActionListener { _, _, _ -> send(); true } }
        fun btn(t: String, f: () -> Unit) = Button(this).apply { text = t; setOnClickListener { f() } }
        val row = LinearLayout(this).apply { orientation = LinearLayout.HORIZONTAL
            addView(input, LinearLayout.LayoutParams(0, -2, 1f))
            addView(btn("🎙️") { listen() }); addView(btn("➤") { send() }); addView(btn("🔑") { keyDialog() }) }
        root.addView(title); root.addView(scroll, LinearLayout.LayoutParams(-1, 0, 1f)); root.addView(row)
        setContentView(root)
        ActivityCompat.requestPermissions(this, arrayOf(Manifest.permission.READ_CONTACTS), 1)
        say("Sistemler çevrimiçi, efendim. Telefonunuz emrinizde.")
    }

    override fun onInit(s: Int) { if (s == TextToSpeech.SUCCESS) { tts.language = Locale("tr", "TR"); tts.setPitch(0.6f); tts.setSpeechRate(0.92f) } }

    private fun say(t: String) { log.append("\nJARVIS: $t\n"); scroll.post { scroll.fullScroll(android.view.View.FOCUS_DOWN) }; tts.speak(t, TextToSpeech.QUEUE_FLUSH, null, "j") }
    private fun send() { val q = input.text.toString().trim(); if (q.isEmpty()) return; input.setText(""); log.append("\nSiz: $q\n"); handle(q) }
    private fun listen() { try { startActivityForResult(Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM).putExtra(RecognizerIntent.EXTRA_LANGUAGE, "tr-TR"), 7) } catch (e: Exception) { say("Ses tanıma bu cihazda yok.") } }
    override fun onActivityResult(r: Int, c: Int, d: Intent?) { super.onActivityResult(r, c, d); if (r == 7 && c == RESULT_OK) d?.getStringArrayListExtra(RecognizerIntent.EXTRA_RESULTS)?.firstOrNull()?.let { log.append("\nSiz: $it\n"); handle(it) } }

    private fun keyDialog() {
        val e = EditText(this).apply { hint = "Anthropic API anahtarı (sk-ant-...)"; setText(prefs.getString("key", "")) }
        AlertDialog.Builder(this).setTitle("Yapay zekâ sohbeti için anahtar").setView(e).setPositiveButton("Kaydet") { _, _ -> prefs.edit().putString("key", e.text.toString().trim()).apply(); say("Anahtar kaydedildi.") }.show()
    }

    private fun handle(q: String) {
        val l = q.lowercase(Locale("tr"))
        val num = Regex("\\d+").findAll(l).map { it.value.toInt() }.toList()
        val n = l.replace("'", "").replace("’", "")
        try { when {
            "youtube" in n && ("ara" in n || "bul" in n) -> ytSearch(cleanQ(n))
            "google" in n && ("ara" in n || "bul" in n) -> gSearch(cleanQ(n))
            "tara" in l || "durum" in l -> scan()
            "pil" in l || "şarj" in l -> say("Pil yüzde ${battery()}.")
            "saat" in l -> say("Saat ${java.text.SimpleDateFormat("HH:mm", Locale("tr")).format(Date())}.")
            "alarm" in l && num.isNotEmpty() -> { startActivity(Intent(AlarmClock.ACTION_SET_ALARM).putExtra(AlarmClock.EXTRA_HOUR, num[0]).putExtra(AlarmClock.EXTRA_MINUTES, num.getOrElse(1) { 0 }).putExtra(AlarmClock.EXTRA_SKIP_UI, true)); say("Alarm ${num[0]}:${"%02d".format(num.getOrElse(1) { 0 })} için kuruldu.") }
            ("zamanlayıcı" in l || "sayaç" in l) && num.isNotEmpty() -> { startActivity(Intent(AlarmClock.ACTION_SET_TIMER).putExtra(AlarmClock.EXTRA_LENGTH, num[0] * 60).putExtra(AlarmClock.EXTRA_SKIP_UI, true)); say("${num[0]} dakikalık zamanlayıcı başladı.") }
            "fener" in l -> { torch = !("kapat" in l); (getSystemService(CAMERA_SERVICE) as CameraManager).let { it.setTorchMode(it.cameraIdList[0], torch) }; say(if (torch) "Fener açık." else "Fener kapalı.") }
            "sesi aç" in l || "sesi artır" in l -> { (getSystemService(AUDIO_SERVICE) as AudioManager).adjustStreamVolume(AudioManager.STREAM_MUSIC, AudioManager.ADJUST_RAISE, AudioManager.FLAG_SHOW_UI); say("Ses yükseltildi.") }
            "sesi kıs" in l || "sesi azalt" in l -> { (getSystemService(AUDIO_SERVICE) as AudioManager).adjustStreamVolume(AudioManager.STREAM_MUSIC, AudioManager.ADJUST_LOWER, AudioManager.FLAG_SHOW_UI); say("Ses kısıldı.") }
            "wifi" in l || "wi-fi" in l -> { startActivity(Intent(Settings.ACTION_WIFI_SETTINGS)); say("Wi-Fi ayarları açıldı.") }
            "bluetooth" in l -> { startActivity(Intent(Settings.ACTION_BLUETOOTH_SETTINGS)); say("Bluetooth ayarları açıldı.") }
            l.startsWith("ara ") -> call(l.removePrefix("ara ").trim())
            l.endsWith(" aç") || l.startsWith("aç ") -> openApp(l.removeSuffix(" aç").removePrefix("aç ").trim())
            else -> ai(q)
        } } catch (e: Exception) { say("Bir aksaklık oldu: ${e.message}") }
    }

    private fun cleanQ(n: String) = n.replace(Regex("youtube\\w*|google\\w*|arama yap\\w*|\\bara\\w*|\\bbul\\w*"), "").replace(Regex("\\s+"), " ").trim()

    private fun ytSearch(q: String) {
        if (q.isEmpty()) { say("YouTube'da ne arayayım, efendim?"); return }
        try { startActivity(Intent(Intent.ACTION_SEARCH).setPackage("com.google.android.youtube").putExtra("query", q)) }
        catch (e: Exception) { startActivity(Intent(Intent.ACTION_VIEW, android.net.Uri.parse("https://www.youtube.com/results?search_query=" + android.net.Uri.encode(q)))) }
        say("YouTube'da \"$q\" aranıyor.")
    }

    private fun gSearch(q: String) {
        if (q.isEmpty()) { say("Google'da ne arayayım, efendim?"); return }
        try { startActivity(Intent(Intent.ACTION_WEB_SEARCH).putExtra(SearchManager.QUERY, q)) }
        catch (e: Exception) { startActivity(Intent(Intent.ACTION_VIEW, android.net.Uri.parse("https://www.google.com/search?q=" + android.net.Uri.encode(q)))) }
        say("Google'da \"$q\" aranıyor.")
    }

    private fun battery(): Int = (getSystemService(BATTERY_SERVICE) as BatteryManager).getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)

    private fun apps() = packageManager.queryIntentActivities(Intent(Intent.ACTION_MAIN).addCategory(Intent.CATEGORY_LAUNCHER), 0)

    private fun openApp(name: String) {
        val m = apps().firstOrNull { it.loadLabel(packageManager).toString().lowercase(Locale("tr")).contains(name) }
        val i = m?.let { packageManager.getLaunchIntentForPackage(it.activityInfo.packageName) }
        if (i != null) { startActivity(i); say("${m.loadLabel(packageManager)} açılıyor.") } else say("\"$name\" adında uygulama bulamadım.")
    }

    private fun call(name: String) {
        if (checkSelfPermission(Manifest.permission.READ_CONTACTS) != PackageManager.PERMISSION_GRANTED) { say("Rehber izni gerekli."); return }
        val c = contentResolver.query(ContactsContract.CommonDataKinds.Phone.CONTENT_URI, arrayOf(ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME, ContactsContract.CommonDataKinds.Phone.NUMBER), null, null, null)
        c?.use { while (it.moveToNext()) if (it.getString(0).lowercase(Locale("tr")).contains(name)) { startActivity(Intent(Intent.ACTION_DIAL, android.net.Uri.parse("tel:" + it.getString(1)))); say("${it.getString(0)} aranıyor."); return } }
        say("Rehberde \"$name\" yok.")
    }

    private fun scan() {
        val s = StatFs(Environment.getDataDirectory().path)
        val free = s.availableBytes / 1073741824; val tot = s.totalBytes / 1073741824
        var contacts = "izin yok"
        if (checkSelfPermission(Manifest.permission.READ_CONTACTS) == PackageManager.PERMISSION_GRANTED)
            contentResolver.query(ContactsContract.Contacts.CONTENT_URI, null, null, null, null)?.use { contacts = it.count.toString() }
        say("Tarama: pil %${battery()}, ${apps().size} uygulama, depolama ${free} GB boş / ${tot} GB, rehberde $contacts kişi.")
    }

    private fun ai(q: String) {
        val key = prefs.getString("key", "") ?: ""
        if (key.isEmpty()) { say("Bu komutu anlamadım. Serbest sohbet için 🔑 ile API anahtarı girin."); return }
        hist.put(JSONObject().put("role", "user").put("content", q))
        thread {
            val reply = try {
                val c = URL("https://api.anthropic.com/v1/messages").openConnection() as HttpURLConnection
                c.requestMethod = "POST"; c.doOutput = true
                c.setRequestProperty("x-api-key", key); c.setRequestProperty("anthropic-version", "2023-06-01"); c.setRequestProperty("content-type", "application/json")
                val body = JSONObject().put("model", "claude-sonnet-5-5").put("max_tokens", 300)
                    .put("system", "Sen J.A.R.V.I.S.'sin, telefondaki sadık, soğukkanlı, hafif iğneleyici asistan. Kullanıcıya 'efendim' de. Türkçe, en fazla 3 kısa cümle. Canlı veri uydurma.")
                    .put("messages", hist)
                c.outputStream.write(body.toString().toByteArray())
                val r = JSONObject(c.inputStream.bufferedReader().readText())
                r.getJSONArray("content").getJSONObject(0).getString("text")
            } catch (e: Exception) { "Bağlantı kurulamadı." }
            hist.put(JSONObject().put("role", "assistant").put("content", reply))
            runOnUiThread { say(reply) }
        }
    }
}# J.a.v.i.r.s
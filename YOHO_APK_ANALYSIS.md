# YoHo (com.voicechat.live.group) v5.47.00 - Kapsamli Backend Akis Analizi

## 1. Genel Bakis

**Uygulama**: YoHo - Sesli Sohbet & Canli Yayin Platformu
**Paket**: `com.voicechat.live.group`
**Versiyon**: 5.47.00 (Build 5470000)
**Format**: XAPK (Split APK: base + config.armeabi_v7a + config.mdpi)
**Base APK Boyutu**: ~120MB
**Min SDK**: 24 (Android 7.0)
**Target SDK**: 36 (Android 16)
**Dil**: Kotlin + Java (Kotlin agirlikli)
**DI Framework**: Hilt (Dagger)

---

## 2. NEDEN BU KADAR OPTIMIZE? - Temel Mimari Kararlari

### 2.1. gRPC + Protobuf (REST Degil!)

Bu uygulamanin en kritik optimizasyon karari: **REST API yerine gRPC kullanmasi**.

**Normal bir uygulama:**
```
Client -> JSON serialize -> HTTP/1.1 -> Server -> JSON deserialize
```

**YoHo'nun yaklasimi:**
```
Client -> Protobuf serialize (binary) -> HTTP/2 (multiplexed) -> Server -> Protobuf deserialize
```

**Neden daha hizli?**
- **Protobuf vs JSON**: Protobuf binary format, JSON'dan **3-10x daha kucuk**, serialize/deserialize **5-20x daha hizli**
- **HTTP/2 Multiplexing**: Tek TCP baglantisi uzerinden birden fazla istek paralel gider. HTTP/1.1'de her istek icin ayri baglanti veya siralama gerekir
- **Header Compression (HPACK)**: HTTP/2 header'lari sikistirir, tekrarlayan header'lar icin sadece index gonderir
- **3036 Protobuf sinifi** tanimli - tum API iletisimi binary formatta

### 2.2. Custom "Cake" Framework

Retrofit'e benzer ama gRPC uzerine kurulu, sirketlerin kendi gelistirdigi bir framework:

```
com.mico.cake.core.Cake          -> Ana framework sinifi (Retrofit gibi)
com.mico.cake.core.Request       -> Istek modeli
com.mico.cake.call.*             -> 50+ gRPC servis tanimlamasi
com.mico.cake.parser.*           -> Protobuf serialization
com.mico.cake.rpc.*              -> gRPC call adapterleri
```

**Cake.Builder konfigurasyonu:**
```kotlin
Cake.Builder()
    .channelBuilder(androidChannelBuilder)  // gRPC channel
    .addInterceptor(retryInterceptor)       // Retry chain
    .callTimeout(30_000)                    // 30s timeout
    .callDispatcher(coroutineDispatcher)    // Kotlin Coroutines
    .dns(customDns)                         // Custom DNS
    .eventListener(lifecycleListener)       // Baglanti izleme
    .build()
```

**Onemli detay:** `ConcurrentHashMap` kullaniyor `serviceMethodCache`, `responseParsers`, `requestParsers`, `callMap` icin - thread-safe ve lock-free okuma sagliyor.

### 2.3. Interceptor Chain (OkHttp benzeri)

```
Request
  |-> MockInterceptor        (dev/test mock)
  |-> RetryInterceptor       (exponential backoff retry)
  |-> LegacyInterceptor      (eski API uyumlulugu)
  |-> ContextAwareInterceptor (context bilgisi ekleme)
  |-> RealCallInterceptor    (gercek gRPC cagrisini yapar)
```

`RealInterceptorChain` sinifi OkHttp'nin interceptor pattern'ini birebir taklit ediyor. Her interceptor, chain uzerinden bir sonrakine gecis yapiyor. Bu sayede her katman bagimsiz ve test edilebilir.

---

## 3. Agresif DNS Optimizasyonu

### 3.1. DNS over HTTPS (DoH)

Normal DNS: ISP'nin DNS sunucusu uzerinden, sifrelenmemis, yavastir ve manipule edilebilir.

YoHo'nun yaklasimi: **4 farkli DoH sunucusu** ile paralel cozumleme:

```kotlin
// DohList.kt - Varsayilan DoH sunuculari
listOf(
    "https://dns.google/dns-query",      // Google DoH
    "https://doh.pub/dns-query"          // Tencent DoH (Cin icin)
)

// OkHttpDohDnsResolver - Bootstrap IP'leri (DNS dongusu kirmak icin)
mapOf(
    "cloudflare-dns.com" to listOf("1.1.1.1", "162.159.36.1"),
    "dns.google"         to listOf("8.8.8.8", "8.8.4.4"),
    "dns.alidns.com"     to listOf("223.5.5.5", "223.6.6.6"),
    "doh.pub"            to listOf("120.53.53.53", "1.12.12.12")
)
```

**Neden onemli?**
1. **Hiz**: DoH, geleneksel DNS'den daha hizli olabilir (ozellikle yavas ISP DNS'lerinde)
2. **Guvenlik**: DNS sorgulari sifrelenir, ISP veya aradaki adam (MITM) goremez
3. **Sansur Atlatma**: Cin'de DNS kirliligini atlatir (Tencent DoH eklenmis)
4. **Paralel Cozumleme**: Birden fazla DoH sunucusuna paralel sorgu gonderir, en hizli cevabi kullanir

### 3.2. Custom DNS Name Resolver

```kotlin
// DnsNameResolver.kt
class DnsNameResolver(name, args, dns: Dns) {
    val port = 443  // Varsayilan port - HTTPS/gRPC
    
    fun resolve() {
        if (shutdown || !resolving.compareAndSet(false, true)) return
        // AtomicBoolean ile thread-safe resolution
        // Executor uzerinden asenkron DNS cozumleme
        executor.execute(Resolve())  // Arka planda cozumle
    }
}
```

DNS resolver **AtomicBoolean** ve **AtomicReference** kullaniyor - lock-free, non-blocking DNS resolution.

---

## 4. Baglanti Yonetimi ve Izleme

### 4.1. GrpcLifecycleListener - Tam Baglanti Gozetimi

Uygulama her baglanti asamasini izliyor:

```
NetworkEventListener:
  - onDns(DnsEvent)                    -> DNS cozumleme suresi
  - onTcpConnect(TcpConnectEvent)      -> TCP baglanti suresi
  - onTlsHandshake(TlsHandshakeEvent)  -> TLS el sikisma suresi
  - onChannelStateChange(ChannelStateChangeEvent) -> Kanal durumu

StreamEventListener:
  - onCallLifecycle(CallLifecycleEvent)     -> API cagri yasam dongusu
  - onStreamLifecycle(StreamLifecycleEvent) -> Stream baslama/bitime
  - onRetry(RetryEvent)                     -> Retry olaylari

MessageEventListener:
  - onInbound(InboundEvent)   -> Gelen mesaj boyutu/suresi
  - onOutbound(OutboundEvent) -> Giden mesaj boyutu/suresi
```

### 4.2. ChannelStateMonitor - Baglanti Durumu Izleme

```kotlin
class ChannelStateMonitor(channel, context) {
    val running = AtomicBoolean(false)
    val transientFailureCount = AtomicInteger(0)
    
    // 5 farkli durum izleniyor:
    IDLE        -> Baglanti bosta
    CONNECTING  -> Baglaniliyor (sure olcumu baslar)
    READY       -> Baglanti hazir (sure olcumu biter, rapor edilir)
    TRANSIENT_FAILURE -> Gecici hata (sayac artar, otomatik yeniden baglanma)
    SHUTDOWN    -> Kanal kapandi
}
```

**TRANSIENT_FAILURE durumunda:**
- `transientFailureCount` artirilir
- `inferLastConnectionFailureIfNeeded()` ile hata nedeni cikarilir
- Otomatik yeniden baglanma mekanizmasi devreye girer

**CONNECTING -> READY gecisinde:**
- Baglanti suresi hesaplanir: `System.currentTimeMillis() - connectingStartTime`
- Bu sure performans izleme sistemine raporlanir

### 4.3. Load Balancing

```kotlin
enum class LoadBalancingPolicy {
    PICK_FIRST,   // Ilk uygun sunucuya baglan
    ROUND_ROBIN,  // Sunucular arasinda sirayla dagit
    UNKNOWN
}
```

gRPC'nin dahili load balancing destegi kullaniliyor. `ROUND_ROBIN` ile birden fazla backend sunucusu arasinda yuk dagitimi yapiliyor.

---

## 5. Retry Mekanizmasi - Akilli Yeniden Deneme

### 5.1. ExponentialBackoff

```kotlin
class ExponentialBackoff(
    initialDelayMillis: Long = 500,     // Ilk bekleme: 500ms
    multiplier: Double = 2.0,            // Her denemede 2x artis
    maxDelayMillis: Long = 1000,         // Maksimum bekleme: 1s (10s'ye kadar olabilir)
    jitterFactor: Double = 0.1           // %10 rastgele sapma
)

// Bekleme suresi hesaplama:
// delay = initialDelay * (multiplier ^ (attempt - 1))
// delay = min(delay, maxDelay)
// delay = delay + (delay * jitterFactor * random())

// Ornek: 500ms -> 1000ms -> 2000ms -> 4000ms (+ %10 jitter)
```

**Jitter neden onemli?** Binlerce istemci ayni anda hata alirsa, hepsi ayni anda retry yapmaya calisir ("thundering herd"). Jitter ekleyerek retry'lari zamana yayar.

**Guvenlik sinirlari:**
- `initialDelayMillis`: 1ms - 10000ms arasi
- `multiplier`: 1.0 - 5.0 arasi
- `maxDelayMillis`: initialDelay'den buyuk, max 10000ms
- `jitterFactor`: 0.0 - 1.0 arasi
- Tum hata mesajlari Cince (Cin gelistirme ekibi)

### 5.2. RetryInterceptor

```kotlin
class RetryInterceptor : ContextAwareInterceptor {
    // RetryPolicy: maxAttempts + RetryPredicate
    // RetryPredicate: Hangi hatalarda retry yapilacagini belirler
    
    // Her retry denemesinde:
    shouldRetry(error)      -> Retry yapilmali mi?
    logRetryAttempt(...)    -> "重试" (retry) loglama
    logRetryResult(...)     -> Sonuc loglama
    logRetryFailure(...)    -> Basarisizlik loglama
}
```

---

## 6. Gercek Zamanli Iletisim Katmani

### 6.1. Zego RTC SDK (Canli Yayin & Sesli Sohbet)

```
com.zego.*      -> Zego Express SDK (ana RTC motoru)
im.zego.*       -> Zego IM SDK (anlik mesajlasma)
```

- **WebRTC tabanli** ses/video streaming
- **Dusuk gecikme**: P2P baglanti mumkun oldugunda direkt, degilse Zego'nun relay sunuculari
- **Adaptif bit rate**: Ag durumuna gore otomatik kalite ayarlama
- Oda yonetimi, mikrofon/hoparlor kontrolu, efekt isleme

### 6.2. MQTT over Netty (Mesajlasma)

```
io.netty.*                    -> Netty NIO framework
io.netty.handler.codec.mqtt.* -> MQTT protokol codec
```

**MQTT neden kullaniliyor?**
- **Dusuk overhead**: Minimum 2 byte header (HTTP'nin ~800 byte'ina karsi)
- **Pub/Sub modeli**: Oda mesajlari icin mukemmel (bir yayinci, cok dinleyici)
- **QoS seviyeleri**: 0 (at most once), 1 (at least once), 2 (exactly once)
- **Keep-alive**: Baglanti canli tutma, kopma aninda aninda haberdar olma
- **Netty uzerinde**: Non-blocking I/O, yuksek performansli event loop

### 6.3. Firebase Cloud Messaging (Push Notification)

```xml
<!-- AndroidManifest.xml -->
<service android:name="com.audionew.common.fcm.FcmService">
    <intent-filter>
        <action android:name="com.google.firebase.MESSAGING_EVENT"/>
    </intent-filter>
</service>
```

---

## 7. Veri Depolama ve Onbellekleme

### 7.1. MMKV (Tencent)

```
com.tencent.mmkv.* -> Mmap tabanli key-value store
```

**Neden SharedPreferences degil?**
- **MMKV**: Memory-mapped file, ~100x daha hizli yazma
- **SharedPreferences**: XML serialize/deserialize, her yazmada tum dosyayi yazar
- **MMKV**: Multi-process destegi, crash-safe
- Kullanici tercihleri, session tokenlari, cache verileri icin

### 7.2. SQLite (Room/Framework DB)

```
com.mico.framework.datastore.db.* -> Veritabani katmani
```

- Kullanici bilgileri, sohbet gecmisi, oda verileri
- `MeService` - Kullanici oturumu yonetimi

### 7.3. Glide (Gorsel Onbellekleme)

```
com.bumptech.glide.* -> Gorsel yukleme ve onbellekleme
```

- Bellek + disk cache
- Gorsel boyutlandirma ve donusum
- Placeholder/error handling

---

## 8. API Endpoint ve Servis Haritasi

### 8.1. gRPC Servisleri (50+ servis)

| Servis | Islem Alani |
|--------|-------------|
| `ApiChatService` | Mesajlasma |
| `ApiRoomMgrService` | Oda yonetimi |
| `ApiPayCenterService` | Odeme merkezi |
| `ApiGiftListService` | Hediye listesi |
| `ApiUserInfoService` | Kullanici bilgileri |
| `ApiVideoRoomService` | Video oda |
| `ApiGameCenterService` | Oyun merkezi |
| `ApiFamilyService` | Aile (klan) sistemi |
| `ApiAudioBroadcastService` | Sesli yayin |
| `ApiWalletService` | Cuzdan islemleri |
| `ApiLevelService` | Seviye sistemi |
| `ApiMatchService` | Eslestirme |
| `ApiMomentService` | Sosyal akis |
| `ApiBannerService` | Banner/reklam |
| `ApiConfigService` | Uzak konfigurasyon |
| `ApiReportService` | Raporlama |
| `ApiSearchService` | Arama |
| `ApiRelationService` | Iliski (takip/engel) |
| `ApiNotificationService` | Bildirimler |

### 8.2. Backend Domainler

| Domain | Kullanim |
|--------|----------|
| `config.micoplatform.com` | Konfigurasyon sunucusu |
| `cdp-api.micoplatform.com` | CDP (Customer Data Platform) API |
| `cdn.hoho.media` | CDN (medya dosyalari) |
| `www.yoho.media` | Web sitesi |
| `www.waka.media` | Alternatif domain |
| `m.hoho.media` | Mobil web |

### 8.3. Deep Link Yapisi

```
yoho.onelink.me       -> AppsFlyer OneLink (attribution)
waka.media             -> Web deep link
waka.page.link         -> Firebase Dynamic Link
www.yoho.media         -> Web deep link
```

---

## 9. Ucuncu Parti SDK Envanteri

| SDK | Amac | Optimizasyon Etkisi |
|-----|-------|---------------------|
| **Zego RTC** | Canli yayin/sesli sohbet | Dusuk gecikmeli medya aktarimi |
| **Netty + MQTT** | Gercek zamanli mesajlasma | Non-blocking I/O, minimal overhead |
| **Protobuf** | Veri serializasyonu | 3-10x kucuk, 5-20x hizli |
| **gRPC/OkHttp** | API iletisimi | HTTP/2 multiplexing |
| **MMKV** | Key-value depolama | mmap ile 100x hizli yazma |
| **Glide** | Gorsel yukleme | Akilli cache yonetimi |
| **Firebase** | Analytics, push, remote config, crashlytics | - |
| **AppsFlyer** | Attribution/tracking | - |
| **Alibaba SDK** | Cin pazari entegrasyonu | - |
| **Sobot Chat** | Musteri hizmetleri | - |
| **Google Billing** | In-app satin alma | - |
| **Huawei HMS** | Huawei cihaz destegi | - |
| **Facebook SDK** | Sosyal giris, analytics | - |
| **Adjust** | Attribution | - |
| **OOM Manager** | Bellek yonetimi | OOM onleme |

---

## 10. Ag Guvenlik Konfigurasyonu

```xml
<!-- network_security_config.xml -->
<network-security-config>
    <base-config cleartextTrafficPermitted="true" />
</network-security-config>
```

**Not:** `cleartextTrafficPermitted="true"` - HTTP (sifrelenmemis) trafige izin veriliyor. Bu bir guvenlik zafiyeti olabilir, ancak muhtemelen belirli CDN veya yerel servisler icin gerekli.

---

## 11. Optimizasyon Teknikleri Ozeti

### Katman 1: Ag Protokolu
| Teknik | Etki |
|--------|------|
| gRPC (HTTP/2) | Multiplexing, header compression, bidirectional streaming |
| Protobuf | 3-10x kucuk payload, 5-20x hizli serialization |
| DoH (4 sunucu) | Hizli + guvenli DNS, paralel cozumleme |
| Custom DNS Resolver | Lock-free, asenkron, bootstrap IP'ler ile DNS dongusu kirmak |

### Katman 2: Baglanti Yonetimi
| Teknik | Etki |
|--------|------|
| Connection pooling (gRPC channel) | Baglanti yeniden kullanimi |
| Load balancing (PICK_FIRST/ROUND_ROBIN) | Sunucu yuku dagitimi |
| ChannelStateMonitor | Baglanti durumu izleme, otomatik yeniden baglanma |
| Exponential backoff + jitter | Akilli retry, thundering herd onleme |

### Katman 3: Veri Katmani
| Teknik | Etki |
|--------|------|
| MMKV | 100x hizli KV yazma, crash-safe |
| ConcurrentHashMap cache | Lock-free service method cache |
| Glide disk/memory cache | Gorsel onbellekleme |

### Katman 4: Asenkron Islem
| Teknik | Etki |
|--------|------|
| Kotlin Coroutines | Hafif asenkron islem, structured concurrency |
| Netty NIO | Non-blocking I/O, event loop tabanli ag islemleri |
| AtomicBoolean/AtomicReference | Lock-free thread safety |

### Katman 5: Gercek Zamanli
| Teknik | Etki |
|--------|------|
| MQTT (2 byte min header) | Ultra-dusuk overhead mesajlasma |
| WebRTC (Zego) | P2P ses/video, adaptif bitrate |
| gRPC bidirectional streaming | Server-push yetenegi |

---

## 12. Mimari Diyagram

```
+------------------------------------------------------------------+
|                        YoHo Application                           |
+------------------------------------------------------------------+
|  MimiApplication (Hilt DI)                                       |
|  +------------------------------------------------------------+  |
|  |  LaunchTask -> Startup Optimization                         |  |
|  |  ActivityLifecycleMonitor -> Performans Izleme             |  |
|  +------------------------------------------------------------+  |
+------------------------------------------------------------------+
|                     Feature Layer                                 |
|  +------------+ +----------+ +----------+ +-----------+         |
|  | Audio Room | | Chat     | | Payment  | | Social    |         |
|  | (Zego RTC) | | (MQTT)   | | (Billing)| | (Moments) |         |
|  +------------+ +----------+ +----------+ +-----------+         |
+------------------------------------------------------------------+
|                     Cake Framework (gRPC)                         |
|  +------------------------------------------------------------+  |
|  | Service Interface (50+ APIs)                                |  |
|  |   -> Java Dynamic Proxy                                     |  |
|  |   -> RequestFactory -> GrpcServiceMethod                    |  |
|  +------------------------------------------------------------+  |
|  | Interceptor Chain                                           |  |
|  |   MockInterceptor -> RetryInterceptor -> RealCallInterceptor|  |
|  +------------------------------------------------------------+  |
|  | Protobuf Parser Layer (3036 message classes)                |  |
|  |   ProtobufRequestParser <-> ProtobufResponseParser          |  |
|  +------------------------------------------------------------+  |
+------------------------------------------------------------------+
|                     Transport Layer                               |
|  +------------------------------------------------------------+  |
|  | AndroidChannelBuilder (OkHttp + gRPC)                       |  |
|  |   - Custom SocketFactory (EventListener)                    |  |
|  |   - Custom SSLSocketFactory (TLS monitoring)                |  |
|  |   - Network change BroadcastReceiver                        |  |
|  +------------------------------------------------------------+  |
|  | DNS Layer                                                   |  |
|  |   DnsNameResolver -> OkHttpDohDnsResolver                   |  |
|  |   (Google DoH | Tencent DoH | Cloudflare | AliDNS)         |  |
|  +------------------------------------------------------------+  |
|  | Connection Monitoring                                       |  |
|  |   GrpcLifecycleListener + ChannelStateMonitor               |  |
|  |   (DNS/TCP/TLS/Stream/Message events)                       |  |
|  +------------------------------------------------------------+  |
+------------------------------------------------------------------+
|                     Storage Layer                                 |
|  +------------+ +-----------+ +------------+                    |
|  | MMKV       | | SQLite/   | | Glide      |                    |
|  | (KV Store) | | Room (DB) | | (Image $)  |                    |
|  +------------+ +-----------+ +------------+                    |
+------------------------------------------------------------------+
```

---

## 13. Sonuc

YoHo'nun optimizasyonu tek bir "sihirli" teknik degil, **katmanli ve sistematik** bir yaklasimin sonucu:

1. **Protokol secimi** (gRPC + Protobuf): En buyuk performans kazanci burada. JSON/REST yerine binary protocol + HTTP/2 kullanmak, her API cagrisinda hem bant genisligi hem de CPU tasarrufu sagliyor.

2. **DNS katmani** (DoH + paralel cozumleme + bootstrap IP'ler): Ilk baglantida bile DNS gecikmesini minimize ediyor. Cin gibi DNS manipulasyonunun yaygin oldugu pazarlarda kritik.

3. **Baglanti dayanikliligi** (exponential backoff + jitter + state monitoring): Ag sorunlarinda akilli davranis, thundering herd onleme, otomatik kurtarma.

4. **Lock-free veri yapilari** (AtomicBoolean, AtomicInteger, ConcurrentHashMap): Thread contention minimizasyonu, ozellikle yogun concurrent erisimde.

5. **Mesajlasma protokolu** (MQTT over Netty): Chat mesajlari icin HTTP yerine MQTT kullanmak, header overhead'i 800+ byte'dan 2 byte'a dusuruyor.

6. **Yerel depolama** (MMKV): SharedPreferences'a gore ~100x hizli, ozellikle uygulama baslangicinda fark yaratiyor.

Bu uygulamayi "normal" bir REST API + JSON + SharedPreferences uygulamasiyla karsilastirdigimizda, tahmini performans farki:
- **API latency**: 2-5x daha dusuk
- **Bant genisligi**: 3-10x daha az
- **Serialization CPU**: 5-20x daha az
- **DNS cozumleme**: 2-3x daha hizli
- **KV store yazma**: ~100x daha hizli
- **Mesajlasma overhead**: ~400x daha az header

---

*Bu analiz jadx ile decompile edilen kaynak kodundan, AndroidManifest.xml'den ve ag konfigurasyonlarindan elde edilmistir.*

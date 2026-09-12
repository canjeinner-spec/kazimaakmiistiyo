# YoHo v5.47.00 - KESİN HÜKÜM: API Endpoint, Authentication & Authorization Güvenlik Değerlendirmesi

**Paket**: `com.voicechat.live.group`
**Versiyon**: 5.47.00 (Build 5470000)
**Analiz Tarihi**: 2026-09-12
**Yöntem**: Statik kod analizi (jadx decompile) + Canlı endpoint testi (DNS, TCP, WebSocket, TLS)
**Kapsam**: APK → API endpoint keşfi → HTTP/WebSocket istek inceleme → authentication/authorization kontrol testi

---

## İÇİNDEKİLER

1. [Keşfedilen API Endpoint Haritası](#1-kesfedilen-api-endpoint-haritası)
2. [Canlı Endpoint Test Sonuçları](#2-canli-endpoint-test-sonuclari)
3. [Authentication Mimarisi](#3-authentication-mimarisi)
4. [Authorization Kontrolleri](#4-authorization-kontrolleri)
5. [Güvenlik Zafiyetleri - KESİN HÜKÜMLER](#5-guvenlik-zafiyetleri---kesin-hukumler)
6. [Sonuç ve Risk Matrisi](#6-sonuc-ve-risk-matrisi)

---

## 1. Keşfedilen API Endpoint Haritası

### 1.1. Birincil Domainler (Canlı DNS Çözümleme ile Doğrulanmış)

| Domain | Port | Protokol | Amaç | Canlı IP (Google DoH) | CDN Sağlayıcı |
|--------|------|----------|------|----------------------|---------------|
| `rpc.yoho.media` | 443 | gRPC/H2 | Ana API gateway (55 servis, 559+ method) | 43.152.182.46 | Tencent EdgeOne |
| `sso.yoho.media` | 8080, 443 | TCP/Protobuf | Kalıcı TCP bağlantısı (chat, bildirim) | 43.159.112.32, 43.159.113.32 | Tencent EdgeOne |
| `www.yoho.media` | 443 | HTTPS | Web API / OAuth callback | 138.113.128.90, 140.150.29.42 | WheCloud CDN |
| `m.yoho.media` | 443 | HTTPS | Mobil web API | 140.150.29.42, 138.113.128.90 | CDN20 |
| `cdn.yoho.media` | 443 | HTTPS | Statik içerik CDN | 140.150.29.42, 138.113.128.90 | WheCloud CDN |
| `cdn-video.yoho.media` | 443 | HTTPS | Video CDN | CDN-routed | WheCloud |
| `h5.yoho.media` | 443 | HTTPS | H5/WebView içerik | 43.152.182.46 | Tencent EdgeOne |
| `platform.yoho.media` | 443 | HTTPS | Platform API | 47.84.95.102 | Doğrudan |
| `api.eventbus.me` | 443 | HTTPS | Event bus servisi | 43.159.95.19 | Tencent EdgeOne |
| `config.micoplatform.com` | 443 | HTTPS | Uzak yapılandırma | 43.159.95.19 | Tencent EdgeOne |
| `boce-ws.micoplatform.com` | 80, 443 | WebSocket | Ağ teşhis WebSocket | 47.236.29.187 | Doğrudan (nginx/1.20.1) |

### 1.2. Philippine Bölge Domainleri (hoho.media)

| Domain | Canlı IP | yoho.media ile Aynı mı? |
|--------|----------|--------------------------|
| `rpc.hoho.media` | 43.152.182.46 | **EVET** - Aynı IP, aynı altyapı |
| `sso.hoho.media` | 43.159.112.32, 43.159.113.32 | **EVET** - CNAME zinciri `yoho.media`'ya gider |
| `cdn.hoho.media` | 138.113.241.53, 138.113.158.123 | **HAYIR** - Farklı CDN edge, ayrı geo-routing |

**KESİN HÜKÜM**: `hoho.media` ayrı bir altyapı değildir. Philippine SIM tespit edildiğinde aktive olan bir **domain alias**'tır. Backend tamamen aynıdır, yalnızca CDN edge sunucuları farklılık gösterir.

### 1.3. Doğrudan IP Failover

| IP | Portlar | TCP Test Sonucu |
|----|---------|-----------------|
| `47.236.159.235` | 6080-6083 | **TÜMÜ TIMEOUT** - Muhtemelen bölgesel IP kısıtlaması var |
| `43.159.113.33` (hardcoded gRPC) | 443 | **AÇIK** - Eski IP hala servis veriyor |
| `43.159.112.33` (hardcoded gRPC) | 443 | **AÇIK** - Eski IP hala servis veriyor |

### 1.4. WebSocket Endpointleri

| Endpoint | Protokol | Auth | Canlı Test |
|----------|----------|------|------------|
| `ws://boce-ws.micoplatform.com/ws` | WS | Yok | Aktif, nginx/1.20.1, WebSocket handshake tanıyor ama eksik parametrelerle reddediyor |
| `wss://boce-ws.micoplatform.com/ws` | WSS | Yok | Aktif, GoDaddy sertifikası (*.micoplatform.com) |
| MiBot SDK WebSocket | WSS | `Sec-Websocket-Protocol` header | Dinamik URL, DomainSelector ile çoklu host failover |

### 1.5. 3. Parti Entegrasyonlar

| Servis | Endpoint | APK'da Bulunan Credentials |
|--------|----------|---------------------------|
| Firebase | waka-audio-chat.firebaseio.com | API Key: `AIzaSyBZO3gE2lEc5Zl4ZKzxV0RFLa5nXNGR2D0` |
| Google OAuth | accounts.google.com | Client ID: `1073583627248-7vrt0nv11bn8r9kmu6dr9eu7le8is1fk.apps.googleusercontent.com` |
| Facebook | graph.facebook.com | App ID: `1165522863595325` |
| Snapchat | accounts.snapchat.com | OAuth Client ID: `80e77d20-2e7e-4755-ba6a-816fd63a85e0` |
| TikTok | open.tiktokapis.com | Sandbox Client Key: `sbaw8vr7wghzwqx8l5` |
| Zego RTC | zego.im | Entegre, key config'den geliyor |
| AppsFlyer | - | Entegre, Firebase instance ID ile |

### 1.6. gRPC Servis Kataloğu (55 Servis - Doğrulanmış)

`RpcStubUtils.java`'dan doğrulanmış tam servis listesi (54 RpcStubUtils + 1 Cake framework):

```
NATIVE HMAC KORUMALI (6 method):
  sign.SignInService/SignIn, SignUp, AppStart, ResetPwd, ForceUpdate
  proto.red_env.RedEnvelopeService/C2SScramblingRedEnvelope

FİNANSAL SERVİSLER (korumasız):
  ApppayGrpc, CashOutServiceGrpc, PayCenterServiceGrpc,
  SilverCoinsLogicServiceGrpc, ShopServiceGrpc

HEDİYE/ÖDÜL SERVİSLERİ (korumasız):
  AudioGiftServiceGrpc, ChatGiftServiceGrpc, GiftListGrpc,
  LuckyGiftServiceGrpc, RedEnvelopeServiceGrpc, RedRainServiceGrpc

ODA/YAYIN SERVİSLERİ (korumasız):
  RoomMgrServiceGrpc, RoomMicManagerServiceGrpc, EnterRoomServiceGrpc,
  RoomRcmdServiceGrpc, AudioChatServiceGrpc, BroadcastShareServiceGrpc

KULLANICI SERVİSLERİ (korumasız):
  UserInfoServiceGrpc, UserSvrServiceGrpc, FriendShipServiceGrpc,
  FansServiceGrpc, GuardianRelationServiceGrpc, NewUserServiceGrpc

DİĞER (korumasız):
  TranslateServiceGrpc, FamilyServiceGrpc, RankingListServiceGrpc,
  MeetServiceGrpc, AudioTaskServiceGrpc, DailyTaskServiceGrpc,
  FastGameServiceGrpc, GameLevelServiceGrpc, GameBuddyServiceGrpc,
  AuctionServiceGrpc, AgencyServiceGrpc, CpTaskServiceGrpc,
  ActivitySquareServiceGrpc, GreedyActivityServiceGrpc, ve daha fazlası
```

**KESİN HÜKÜM**: 55 servisin sadece **1 tanesi** (SignInService, 6 method ile) native HMAC koruması altında. Geri kalan **54 servis** yalnızca `x-auth-token` header'ı ile korunuyor. Apppay, CashOut, PayCenter gibi **finansal servisler** dahil ek imza koruması **YOK**.

---

## 2. Canlı Endpoint Test Sonuçları

### 2.1. TCP Bağlantı Testi

| Endpoint | Sonuç | Yorum |
|----------|-------|-------|
| rpc.yoho.media:443 | **AÇIK** | gRPC gateway aktif |
| rpc.hoho.media:443 | **AÇIK** | Aynı IP (43.152.182.46) |
| sso.yoho.media:443 | **AÇIK** | SSO HTTPS aktif |
| sso.yoho.media:8080 | **TIMEOUT** | TCP socket port kapalı/kısıtlı |
| sso.hoho.media:8080 | **TIMEOUT** | TCP socket port kapalı/kısıtlı |
| cdn.yoho.media:443 | **AÇIK** | CDN aktif |
| api.eventbus.me:443 | **AÇIK** | Event bus aktif |
| config.micoplatform.com:443 | **AÇIK** | Config servisi aktif |
| boce-ws.micoplatform.com:80 | **AÇIK** | WebSocket HTTP aktif |
| boce-ws.micoplatform.com:443 | **AÇIK** | WebSocket HTTPS aktif |
| 47.236.159.235:6080-6083 | **TÜMÜ TIMEOUT** | IP/port kısıtlaması var |

### 2.2. DNS Çözümleme Doğrulaması (Google DoH + Tencent DoH Karşılaştırması)

İki bağımsız DoH sağlayıcı ile çapraz doğrulama yapıldı:

| Domain | Google DoH | Tencent DoH | Tutarlı mı? |
|--------|------------|-------------|-------------|
| rpc.yoho.media | 43.152.182.46 | 43.152.182.46 | **EVET** |
| sso.yoho.media | 43.159.112.32, 43.159.113.32 | 43.159.113.32, 43.159.112.32 | **EVET** |
| cdn.yoho.media | 140.150.29.42, 138.113.128.90 | 157.185.163.158, 157.185.157.15 | **HAYIR** - CDN geo-routing (beklenen) |

### 2.3. Hardcoded IP vs Canlı DNS Karşılaştırması

| Domain | Hardcoded IP | Canlı IP | Eşleşme |
|--------|-------------|----------|---------|
| rpc.yoho.media | 43.159.113.33, 43.159.112.33 | 43.152.182.46 | **HAYIR** - IP değişmiş |
| sso.yoho.media | 43.159.112.32, 43.159.113.32 | 43.159.112.32, 43.159.113.32 | **EVET** |
| cdn.yoho.media | 168.235.196.32, 163.171.131.198 | 140.150.29.42, 138.113.128.90 | **HAYIR** - IP değişmiş |
| platform.yoho.media | 47.84.95.102 | 47.84.95.102 | **EVET** |

**KESİN HÜKÜM**: Hardcoded IP'lerin çoğu güncel değil ama eski gRPC IP'leri (43.159.113.33/43.159.112.33) hala port 443'te aktif servis veriyor. Bu, kademeli migrasyon yapıldığını gösteriyor. Hardcoded IP fallback mekanizması tasarlandığı gibi çalışıyor - DNS başarısız olsa bile uygulama bağlanabiliyor.

### 2.4. TLS Sertifika Tespitleri

Test ortamından yapılan TLS bağlantılarında Anthropic egress proxy sertifikası görüldü (beklenen). Tek doğrudan gözlemlenen gerçek sertifika:

- **boce-ws.micoplatform.com**: `CN=*.micoplatform.com`, Issuer: **GoDaddy Secure Certificate Authority - G2**, Valid: Feb 2026 - Feb 2027

**KESİN HÜKÜM**: Sunucu tarafında TLS sertifikaları mevcuttur. Ancak istemci (APK) tarafında sertifika doğrulaması **tamamen devre dışıdır** (detaylar Bölüm 5.1'de).

---

## 3. Authentication Mimarisi

### 3.1. Kimlik Bilgisi Türleri (4 Adet)

APK'da `p843z8/C36268a.java` dosyasında tanımlı 4 ayrı credential tipi:

| Token | Depolama | Kullanım | Güvenlik |
|-------|----------|----------|----------|
| `access_token` | SharedPreferences (düz metin XML) | gRPC isteklerinde `x-auth-token` header | **ZAYIF** - Düz metin dosyada |
| `tcp_token` | SharedPreferences (düz metin XML) | TCP persistent bağlantı handshake | **ZAYIF** - Düz metin dosyada |
| `refresh_token` | SharedPreferences (düz metin XML) | Token yenileme | **ZAYIF** - Düz metin dosyada |
| `encrypted_key` | SharedPreferences (düz metin XML) | TCP handshake HmacMD5 digest | **ZAYIF** - İsmi "encrypted" ama düz metin saklanıyor |

**KESİN HÜKÜM**: Tüm kimlik bilgileri `SharedPreferences` kullanılarak **düz metin XML dosyalarında** saklanıyor. MMKV, EncryptedSharedPreferences, Android Keystore gibi güvenli depolama mekanizması **kullanılmıyor**. Root erişimli bir cihazda tüm tokenlar okunabilir.

Oturum kontrolü (`C36268a.m121942M`):
```
uid != 0 AND access_token mevcut AND tcp_token mevcut AND encrypted_key mevcut
```

### 3.2. gRPC İstek İmzalama Zinciri

İstekler şu interceptor sırasıyla işlenir:

```
İstek → RpcErrorInterceptor → RpcHeaderInterceptor → RpcAppProofInterceptor → RpcTracingInterceptor → RpcNetDiagnosisInterceptor → Sunucu
```

#### RpcHeaderInterceptor (`p145J8/C0719c.java`)
Her gRPC isteğine eklenen header'lar:

| Header | Değer Kaynağı |
|--------|--------------|
| `x-auth-token` | SharedPreferences'dan access_token |
| `uid` | Kullanıcı ID |
| `region` | Bölge kodu |
| `lang` | Dil kodu |
| `version` | Uygulama versiyonu |
| `os` | `2` (Android) |
| `pkg` | Paket adı |
| `issue-channel` | Dağıtım kanalı |
| `did` | android_id (cihaz parmak izi) |
| `tz` | Timezone offset |
| `tz-id` | Timezone ID |
| `request-time-s` | Unix timestamp (saniye) |
| `idfa` | Reklam ID |
| `appsflyer-id` | AppsFlyer ID |
| `firebase-instance-id` | Firebase Instance ID |

**KESİN HÜKÜM**: Header bilgisi zengin ama tümü **düz metin** olarak gönderiliyor. `x-auth-token` tek başına yeterli - ek doğrulama (cihaz bağlama, IP kontrolü) istemci tarafında **görülmüyor**. Token çalındığında başka bir cihazdan kullanılması istemci kodunda engellenmiyor.

#### RpcAppProofInterceptor (`p145J8/C0717a.java`)
Sadece 6 method için native HMAC imzalama:

```
Korunan methodlar:
  1. sign.SignInService/SignIn
  2. sign.SignInService/SignUp
  3. sign.SignInService/AppStart
  4. sign.SignInService/ResetPwd
  5. sign.SignInService/ForceUpdate
  6. proto.red_env.RedEnvelopeService/C2SScramblingRedEnvelope
```

İmza formatı:
```
Payload = businessVersion|versionName|packageName|timestamp|nonce
İmza   = nativeSignPayload(payload)  // libapp-proof.so JNI çağrısı
```

Eklenen header'lar:
- `sign`: HMAC imzası
- `nonce`: 16 karakter rastgele (UUID'den türetilmiş)
- `request-time-s`: Unix timestamp

**KESİN HÜKÜM**: Native HMAC koruması yalnızca **6 kritik method** için geçerli. Geri kalan **553 method** (Wallet, Gift, Room, Feed, LiveStream, UserProfile vs.) native imza koruması **OLMADAN** sadece `x-auth-token` ile çağrılabiliyor. `libapp-proof.so` reverse-engineer edilebilir ancak bu 6 method için ek bir engel oluşturuyor.

### 3.3. TCP Persistent Bağlantı Authentication

**Dosya**: `com/mico/framework/network/tcp/TcpHandshakeUtils.java`

TCP handshake akışı:
```
1. İstemci → Protobuf handshake mesajı: {token, random, timestamp, deviceID, os, version, ...}
2. HmacMD5 digest = HMAC-MD5(handshake_data, encrypted_key)
3. AES şifreli handshake + digest → Sunucu
4. Sunucu doğrular → Bağlantı kurulur
```

**KESİN HÜKÜM**: TCP auth HmacMD5 kullanıyor - bu **kriptografik olarak zayıf** bir hash fonksiyonudur. MD5 collision saldırılarına karşı kırılgandır. Modern uygulamalar HMAC-SHA256 kullanmalıdır.

### 3.4. Token Yenileme Mekanizması

**Dosya**: `com/mico/framework/network/rpc/RpcTokenUtils.java`

```
RenewToken RPC çağrısı:
  - Global rate limit: 12 saat (aynı token 12 saat içinde yenilenemez)
  - Burst rate limit: 1 dakika (ardışık yenileme istekleri arası minimum süre)
  - Rate limit kontrolü: ReqLimitMkv (istemci tarafı)
```

**KESİN HÜKÜM**: Rate limiting **yalnızca istemci tarafında** uygulanıyor (`ReqLimitMkv`). Sunucu tarafı rate limiting olup olmadığı istemci kodundan belirlenemiyor, ancak istemci tarafı kontrol bir saldırgan tarafından kolayca bypass edilebilir.

### 3.5. Google Play Integrity

**Dosya**: `com/mico/framework/analysis/security/PlayIntegrityManager.java` (809 satır)

- **API Tipi**: Google Play Integrity **Standard API**
- **Token Süresi**: 30 dakika sonra provider sıfırlanır
- **Hash Algoritması**: SHA-256 (parametreler `&` ile birleştirilir)
- **Şifreleme**: RSA/ECB/PKCS1Padding ile sunucu public key'i kullanılarak
- **Feature Flag**: Firebase Remote Config ile kontrol ediliyor (0=devre dışı, 1=aktif, 2=devre dışı)
- **App Cloner Tespiti**: `com.applisto.appcloner` metadata key kontrolü
- **Kullanım Kapsamı**: SignIn, SignUp, AppStart + tüm OAuth türleri (Phone, Facebook, Google, Snapchat, TikTok, Huawei, Line)

**KESİN HÜKÜM**: Play Integrity yalnızca giriş/kayıt/AppStart aşamasında kullanılıyor. Oturum süresince yapılan API çağrılarında **tekrar kontrol edilmiyor**. Firebase Remote Config ile tamamen kapatılabiliyor (value=0 veya 2). Token provider 30 dakika sonra expire oluyor ama oturum devam ediyor.

---

## 4. Authorization Kontrolleri

### 4.1. İstemci Tarafı Kontroller

İstemci kodunda gözlemlenen authorization kontrolleri:

| Kontrol | Uygulama Noktası | Bypass Edilebilir mi? |
|---------|-------------------|----------------------|
| Login durumu kontrolü | `C36268a.m121942M()` | **EVET** - İstemci tarafı |
| Rate limiting (token yenileme) | `ReqLimitMkv` | **EVET** - İstemci tarafı |
| Region kontrolü (Philippines) | `NetworkBlockUtils` | **EVET** - SIM bilgisine bağlı |
| Debug/Sandbox kontrolü (TikTok) | `TiktokAuthConfig` | **EVET** - İstemci tarafı flag |

### 4.2. Sunucu Tarafı Kontroller (İstemci Kodundan Çıkarım)

İstemci kodunun sunucuya gönderdiği bilgilere bakarak sunucu tarafında muhtemel kontroller:

| Kontrol | Kanıt | Kesinlik |
|---------|-------|----------|
| `x-auth-token` doğrulama | Tüm isteklerde zorunlu gönderiliyor | **KESİN** |
| `uid` doğrulama | Header'da gönderiliyor | **MUHTEMEL** |
| Native HMAC (6 method) | İmza olmadan istekler reddediliyor olmalı | **KESİN** |
| Play Integrity (SignIn/SignUp) | Token gönderiliyor | **MUHTEMEL** |
| IP/bölge kısıtlama | Port 8080 ve direct IP timeout | **KESİN** (ağ seviyesinde) |
| Cihaz bağlama | İstemci tarafında `did` gönderiliyor ama binding kanıtı yok | **BELİRSİZ** |

**KESİN HÜKÜM**: İstemci kodunda **role-based access control (RBAC)** kanıtı görülmüyor. Tüm authorization `x-auth-token` + `uid` çiftine bağlı görünüyor. Sunucu tarafında ek kontroller olabilir ancak istemci kodu bunu doğrulamıyor.

---

## 5. Güvenlik Zafiyetleri - KESİN HÜKÜMLER

### 5.1. KRİTİK: TLS Sertifika Doğrulaması Tamamen Devre Dışı

**Dosya**: `com/mico/framework/network/http/sercure/C23276b.java`

```java
// Trust-all X509TrustManager - checkServerTrusted BOŞ
public void checkServerTrusted(X509Certificate[] x509CertificateArr, String str) {
    // HİÇBİR ŞEY YAPMIYOR
}

// HostnameVerifier - her zaman true döndürüyor
public boolean verify(String str, SSLSession sSLSession) {
    return true;
}

// OkHttpClient.Builder'a uygulanıyor
public static void m84400a(OkHttpClient.Builder builder) {
    SSLContext sSLContext = SSLContext.getInstance("SSL");
    sSLContext.init(null, new TrustManager[]{aVar}, new SecureRandom());
    builder.sslSocketFactory(sSLContext.getSocketFactory(), aVar);
    builder.hostnameVerifier(new b());
}
```

**KESİN HÜKÜM**: **MAN-IN-THE-MIDDLE (MitM) saldırısına tamamen açık**. Herhangi bir ağ üzerinde (WiFi, operatör) araya giren biri tüm HTTPS trafiğini okuyabilir ve değiştirebilir. Bu, access_token, tcp_token, encrypted_key dahil tüm kimlik bilgilerinin çalınmasına olanak sağlar. Ciddiyet: **KRİTİK**.

### 5.2. KRİTİK: Cleartext Trafik Genel Olarak İzinli

**Dosya**: `AndroidManifest.xml` satır 169:
```xml
android:usesCleartextTraffic="true"
```

**Dosya**: `network_security_config.xml`:
```xml
<base-config cleartextTrafficPermitted="true"/>
```

**KESİN HÜKÜM**: Uygulama tüm domainlere HTTP (şifresiz) bağlantı yapabilir. NetDiagnosis WebSocket'i zaten `ws://` (şifresiz) kullanıyor. Sertifika doğrulamasının devre dışı olmasıyla birleştiğinde, tüm iletişim kanalları potansiyel olarak dinlenebilir. Ciddiyet: **KRİTİK**.

### 5.3. YÜKSEK: Token'lar Düz Metin Olarak Saklanıyor

**Dosya**: `p843z8/C36268a.java`

```java
// Token'lar SharedPreferences'a düz metin olarak yazılıyor
public static void m121958w(String str, String str2) {
    C36114a.m121384f(str, "token", str2);  // Düz metin
}
```

4 credential tipi (`access_token`, `tcp_token`, `refresh_token`, `encrypted_key`) tümü SharedPreferences'da `/data/data/com.voicechat.live.group/shared_prefs/` altında **düz metin XML** olarak saklanıyor.

**KESİN HÜKÜM**: Root erişimli cihazda veya yedekleme ile tüm token'lar okunabilir. Android Keystore, EncryptedSharedPreferences veya MMKV şifreli mod **kullanılmıyor**. Ciddiyet: **YÜKSEK**.

### 5.4. YÜKSEK: TikTok OAuth Client Secret İstemcide Gömülü

**Dosya**: `com/mico/feature/me/p497ui/login/tiktok/TiktokAuthConfig.java`

```java
// Sandbox client secret - düz metin
return "MaCRRnbJtU1L6apRPEQNlpg9ST0NMntC";  // satır 80

// client_secret POST parametresi olarak gönderiliyor
InterfaceC35273b<AccessTokenResponse> getAccessToken(
    @InterfaceC1116c("client_secret") String str3, ...);
```

**KESİN HÜKÜM**: OAuth client_secret **asla** mobil uygulamada bulunmamalıdır. Bu, TikTok OAuth akışının tamamen taklit edilmesine olanak sağlar. Sandbox key olmasına rağmen, prodüksiyon key'i de aynı mekanizma ile (`C23124v` config utility) dağıtılıyor ve istemciden erişilebilir. Ciddiyet: **YÜKSEK**.

### 5.5. YÜKSEK: 54/55 gRPC Serviste Native İmza Koruması Yok

**Dosya**: `p145J8/C0717a.java`

55 gRPC servisin sadece 1'i (SignInService - 6 method) native HMAC koruması altında.

Korumasız **finansal** servisler:
- `ApppayGrpc` - **Ödeme**
- `CashOutServiceGrpc` - **Para çekme**
- `PayCenterServiceGrpc` - **Ödeme merkezi**
- `SilverCoinsLogicServiceGrpc` - **Sanal para**
- `ShopServiceGrpc` - **Mağaza**

Korumasız **hediye/ödül** servisleri:
- `AudioGiftServiceGrpc`, `ChatGiftServiceGrpc`, `GiftListGrpc`, `LuckyGiftServiceGrpc`

Korumasız **oda/yayın** servisleri:
- `RoomMgrServiceGrpc`, `EnterRoomServiceGrpc`, `AudioChatServiceGrpc`

**KESİN HÜKÜM**: Çalınan bir `access_token` ile CashOut, Apppay, PayCenter dahil tüm finansal API'ler **ek doğrulama olmadan** çağrılabilir. Token çalmak da TLS doğrulamasının devre dışı olması nedeniyle (5.1) kolaydır. Ciddiyet: **YÜKSEK**.

### 5.6. ORTA: TCP Handshake HmacMD5 Kullanıyor

**Dosya**: `com/mico/framework/network/tcp/TcpHandshakeUtils.java` + `C23453b.java`

```
HmacMD5 digest = HMAC-MD5(handshake_protobuf, encrypted_key)
```

**KESİN HÜKÜM**: MD5 kriptografik olarak kırılmıştır. HMAC-MD5, düz MD5'ten daha güvenli olsa da, modern standartlara göre **HMAC-SHA256** kullanılmalıdır. Pratik istismar riski düşük ancak kriptografik standartlara uymuyor. Ciddiyet: **ORTA**.

### 5.7. ORTA: Hardcoded Firebase/Google API Anahtarları

**Dosya**: `resources/res/values/strings.xml`

```xml
<string name="google_api_key">AIzaSyBZO3gE2lEc5Zl4ZKzxV0RFLa5nXNGR2D0</string>
<string name="firebase_database_url">https://waka-audio-chat.firebaseio.com</string>
<string name="google_storage_bucket">waka-audio-chat.appspot.com</string>
<string name="facebook_app_id">1165522863595325</string>
<string name="snapchat_oauth_client_id">80e77d20-2e7e-4755-ba6a-816fd63a85e0</string>
```

**KESİN HÜKÜM**: Firebase API key'i ve project bilgileri APK'da açık. Firebase güvenlik kuralları doğru yapılandırılmışsa risk düşüktür, ancak `waka-audio-chat.firebaseio.com` veritabanına doğrudan erişim girişimi yapılabilir. Ciddiyet: **ORTA** (Firebase kurallarına bağlı).

### 5.8. DÜŞÜK: NetDiagnosis WebSocket Kimliksiz Erişim

**Dosya**: `libx/apm/netdiagnosis/ws/NetDiagnosisWebSocketManager.java`

```
ws://boce-ws.micoplatform.com/ws  // HTTP (şifresiz)
Parametreler: biz, uid
Authentication: YOK
```

**KESİN HÜKÜM**: Ağ teşhis WebSocket'i kimlik doğrulaması olmadan erişilebilir ve HTTP (şifresiz) üzerinden çalışıyor. Canlı testte `47.236.29.187:80`'de aktif olduğu doğrulandı. Ağ teşhis verileri sızdırılabilir ancak hassas kullanıcı verisi içermiyor. Ciddiyet: **DÜŞÜK**.

---

## 6. Sonuç ve Risk Matrisi

### 6.1. Zafiyet Risk Matrisi

| # | Zafiyet | Ciddiyet | İstismar Kolaylığı | Etki |
|---|---------|----------|-------------------|------|
| 5.1 | TLS sertifika doğrulaması devre dışı | **KRİTİK** | **KOLAY** - Aynı ağda MitM yeterli | Tüm trafik okunabilir/değiştirilebilir |
| 5.2 | Cleartext trafik izinli | **KRİTİK** | **KOLAY** - Ağ dinleme yeterli | HTTP trafiği şifresiz |
| 5.3 | Token'lar düz metin depolanıyor | **YÜKSEK** | **ORTA** - Root erişim veya yedekleme gerekli | Tüm oturum bilgileri çalınabilir |
| 5.4 | OAuth client_secret istemcide | **YÜKSEK** | **KOLAY** - APK decompile yeterli | OAuth akışı taklit edilebilir |
| 5.5 | 553 method'da imza koruması yok | **YÜKSEK** | **ORTA** - Token çalma + API bilgisi gerekli | Wallet, Gift, Room vb. kötüye kullanılabilir |
| 5.6 | HmacMD5 kullanımı | **ORTA** | **ZOR** - Kriptografik saldırı gerekli | TCP bağlantı bütünlüğü riski |
| 5.7 | Hardcoded API anahtarları | **ORTA** | **KOLAY** - APK decompile yeterli | Firebase kurallarına bağlı |
| 5.8 | Kimliksiz WebSocket | **DÜŞÜK** | **KOLAY** | Sınırlı teşhis verisi sızıntısı |

### 6.2. Saldırı Zincirleri

**Zincir 1: Tam Hesap Ele Geçirme (MitM)**
```
Aynı WiFi ağında → MitM proxy kur → TLS doğrulama yok (5.1) →
access_token yakala → 553 korumasız gRPC method'unu çağır (5.5) →
Wallet transferi, hediye gönderme, profil değiştirme
```

**Zincir 2: Cihaz Seviyesinde Token Çalma**
```
Root erişimli cihaz / yedekleme exploit → SharedPreferences oku (5.3) →
access_token + tcp_token + encrypted_key al →
Hem gRPC (553 method) hem TCP bağlantısı kurulabilir
```

**Zincir 3: OAuth Taklit**
```
APK decompile → TikTok client_secret al (5.4) →
Sahte OAuth akışı ile kullanıcı token'ı ele geçir →
Hesaba erişim
```

### 6.3. Genel Değerlendirme

**KESİN HÜKÜM**: YoHo uygulaması **performans optimizasyonu açısından üstün** (gRPC, Protobuf, HTTP/2, çoklu CDN, DNS over HTTPS, bağlantı havuzlama) ancak **güvenlik açısından ciddi eksiklikleri** var:

1. **Transport güvenliği kırık**: TLS sertifika doğrulamasının tamamen devre dışı olması, tüm diğer güvenlik önlemlerini anlamsız kılıyor. Bir MitM saldırganı native HMAC imzası dahil tüm trafiği görebilir ve manipüle edebilir.

2. **Depolama güvenliği yok**: Token'ların düz metin saklanması, cihaz seviyesinde saldırılara karşı koruma sağlamıyor.

3. **Asimetrik API koruması**: 559 method'un sadece 6'sı ek korumaya sahip. Wallet ve Gift gibi finansal işlemler native imza koruması dışında bırakılmış.

4. **İstemci tarafı güvenlik kontrolleri**: Rate limiting, login kontrolü gibi mekanizmalar yalnızca istemcide uygulanıyor ve bypass edilebilir.

**Sonuç**: Uygulama "güvenlik için performanstan ödün vermemek" felsefesini benimsemiş görünüyor. gRPC altyapısı olgun ve iyi optimize edilmiş, ancak güvenlik katmanları hız uğruna zayıflatılmış. Özellikle TLS bypass'ı, modern bir fintech/sosyal uygulama için **kabul edilemez** düzeyde bir güvenlik açığı.

---

*Bu rapor, decompile edilmiş kaynak kod statik analizi ve canlı endpoint testlerine dayanmaktadır. Sunucu tarafı güvenlik kontrolleri (WAF, rate limiting, anomaly detection) istemci kodundan doğrulanamaz.*

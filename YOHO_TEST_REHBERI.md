# YoHo v5.47.00 - Kendi Hesabınla Test Rehberi

## Ön Koşullar

- Kendi YoHo hesabın (giriş yapılmış)
- Bilgisayarında Python 3.8+ veya Go kurulu
- Aynı WiFi ağında telefon ve bilgisayar
- Android telefon (root gerekmez)

---

## ADIM 1: Token Yakalama (mitmproxy)

YoHo TLS sertifika doğrulaması yapmıyor, bu yüzden proxy sertifikası yüklemeye bile gerek yok.

### 1.1 mitmproxy Kurulumu

```bash
# macOS
brew install mitmproxy

# Linux
pip install mitmproxy

# Windows
# https://mitmproxy.org/ adresinden indir
```

### 1.2 Proxy Başlat

```bash
mitmproxy --listen-port 8080 --set block_global=false
```

### 1.3 Telefon Proxy Ayarı

```
Ayarlar → WiFi → Bağlı ağa uzun bas → Proxy → Manuel
  Host: [bilgisayarının IP'si, ör. 192.168.1.100]
  Port: 8080
```

### 1.4 YoHo'yu Aç ve Giriş Yap

mitmproxy'de tüm istekleri göreceksin. Şu header'ları not al:

```
x-auth-token: <senin_tokenin>
uid: <senin_uid>
region: <bölge_kodu>
did: <cihaz_id>
```

### 1.5 Alternatif: Doğrudan Cihazdan Token Çıkarma

Root erişimin varsa veya ADB backup alabiliyorsan:

```bash
# ADB ile SharedPreferences çekme (root gerekli)
adb shell "su -c 'cat /data/data/com.voicechat.live.group/shared_prefs/*.xml'"

# Token'lar bu XML dosyalarında düz metin olarak saklanıyor:
# - access_token
# - tcp_token
# - refresh_token
# - encrypted_key
```

---

## ADIM 2: grpcurl Kurulumu

```bash
# macOS
brew install grpcurl

# Linux
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest

# veya binary indir:
# https://github.com/fullstorydev/grpcurl/releases
```

---

## ADIM 3: gRPC İstek Yapısı

### 3.1 Temel Bağlantı Bilgileri

```
Host: rpc.yoho.media
Port: 443
Protokol: gRPC over HTTP/2 + TLS
Auth: x-auth-token header
```

### 3.2 Zorunlu Header'lar (Her İstekte)

```
x-auth-token: <senin_access_tokenin>
uid: <senin_uid>
region: <bölge_kodun>
lang: <dil, ör. en>
version: <build_version, ör. 5470000>
os: 2
pkg: com.voicechat.live.group
did: <cihaz_android_id>
tz: <timezone_offset>
tz-id: <timezone_id, ör. Europe/Istanbul>
request-time-s: <unix_timestamp_ms>
```

### 3.3 grpcurl ile Test Komutu Şablonu

YoHo Protobuf reflection desteklemiyor (muhtemelen), bu yüzden proto dosyaları olmadan doğrudan binary data göndermelisin. Ancak önce reflection'ı test et:

```bash
# Reflection test (muhtemelen çalışmaz)
grpcurl \
  -H "x-auth-token: SENIN_TOKENIN" \
  -H "uid: SENIN_UID" \
  -H "region: TR" \
  -H "lang: tr" \
  -H "version: 5470000" \
  -H "os: 2" \
  -H "pkg: com.voicechat.live.group" \
  -H "did: SENIN_DID" \
  -H "request-time-s: $(date +%s000)" \
  rpc.yoho.media:443 \
  list
```

### 3.4 mitmproxy ile Binary Protobuf Yakalama ve Tekrar Gönderme

Reflection yoksa en etkili yöntem: mitmproxy ile gerçek istekleri yakala, sonra tekrar gönder.

```python
# save_grpc_requests.py - mitmproxy addon
# mitmproxy -s save_grpc_requests.py --listen-port 8080

import json
import os
from mitmproxy import http

OUTPUT_DIR = "./captured_grpc"
os.makedirs(OUTPUT_DIR, exist_ok=True)

def response(flow: http.HTTPFlow):
    if flow.request.headers.get("content-type", "").startswith("application/grpc"):
        method = flow.request.path
        headers = dict(flow.request.headers)
        
        safe_name = method.replace("/", "_").strip("_")
        
        # Header'ları kaydet
        with open(f"{OUTPUT_DIR}/{safe_name}_headers.json", "w") as f:
            json.dump(headers, f, indent=2)
        
        # Request body'yi kaydet (protobuf binary)
        if flow.request.content:
            with open(f"{OUTPUT_DIR}/{safe_name}_request.bin", "wb") as f:
                f.write(flow.request.content)
        
        # Response body'yi kaydet
        if flow.response and flow.response.content:
            with open(f"{OUTPUT_DIR}/{safe_name}_response.bin", "wb") as f:
                f.write(flow.response.content)
        
        print(f"[CAPTURED] {method} -> {len(flow.request.content or b'')} bytes req, "
              f"{len(flow.response.content or b'')} bytes rsp")
```

Çalıştır:
```bash
mitmproxy -s save_grpc_requests.py --listen-port 8080
# Sonra telefondan YoHo'da işlem yap
# captured_grpc/ klasöründe tüm istekler binary olarak kaydedilir
```

### 3.5 Yakalanan İsteği Tekrar Gönder (Replay)

```bash
# Yakalanan binary protobuf'u grpcurl ile gönder
grpcurl \
  -H "x-auth-token: SENIN_TOKENIN" \
  -H "uid: SENIN_UID" \
  -H "region: TR" \
  -H "lang: tr" \
  -H "version: 5470000" \
  -H "os: 2" \
  -H "pkg: com.voicechat.live.group" \
  -H "did: SENIN_DID" \
  -H "request-time-s: $(date +%s000)" \
  -binary \
  -d "$(cat captured_grpc/ISTEK_request.bin | base64)" \
  rpc.yoho.media:443 \
  proto.user_svr.UserSvrService/GetUserProfile
```

---

## ADIM 4: Güvenlik Testleri

### Test 1: Kullanıcı Bilgisi Çekme (Zararsız)

```
Servis: user_info.UserInfoService
Method'lar: GetUserProfile, GetUserMiniInfo, GetUserInfoFromClient
```

mitmproxy'de kendi profil sayfanı açtığında bu isteği yakalayacaksın. Yakalanan isteği replay ederek kendi bilgilerini çek.

### Test 2: Token Geçerliliği Testi

Aynı token'ı farklı cihaz bilgileriyle gönder:

```bash
# Orijinal did ile
grpcurl -H "did: SENIN_GERCEK_DID" ... proto.user_svr.UserSvrService/GetUserProfile

# Sahte did ile (cihaz bağlama var mı test et)
grpcurl -H "did: fake_device_12345" ... proto.user_svr.UserSvrService/GetUserProfile

# Eğer ikisi de çalışıyorsa: cihaz bağlama YOK
```

### Test 3: Yetki Kontrolü (Kendi Hesabında)

```
# Kendi cüzdan bilgini çek
paycenter.PayCenterService - yakalanan method

# Kendi hediye listeni çek
proto.giftlist.GiftList - yakalanan method

# Kendi oda bilgini çek
proto.enter_room.EnterRoomService - yakalanan method
```

### Test 4: Rate Limiting Testi

```bash
# Aynı isteği 10 kez hızlı gönder
for i in $(seq 1 10); do
  grpcurl \
    -H "x-auth-token: SENIN_TOKENIN" \
    -H "uid: SENIN_UID" \
    ... \
    rpc.yoho.media:443 \
    proto.user_svr.UserSvrService/GetUserProfile
  echo "--- Request $i done ---"
done
# Sunucu tarafında rate limiting var mı gözlemle
```

### Test 5: Expire Edilmiş Token Testi

```bash
# Birkaç gün eski token ile dene (kendi eski tokenin)
grpcurl \
  -H "x-auth-token: ESKI_TOKENIN" \
  ... \
  rpc.yoho.media:443 \
  proto.user_svr.UserSvrService/GetUserProfile
# Token expire süresi ne kadar?
```

---

## ADIM 5: Tam gRPC Servis Haritası

Aşağıdaki servisler `rpc.yoho.media:443` üzerinde çalışıyor. Tümü `x-auth-token` ile korunuyor, **hiçbirinde native HMAC koruması yok** (SignInService hariç):

### Finansal Servisler (Yüksek Risk)
```
cashout.CashOutService          - Para çekme
paycenter.PayCenterService      - Ödeme merkezi
silver_coins_logic.SilverCoinsLogicService - Sanal para
proto.goods.ShopService         - Mağaza
proto.second_charge.SecondChargeService - İkinci şarj
```

### Hediye/Ödül Servisleri
```
proto.audio.AudioGiftService    - Sesli hediye gönderme
  → AudioSendGift, AudioCartSendGift, AudioSendTrick, GetGiftTips
proto.chat_send_gift.ChatGiftService - Chat hediye
proto.giftlist.GiftList         - Hediye listesi
proto.lucky_gift.LuckyGiftService - Şanslı hediye
proto.red_env.RedEnvelopeService - Kırmızı zarf
proto.red_rain.RedRainService   - Kırmızı yağmur
proto.treasure_box.TreasureBoxService - Hazine kutusu
```

### Kullanıcı Servisleri
```
user_info.UserInfoService       - Kullanıcı bilgi
proto.user_svr.UserSvrService   - Kullanıcı sunucu
  → GetUserProfile, GetUserMiniInfo, GetUserInfoFromClient, 
    GetUserInfoBatFromClient, DecodeUsText
proto.friendship.FriendShipService - Arkadaşlık
fans.FansService                - Takipçi
proto.guardian_relation.GuardianRelationService - Koruyucu ilişkisi
proto.new_user.NewUserService   - Yeni kullanıcı
```

### Oda/Yayın Servisleri
```
proto.roommgr.RoomMgrService    - Oda yönetimi
proto.enter_room.EnterRoomService - Oda giriş
proto.room_rcmd.RoomRcmdService - Oda önerisi
proto.mic_manager.RoomMicManagerService - Mikrofon yönetimi
proto.audio_chat.AudioChatService - Sesli sohbet
proto.room_manager.RoomManagerService - Oda yöneticisi
proto.video_room.VideoRoomService - Video oda
broadcast_share.BroadcastShareService - Yayın paylaşma
proto.room_pk.RoomPkV2Service   - Oda PK
proto.pk_feast.PKFeastService   - PK şöleni
proto.team_pk.TeamPKService     - Takım PK
```

### Oyun Servisleri
```
proto.game.FastGameService      - Hızlı oyun
  → EntryQuery, FastGame
proto.game.GameMatchingService  - Oyun eşleştirme
proto.buddy.GameBuddyService   - Oyun arkadaşı
proto.Game.GameTaskRewardService - Oyun ödülü
proto.audio.GameCenterService   - Oyun merkezi
proto.game_room.H5GameRoomV2Service - H5 oyun oda
proto.game_pk.GamePkService     - Oyun PK
```

### Diğer Servisler
```
sign.SignInService              - Giriş (6 method HMAC korumalı!)
SvrConfig.SvrConfig             - Sunucu config
report.Report                   - Raporlama
proto.translate.TranslateService - Çeviri
proto.rankinglist.RankingListService - Sıralama
proto.daily_task.DailyTaskService - Günlük görev
proto.audio_task.AudioTaskService - Ses görevi
proto.audio_reward_task.AudioRewardTaskService - Ses ödül görevi
proto.audio_visit.AudioVisitService - Ziyaret
proto.audio_order.AudioOrderService - Sipariş
proto.activity_square.ActivitySquareService - Aktivite
proto.activity_entrace.ActivityEntraceService - Aktivite giriş
proto.greedy_activity.GreedyActivityService - Açgözlü aktivite
proto.family.FamilyService      - Aile
proto.friendly_point.FriendlyPointService - Dostluk puanı
proto.meet.MeetService          - Tanışma
proto.cp_task.CpTaskService     - CP görevi
proto.agency.AgencyService      - Ajans
agency_task.AgencyTask          - Ajans görevi
proto.auction.AuctionService    - Açık artırma
proto.auction.StageService      - Sahne
proto.raise_national_flag.RaiseNationalFlagService - Bayrak
proto.voice_identify.VoiceIdentifyService - Ses tanıma
proto.privilege.PrivilegeService - Ayrıcalık
proto.moment.MomentService     - An
proto.moment.MomentListService  - An listesi
proto.score_board.ScoreboardService - Puan tablosu
proto.badge_svr.BadgeSvrService - Rozet
proto.noble_svr.NobleSvrService - Asil
proto.chat_svr.ChatSvrService   - Chat
proto.sensitive_words.SensitiveWordsService - Hassas kelime
proto.secure_detection.CommonDetectionService - Güvenlik tespiti
proto.competion_svr.CompetionService - Yarışma
proto.guinness_record.GuinnessRecordService - Rekor
proto.universal_popup.UniversalPopupService - Popup
lottery_svr.LotteryService      - Çekiliş
ad_report.AdReportService       - Reklam raporu
user_history_record.UserHistoryRecordService - Geçmiş
proto.guardian_relation.MentorService - Mentor
```

---

## ADIM 6: Python ile Otomatik Test Script'i

```python
#!/usr/bin/env python3
"""
YoHo gRPC Security Test Script
Kendi hesabınla test etmek için kullan.
"""

import grpc
import time
import sys

# ---- KENDİ BİLGİLERİNİ GİR ----
AUTH_TOKEN = "BURAYA_KENDI_TOKENINI_YAZ"
UID = "BURAYA_KENDI_UID"
REGION = "TR"
DID = "BURAYA_KENDI_DID"
# ---- /KENDİ BİLGİLERİN ----

TARGET = "rpc.yoho.media:443"

def make_metadata():
    return [
        ("x-auth-token", AUTH_TOKEN),
        ("uid", UID),
        ("region", REGION),
        ("lang", "tr"),
        ("version", "5470000"),
        ("os", "2"),
        ("pkg", "com.voicechat.live.group"),
        ("did", DID),
        ("tz", "10800"),
        ("tz-id", "Europe/Istanbul"),
        ("request-time-s", str(int(time.time() * 1000))),
    ]

def test_connection():
    """Temel bağlantı testi"""
    creds = grpc.ssl_channel_credentials()
    channel = grpc.secure_channel(TARGET, creds)
    
    try:
        grpc.channel_ready_future(channel).result(timeout=10)
        print("[OK] gRPC kanalı açıldı: " + TARGET)
        return channel
    except grpc.FutureTimeoutError:
        print("[HATA] Bağlantı timeout")
        return None

def test_unary_call(channel, service_method, request_bytes=b''):
    """Generic unary gRPC çağrısı"""
    metadata = make_metadata()
    
    try:
        response = channel.unary_unary(
            "/" + service_method,
            request_serializer=lambda x: x,
            response_deserializer=lambda x: x,
        )(request_bytes, metadata=metadata, timeout=30)
        
        print(f"[OK] {service_method} -> {len(response)} bytes yanıt")
        return response
    except grpc.RpcError as e:
        print(f"[{e.code().name}] {service_method} -> {e.details()}")
        return None

def test_token_binding(channel):
    """Cihaz bağlama testi - aynı token farklı did ile"""
    method = "proto.user_svr.UserSvrService/GetUserProfile"
    
    # Orijinal did ile
    meta_original = make_metadata()
    
    # Sahte did ile
    meta_fake = [(k, v) for k, v in make_metadata() if k != "did"]
    meta_fake.append(("did", "fake_device_test_12345"))
    
    try:
        resp1 = channel.unary_unary(
            "/" + method,
            request_serializer=lambda x: x,
            response_deserializer=lambda x: x,
        )(b'', metadata=meta_original, timeout=30)
        print(f"[OK] Orijinal DID: {len(resp1)} bytes")
    except grpc.RpcError as e:
        print(f"[{e.code().name}] Orijinal DID: {e.details()}")
    
    try:
        resp2 = channel.unary_unary(
            "/" + method,
            request_serializer=lambda x: x,
            response_deserializer=lambda x: x,
        )(b'', metadata=meta_fake, timeout=30)
        print(f"[OK] Sahte DID: {len(resp2)} bytes -> CİHAZ BAĞLAMA YOK!")
    except grpc.RpcError as e:
        print(f"[{e.code().name}] Sahte DID: {e.details()}")
        if e.code() == grpc.StatusCode.UNAUTHENTICATED:
            print("   -> Cihaz bağlama VAR (sunucu reddetti)")

def test_rate_limiting(channel, count=10):
    """Sunucu tarafı rate limiting testi"""
    method = "proto.user_svr.UserSvrService/GetUserMiniInfo"
    
    print(f"\n--- Rate Limiting Testi ({count} istek) ---")
    for i in range(count):
        try:
            resp = channel.unary_unary(
                "/" + method,
                request_serializer=lambda x: x,
                response_deserializer=lambda x: x,
            )(b'', metadata=make_metadata(), timeout=10)
            print(f"  [{i+1}/{count}] OK - {len(resp)} bytes")
        except grpc.RpcError as e:
            print(f"  [{i+1}/{count}] {e.code().name}: {e.details()}")
            if e.code() == grpc.StatusCode.RESOURCE_EXHAUSTED:
                print("   -> SUNUCU TARAFI RATE LIMITING TESPİT EDİLDİ!")
                break

def main():
    if AUTH_TOKEN == "BURAYA_KENDI_TOKENINI_YAZ":
        print("HATA: Önce kendi token bilgilerini script'e gir!")
        print("mitmproxy ile token yakala, sonra bu script'i düzenle.")
        sys.exit(1)
    
    print("=" * 60)
    print("YoHo gRPC Güvenlik Testi - Kendi Hesabın")
    print("=" * 60)
    
    # Bağlantı testi
    channel = test_connection()
    if not channel:
        sys.exit(1)
    
    print("\n--- Temel API Testleri ---")
    
    # Kullanıcı servisleri
    test_unary_call(channel, "proto.user_svr.UserSvrService/GetUserProfile")
    test_unary_call(channel, "proto.user_svr.UserSvrService/GetUserMiniInfo")
    test_unary_call(channel, "user_info.UserInfoService/GetUserInfo")
    
    # Finansal servisler (sadece okuma)
    test_unary_call(channel, "proto.giftlist.GiftList/GetGiftList")
    test_unary_call(channel, "proto.privilege.PrivilegeService/GetUserPrivilege")
    
    # Config
    test_unary_call(channel, "SvrConfig.SvrConfig/GetSvrConfig")
    
    print("\n--- Cihaz Bağlama Testi ---")
    test_token_binding(channel)
    
    print("\n--- Rate Limiting Testi ---")
    test_rate_limiting(channel)
    
    channel.close()
    print("\n" + "=" * 60)
    print("Test tamamlandı. Sonuçları değerlendir.")

if __name__ == "__main__":
    main()
```

### Kullanım:

```bash
# Gerekli kütüphane
pip install grpcio

# Token bilgilerini düzenle, sonra çalıştır
python3 yoho_test.py
```

---

## Güvenlik Notu

Bu rehber **yalnızca kendi hesabın** ile test içindir. Başkasının hesabına erişim, başkasının token'ını kullanma veya üçüncü taraf kullanıcılara karşı işlem yapma **yasadışıdır**. Test sonuçlarını sorumlu açıklama (responsible disclosure) kapsamında YoHo'ya bildirebilirsin.

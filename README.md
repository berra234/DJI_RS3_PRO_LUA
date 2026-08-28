# DJI RS2 / RS3 Pro Gimbal Driver — ArduPilot Lua

ArduPilot'un DJI RS serisi gimbal'leri CAN üzerinden sürdüğü Lua mount driver'ı. Bu depodaki sürüm, ArduPilot'un resmi `mount-djirs2-driver.lua` dosyasına **eski firmware sürümleriyle uyumluluk düzeltmesi** eklenmiş halidir.

Depoda ayrıca, gerçek GPS olmadan yer testi yapabilmek için kullanılan sahte GPS firmware'i bulunur: [`NMEA_GPS/`](NMEA_GPS/README.md)

---

## Bu sürümdeki değişiklikler

| Değişiklik | Sebep |
|---|---|
| `get_vehicle_yaw_rad()` uyumluluk katmanı | `ahrs:get_yaw_rad()` binding'i ArduPilot 4.6 ile geldi. Daha eski firmware'de metot `nil` olduğu için script `attempt to call a nil value (method 'get_yaw_rad')` hatasıyla çöküyordu. Artık binding varsa yenisi, yoksa `ahrs:get_yaw()` kullanılıyor. |
| Başlangıç mesajına `(yaw-compat)` etiketi | SD karttaki dosyanın gerçekten güncellenip güncellenmediğini tahmin etmeden görebilmek için. |

Düzeltme, dosyanın kendi üslubuna uyar — script zaten `mount:set_natively_supported_mount_target_types` için aynı guard desenini kullanıyordu.

---

## Gereksinimler

- CAN portu olan bir uçuş kontrolcüsü (ArduPilot)
- SD kart (script `APM/scripts/` klasöründen çalışır)
- DJI RS2 / RS3 / RS3 Pro gimbal

---

## Kurulum

### 1. Parametreler

Mission Planner → **Full Parameter List**'e yapıştır:

```
SCR_ENABLE      = 1
SCR_HEAP_SIZE   = 120000
MNT1_TYPE       = 9
CAN_P1_DRIVER   = 1
CAN_D1_PROTOCOL = 10
CAN_P1_BITRATE  = 1000000
```

CAN2 portunu kullanıyorsan son üçü yerine:

```
CAN_P2_DRIVER   = 2
CAN_D2_PROTOCOL = 10
CAN_P2_BITRATE  = 1000000
```

Ardından **FC'yi reboot et.**

### 2. Script'i yükle

`mount-djirs2-driver.lua` dosyasını SD kartta `APM/scripts/` klasörüne kopyala. İki yol var:

- **SD kartı çıkararak** (en garantisi): kartı çıkar, kopyala, geri tak.
- **MAVFtp ile**: Mission Planner → **Config/Tuning → MAVFtp** → `APM/scripts/` klasörüne yükle.

Sonra **FC'yi tekrar reboot et.**

> [!IMPORTANT]
> ArduPilot, script dosyası değiştiğinde onu otomatik yeniden yüklemez. Her değişiklikten sonra reboot şart.

### 3. Doğrula

Messages sekmesinde şu satırı ara:

```
DJIR: mount driver started (yaw-compat)
```

Sonundaki `(yaw-compat)` etiketi, güncel dosyanın çalıştığının kesin kanıtıdır. Etiketsiz eski mesajı görüyorsan dosya SD karta ulaşmamıştır.

---

## Kablolama

```
FC CANH  ──────────►  Gimbal CANH
FC CANL  ──────────►  Gimbal CANL
FC GND   ──────────►  Gimbal GND
```

CAN, gimbal kolundaki **RSA/CAN** portuna bağlanır — USB-C veya RSS portu değil.

**Sonlandırma direnci kontrolü:** FC kapalıyken multimetreyle CANH–CANL arasına bak.

| Ölçüm | Anlamı |
|---|---|
| ~60 Ω | İki uçta da 120 Ω terminatör var — doğru |
| ~120 Ω | Tek terminatör var — genelde çalışır, sınırda |
| Açık devre | Hat kopuk veya terminatör yok — iletişim kurulmaz |

---

## Script parametreleri

Script ilk çalıştığında bu parametreleri kendisi oluşturur:

| Parametre | Değerler | Açıklama |
|---|---|---|
| `DJIR_DEBUG` | `0` kapalı, `1` sayaçlar, `2` sayaçlar + açı raporu | Hata ayıklama seviyesi |
| `DJIR_UPSIDEDOWN` | `0` düz, `1` ters | Gimbal ters monteliyse yaw'a 180° eklenir |

---

## Nasıl çalışır

Gimbal'e **koordinat gönderilmez.** DJI RS3 Pro enlem/boylam kavramını bilmez; ona yalnızca roll/pitch/yaw açıları gider. Koordinat → açı dönüşümü ArduPilot'un C++ mount kütüphanesinde, Lua devreye girmeden önce yapılır.

| # | Adım | Nerede |
|---|---|---|
| 1 | GCS `MAV_CMD_DO_SET_ROI_LOCATION` ile lat/lon/alt gönderir | MAVLink |
| 2 | `AP_Mount` hedefi saklar ve *"aracın şu anki konumundan bu noktaya bakmak için açı ne olmalı"* hesabını yapar | ArduPilot C++ |
| 3 | Script hazır açıları `mount:get_angle_target()` ile alır | `update()` |
| 4 | Earth-frame yaw, araç yaw'ı çıkarılarak body-frame'e çevrilir | `get_vehicle_yaw_rad()` |
| 5 | Açılar ×10 int16 olarak DJI paketine yazılır (`CmdSet 0x0E`, `CmdID 0x00`) | `send_target_angles()` |
| 6 | Sıra numarası, CRC-16 ve CRC-32 doldurulur | `update_msg_seq_and_crc()` |
| 7 | Paket 8'erlik CAN frame'lerine bölünüp yazılır | `send_msg()` |

Frame ID'leri: gidiş `0x223`, dönüş `0x222`.

Aracın konumu EKF'ten gelir — yani sahte GPS besliyorsan gimbal'in nişan aldığı açı doğrudan o sahte konuma bağlıdır.

---

## Hata ayıklama sayaçlarını okumak

`DJIR_DEBUG = 1` iken 5 saniyede bir şu satır basılır:

```
DJIR: r:0  w:249  fail:176,0  perr:0  to:11
       │     │      │    │      │       └ cevap zaman aşımı sayısı
       │     │      │    │      └ ayrıştırılamayan byte sayısı
       │     │      │    └ gimbal'in komutu yürütememe sayısı
       │     │      └ CAN yazma hatası sayısı
       │     └ yazılan byte sayısı
       └ OKUNAN byte sayısı
```

| Örüntü | Teşhis |
|---|---|
| `r:0` + yüksek `fail` | **CAN hattında gimbal yok.** CAN'de her frame'in başka bir düğüm tarafından ACK'lenmesi gerekir; karşıda aktif düğüm yoksa verici sonsuz yeniden dener ve `write_frame` zaman aşımına düşer. |
| `r` artıyor + `perr` yüksek | Veri geliyor ama bozuk — bitrate uyuşmazlığı veya hat gürültüsü |
| `r` ve `w` artıyor, `fail:0` | İletişim sağlıklı |

---

## Bilinen davranış: gimbal sessizken hedef gönderilmez

Gimbal cevap vermediğinde script bir **istek → zaman aşımı → istek** döngüsüne kilitlenir:

1. `request_attitude()` çağrılır, `expected_reply = ATTITUDE` işaretlenir, fonksiyon `return` eder.
2. Sonraki 100 ms boyunca `update()` her turda erken çıkar (cevap bekleniyor).
3. 100 ms dolunca `DJIR: timeout expecting 1` basılır, bayrak temizlenir.
4. Akış hemen aşağı düşer, attitude isteği sayacı da dolmuş olduğu için **yeni istek gönderilip yine `return` edilir.**

Sonuç: hedef açıların okunduğu blok (`mount:get_angle_target()` → `send_target_angles()`) her turda bu `return`'ün arkasında kalır ve **hiç çalışmaz.**

Bu bir iptal değil, **açlık (starvation)** durumudur — ROI hedefi `AP_Mount` içinde sapasağlam durur, sadece hattan dışarı çıkamaz. Gimbal cevap vermeye başladığı an döngü kırılır ve hedef açılar gönderilmeye başlar; script'te değişiklik gerekmez.

---

## Bağlantı kopması durumunda davranış

| Senaryo | Sonuç |
|---|---|
| **YKİ (yer istasyonu) bağlantısı kopar** | Gimbal atanan ROI hedefini **izlemeye devam eder.** Hedef yer istasyonunda değil FC'nin belleğinde saklanır; araç hareket ettikçe açı yeniden hesaplanır. Başlangıç konumuna dönmez. |
| **CAN bağlantısı kopar** | Gimbal **son komut edilen açıda kalır** ve kendi IMU'suyla stabilize etmeye devam eder. Script mutlak (absolute) konum komutu gönderdiği için her komut hedefi tam tarif eder; komut gelmeyince yeni hedef oluşmaz. Başlangıç konumuna dönmez. |
| **FC yeniden başlar** | Mount, `MNT1_DEFLT_MODE` parametresindeki varsayılan moda döner. |

> [!WARNING]
> Bu driver'da bağlantı kaybını algılayıp gimbal'i güvenli bir açıya çeken **failsafe yoktur.** Yalnızca `write_frame` hatalarını sayar ve denemeye devam eder. Görev profilin gerektiriyorsa bu davranışı kendin eklemelisin.

---

## Sorun giderme

| Belirti | Sebep | Çözüm |
|---|---|---|
| `attempt to call a nil value (method 'get_yaw_rad')` | Firmware 4.6'dan eski, script yeni binding kullanıyor | Bu depodaki uyumluluk düzeltmesini kullan |
| Hata mesajındaki satır numarası değişmiyor | SD karttaki dosya güncellenmemiş | Messages'ta `(yaw-compat)` etiketini ara; yoksa dosyayı tekrar yükle ve reboot et |
| `DJIR: failed to connect to CAN bus` | `CAN_Dx_PROTOCOL` scripting (`10`) değil | Parametreleri kontrol et ve reboot et |
| `DJIR: set MNT1_TYPE=9` | Mount tipi scripting değil | `MNT1_TYPE = 9` yap ve reboot et |
| `r:0` + `timeout expecting 1` | Gimbal CAN hattında yok | CANH/CANL ters bağlanmış olabilir; gimbal açık mı, terminatör direnci ~60 Ω mu kontrol et |
| Script hiç yüklenmiyor | `SCR_ENABLE = 0` veya heap yetersiz | `SCR_ENABLE = 1`, `SCR_HEAP_SIZE = 120000` |
| Gimbal ters yönde dönüyor | Ters montaj | `DJIR_UPSIDEDOWN = 1` |

---

## Kaynaklar

Orijinal script'in geliştirilmesinde kullanılan referanslar:

- [Constant Robotics DJIR SDK](https://github.com/ConstantRobotics/DJIR_SDK)
- [Ceinem's ROS node for DJI RS2](https://github.com/ceinem/dji_rs2_ros_controller)

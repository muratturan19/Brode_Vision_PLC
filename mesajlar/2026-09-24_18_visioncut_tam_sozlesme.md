# 18 · VisionCut — okuduğumuz her sinyal, yazdığımız her sinyal, aradaki tüm hesap

**Tarih:** 24 Eylül 2026, akşam
**Sürüm:** `1.0.0-rc.32` (sahadaki makinede kurulu)
**Amaç:** Yücel Bey'in isteği — iki tarafın aldığı/verdiği sinyaller ve
aralarındaki mantık, ajanınızın inceleyebileceği ayrıntıda.

Bugün makineyi dört ayrı yerde durdurduk. Dördü de bizim tarafımızdaydı ve
dördü de **bu belge olmadığı için** saatler aldı. Yazmamız gerekirdi.

Aşağıdaki her formül koddan alınmıştır, ezberden değil; dosya ve satır
verilmiştir. Ölçülen sayılar bugünün saha kaydındandır.

---

## 0. Bugünün kesim kaydı — tartışacağımız olgu

`09:59:45` – `09:59:47`, tek kesim denemesi, telemetriden:

| saat | durum | PLC X | X hızı | **PLC Y** | bizim sapma | fark |
|---|---|---:|---:|---:|---:|---:|
| 09:59:45 | CUTTING | 7,4 | 29 | **0,028** | −0,047 | −0,075 |
| 09:59:46 | CUTTING | 46,2 | 153 | **−0,012** | +0,473 | +0,485 |
| 09:59:46 | CUTTING | 78,2 | 155 | **−0,021** | +0,667 | +0,688 |
| 09:59:46 | CUTTING | 88,1 | 90 | **−0,022** | +0,803 | +0,825 |
| 09:59:46 | CUTTING | 85,7 | −1 | −0,011 | +0,864 | bıçak kalktı |
| 09:59:47 | CUTTING | 2,5 | −247 | — | — | geri dönüş |

**Y ekseni hiç hareket etmedi.** 88 mm boyunca `lrY_ActualPosition`
−0,022…+0,028 mm arasında kaldı; bu arada ölçülen hat −0,05'ten +0,86 mm'ye
gitti. Takip hatası 0,88 mm'ye çıktı ve büyüyordu.

Kesim boyunca: **43/43 karede koridor bulundu, hat kaybı 0**, kontrast medyan
0,439, güven medyan 90. Yani görü tarafı çalışıyordu. Eksen sürülmedi.

`VisionFault` bu süre boyunca **FALSE**, `fault_code = 0`. Duruş bizim
arızamızdan gelmedi.

---

## 1. PLC'den OKUDUĞUMUZ alanlar ve her birini ne için kullandığımız

`brode_vision_console/core/models.py :: PlcInputState`

| Alan | Düğüm | Nerede kullanılıyor | Kullanılmazsa ne olur |
|---|---|---|---|
| `machine_ready` | `GVL.MachineReady` | IDLE→ARMED şartı | Çevrim hiç başlamaz |
| `feed_complete` | `GVL.FeedComplete` | IDLE→ARMED şartı | Aynı |
| `feed_active` | `GVL.FeedActive` | Ölçüm kapısını kapatır | Kumaş akarken ölçüm tamponlanır |
| `actual_x_mm` | `GVL.ActualX_mm` | **Her ölçümün etiketi**; kapı; hedef X | Ölçüm konumsuz kalır |
| `x_velocity_mm_s` | `GVL.X_Velocity` | Kapı (`dX/dt > 1 mm/s`), ARMED→PRELOAD | Kapı hiç açılmaz |
| `actual_y_mm` | `GVL.ActualY_mm` | Yalnız teşhis/telemetri | — |
| `cut_end_x_mm` | `GVL.CutEndX_mm` | Kesim sonu, ileri bakış tavanı | Kesim sonu bilinmez |
| `blade_z_down` | `GVL.BladeZDown` | 603 çapraz kontrolü | Bıçak inmediği fark edilmez |
| `emergency_state` | `GVL.EmergencyState` | Anında FAULT | — |
| `cut_active` | `GVL.CutActive` | Teşhis | — |

⚠️ **Yazmadığımız hiçbir şey yok listesinde:** `xTrajectoryValid`,
`xTrajectoryFault`, `xStartPermitted`, `eMachineState`, sensör bitleri ve
fiziksel çıkışlar bizim yazma listemizde **değil**. `xTrajectoryFault`'u
**okumuyoruz da** — düğüm haritamızda yok. Sorunuzun cevabı (mesaj 18.2):
**ilk trajectory hatasında hiçbir alanımızı değiştirmiyoruz.**

---

## 2. PLC'ye YAZDIĞIMIZ alanlar — tam formüller

`brode_vision_console/services/plc_worker.py :: plc_outputs_from` (satır 236)

```python
vision_ready     = (fault_code == 0)
line_valid       = (target_y_mm is not None)
target_y_mm      = float(target_y_mm or 0.0)
target_x_mm      = cycle.target_x_mm
confidence       = cycle.confidence / 100.0      # ic skor 0..100, telde 0..1
vision_fault     = (fault_code != 0)
fault_code       = cycle.fault_code
cut_permit       = cycle.cut_permit
z_down_request   = cycle.z_down_request
z_down_at_x_mm   = z_down_at_x_mm if z_down_request else 0.0
end_buffer_ready = cycle.end_buffer_ready
```

`heartbeat` ve `udiVisionSequence` bunların dışında, **her yazma çevriminde**
`PlcManager` tarafından üretiliyor (`plc_manager.py:346`):

```python
"vision_sequence": self.next_sequence()      # (n+1) mod 2^32, her yazmada
```

Yazma sırası (`plc/opcua.py :: write_outputs`): önce tüm veri alanları **tek
toplu yazma**, sonra **ayrı bir servis çağrısında** `udiVisionSequence`.
Toplu yazma düşerse sequence hiç artırılmaz.

---

## 3. Pikselden PLC Y koordinatına — tam dönüşüm (17.2 cevabı)

### 3.1 Montaj

- Kamera **X arabasına bağlı**, bıçağın **+X yönünde 50 mm önünde**
  (24 Eyl sahada ölçüldü; önce 40 sanıyorduk).
- Kamera **Y'de sabit**. Y'yi yalnızca bıçak hareket ettirir. Kamera Y ile
  birlikte hareket **etmiyor**.
- Bu yüzden ölçüm **mutlaktır**: mevcut Y'ye göre bir düzeltme değil.
  Kapalı çevrim yok, salınım riski yok.

### 3.2 Y dönüşümü

`vision/calibration.py :: px_to_mm` (satır 25):

```python
seen = y_sign * (center_px - reference_center_px) / pixels_per_mm
Y     = seen + blade_y_offset_mm
```

| terim | sahadaki değer | anlamı |
|---|---|---|
| `center_px` | ölçülen | koridorun iki kenarının **orta noktası** (tek parlak piksel asla kullanılmaz) |
| `reference_center_px` | **620,00** | bıçağın Y=0'da kestiği profil satırı; **operatör öğretir** |
| `pixels_per_mm` | **18,325** | bilinen 20 mm'lik bir nesneyle kalibre edildi (kumpas) |
| `y_sign` | **+1** | profil indeksi artarken Y artar |
| `blade_y_offset_mm` | **0,00** | bıçak–kamera Y kaçıklığı; **kesilerek ölçülür, henüz ölçülmedi** |

**Sorunuzun cevabı — merkez çizgimiz PLC Y=0 ile nasıl çakışıyor:**
`reference_center_px` üzerinden. Operatör bıçağı kumaşa değdirir, arabayı
ofset kadar geri alır, kamera işaretin üstüne gelir ve o andaki koridor
merkezi referans olarak kaydedilir. **Yani çakışma kalibrasyonla kurulur,
hesapla değil.** `blade_y_offset_mm` hâlâ 0 ve bu bir açık maddedir: bıçak
ile kameranın Y kaçıklığı görüntüden asla anlaşılamaz, ancak kesip ölçerek
bulunur.

⚠️ Bugün bir kaza oldu: operatör yanlışlıkla "mevcut merkezi referans kaydet"
düğmesine bastı, referans 620,00 → 637,52 oldu. Geri alındı. Kalibrasyon
geçmişine yazılmıyordu, o da düzeltildi.

### 3.3 X dönüşümü ve zaman eşlemesi

```python
camera_x_mm        = interpolator.x_at(frame.monotonic_ts)   # karenin CEKILDIGI an
target_blade_x_mm  = camera_x_mm + 50.0                      # kamera ofseti
```

`pipeline.py:305`. Ölçüm tampona **`target_blade_x_mm`** ile giriyor
(`measurement_buffer.py:83`) — yani "bıçak bu X'e geldiğinde bu Y".

**Zaman eşlemesi (17.2):** her kare yakalandığı andaki `monotonic_ts` ile
damgalanıyor. `PositionInterpolator` PLC'den gelen `(zaman, ActualX)`
örneklerini tutuyor ve o zamana ait X'i **enterpolasyonla** veriyor —
gönderim anındaki X'i eklemiyoruz. Uyarınız (*"eski kareye gönderim anındaki
ActualX'i eklemek ölçümü başka kumaş noktasına taşır"*) bizde uygulanmış
durumda.

⚠️ Sınır: enterpolatör 50 ms'ten eski bir örnekle çalışmayı reddediyor;
o karede X etiketi `None` döner ve kare **tamponlanmaz**. 175 mm/s'de 50 ms
= 8,75 mm.

Bugün ölçülen uçtan uca gecikme: **40–74 ms** (ekranda "Gecikme").

---

## 4. `TargetX` nasıl seçiliyor — bugünkü duruşun merkezi

`sync/cut_state_machine.py :: _target_x_ahead` (satır 549)

### 4.1 Duruşta (state 40/50/60)

```python
TargetX = ActualX + 50        # kameranin o anda baktigi kumas X'i
TargetY = duruşta oturmuş karelerden ölçülen hattın mutlak Y'si
```

50, uydurulmuş bir ileri nokta **değil**: ölçüm gerçekten oraya ait.

### 4.2 Kesimde (state 80)

```python
tavan   = max(0, 50 - 2 * kare_adimi)            # kare_adimi = kesim_hizi / fps
ileri   = ActualX + min(target_lookahead_mm, tavan)
ileri   = min(ileri, CutEndX_mm)
ileri   = min(ileri, tamponun_en_ileri_X'i)      # OLCULMEMIS X'e asla isaret etmeyiz
TargetX = max(ActualX + 1.0, ileri)
```

`target_lookahead_mm` şu an **20 mm**.

### 4.3 ⚠️ Bugünün duruşunu bu açıklıyor

X=46,2'deki gerçek kayıttan:

```
TargetX = 46,2 + 20 = 66,2        ->  DeltaX = 20,0 mm
TargetY ~ +0,6   (hattin o X'teki Y'si)
Y0      = -0,012 (takip kapali, gercek Y)
DeltaY  = 0,61 mm
egim    = 0,61 / 20 = 0,0305      ->  lrMaxAllowedSlope = 0,020   ASILIYOR
```

Yani **paketimiz reddediliyor, Y komut almıyor, hata birikiyor.**

⚠️ Dikkat: **hattın kendi eğimi sorun değil.** 88 mm'de 0,9 mm = **0,010**,
sınırın yarısı. Sorun bizim ileri bakışımızın kısalığı: 20 mm'lik bir koşuda
0,6 mm düzeltme istemek, hattın kendisinden üç kat dik görünüyor.

**Düzeltme yönümüz:** `DeltaX >= DeltaY / 0,020`. 0,6 mm düzeltme için en az
30 mm gerekiyor; tavanımız `50 − 2×kare_adımı ≈ 36 mm`. Yani ileri bakışı
20 → 35 yapmak eğimi 0,017'ye indirir.

**Bunu henüz göndermedik.** Tezgahta sizin state akışınızı taklit edip
doğrulamadan makineye koymayacağız. Sizden de görüşünüzü istiyoruz: ileri
bakışı uzatmak sizin trajectory hesabınızla uyumlu mu?

---

## 5. Ölçüm kapısı — hangi kare tampona girer

`cut_state_machine.py :: evaluate_gate` (satır 491)

```python
kapi_acik =
        durum in (PRELOAD, READY_TO_CUT, CUTTING)
    and not FeedActive
    and X_Velocity > 1.0 mm/s            # ileri hareket
    and 0 <= (ActualX + 50) <= CutEndX_mm
    and durum != FAULT
```

Kapalıysa kare **atılır**, tampona girmez. Bu, dönüş stroğunda kameranın
kumaşın üstünden geriye uçarken ölçüm biriktirmesini engelliyor.

⚠️ Kapı **`BladeZDown`'a bağlı değil**, bilerek. Bıçak X=0'da inen bir
makinede Z'ye bağlamak kapıyı kesim boyunca kapalı tutardı.

---

## 6. Tampondan hedef — enterpolasyon, uydurma yok

`sync/measurement_buffer.py :: get_target_for_x` (satır 113)

```python
TargetY = iki komsu olcum arasinda dogrusal enterpolasyon
```

⚠️ **Eğim uzatması yok.** Son ölçümün ötesinde dürüst cevap "veri yok"tur.
Tek istisna kumaş sonu: son `end_hold_mm` boyunca **son değer tutulur**
(uzatılmaz), çünkü kamera kumaşı bıçaktan bir kare adımı önce terk eder.

Mesaj 17 §7'deki uyarınıza karşılık: **eski hedefi yeni ölçüm gibi
etiketlemiyoruz.** Tampon kapsamıyorsa `line_valid` düşer.

---

## 7. Durum makinemiz ve sizin state'lerinizle eşleşmesi

| Bizim | Şartı | Sizin karşılığınız |
|---|---|---|
| `IDLE` | — | 20 Perde Bekliyor |
| `ARMED` | `MachineReady ∧ ¬FeedActive ∧ FeedComplete` | 30 Baskı İniyor / 40 Kamera Bekleniyor |
| `PRELOAD` | ARMED + `X_Velocity > 1` | *(bu makinede atlanıyor)* |
| `CUTTING` | run-in yoksa doğrudan | 80 Kesim |
| `FAULT` | arıza | 900 / 500 |

### 7.1 ARMED'da ne yapıyoruz

1. Kumaşın **oturmasını** bekliyoruz (σ < `settle_sigma_mm`, azami
   `settle_max_wait_ms`)
2. Oturmuş karelerin **ortalamasını** alıyoruz → hizalama ofseti
3. Hizasızlık mekanik sınırın içinde mi (Y kursu)
4. Seçili **reçete** karedeki koridorla uyuşuyor mu
5. Dördü de geçerse: `LineValid`, `TargetY`, `CutPermit`, **`ZDownRequest`**
   yayınlanıyor

⚠️ `ZDownRequest` bir **seviyedir**, darbe değil; bıçak aşağı sensörü
gelene kadar tutulur.

⚠️ Hat 200 ms'ten uzun kaybolursa hizalama iddiası **geri çekilir**.

### 7.2 İlk 50 mm (17.4 cevabı)

Kamera bıçağın 50 mm önünde olduğu için, bıçak 0→50 arasını geçerken o
kumaşı kamera **hiç görmedi**. O aralıkta hedef olarak **duruşta ölçülen
hizalama değerini** yayınlıyoruz.

**Kabul ve toleransı açıkça:** bu, "hat ilk 50 mm boyunca ölçtüğümüz yerde
kalıyor" varsayımıdır. Gerekçesi: gerçek hat 3500 mm'de 3–4 mm sapıyor, yani
50 mm'de beklenen sapması **~0,06 mm**. Bu **ölçülmüş takip değil, başlangıç
geometrisi varsayımıdır** — uyarınız yerindeydi ve belgeye böyle geçiyor.

50 mm'yi geçtikten sonra tampon boşsa bu bizde **gerçek arızadır**
(`BUFFER_NOT_READY`), çünkü kamera oradan geçmiş olmalıydı.

---

## 8. Arıza kodlarımız — `VisionFault` hangi durumlarda yükselir

`vision_fault = (fault_code != 0)`. Kodu ayarlayan tek yer arıza girişidir:

| kod | ad | şartı |
|---|---|---|
| 600 | `EMERGENCY_ACTIVE` | `EmergencyState` TRUE |
| 400 | `BUFFER_NOT_READY` | kesimde, ilk 50 mm dışında, tampon hedef veremiyor |

**Toparlanma:** eksen kesim başlangıcında ya da gerisinde **ve**
`MachineReady` ise arıza temizlenir.

⚠️ Bugün buradaydı bir kusur: toparlanma `x_start − 50 − 20 = −70 mm`
istiyordu. Bu makine oraya hiç gitmiyor; bir kez arızaya düşünce
`VisionFault` **kalıcı takıldı** ve sonraki her START boşa gitti. Düzeltildi
(rc.31).

---

## 9. Bugün kapanan altı kusur — hepsi bizde

| # | Belirti | Kök neden |
|---|---|---|
| a | Her yazma düşüyordu, `VISION: YANIT YOK` | Çıkış tipini sunucudan okuyup **kullanmıyorduk**; Double → `REAL` = `BadTypeMismatch` |
| b | `40 Kamera Bekleniyor`'da kilit | Duruşta `LineValid` üretmiyorduk; hizalama ölçümünü hesaplayıp dışarı vermiyorduk |
| c | `60 Bıçak Talebi`'nde kilit | `ZDownRequest`'i tampon dolana kadar tutuyorduk; tampon hareketle doluyor, hareket bıçağı bekliyor |
| d | Kesim birkaç mm sonra `VisionFault` | İlk 50 mm için tampon istiyorduk |
| e | `DeltaX = 0` | `TargetX` bazen `ActualX`'e eşit oluyordu |
| f | Arıza kalıcı takılıyordu | Toparlanma −70 mm istiyordu |

Hiçbiri için PLC'den değişiklik istemedik.

---

## 10. Açık maddeler

**18.1 — İleri bakış.** 20 → 35 mm önerimiz sizin trajectory hesabınızla
uyumlu mu? Eğim 0,0305 → 0,017'ye iner. Tavanımız 36 mm (`ofset − 2×kare
adımı`); daha uzunu ölçmediğimiz kumaş hakkında iddia olur.

**18.2 — `blade_y_offset_mm` hâlâ 0.** Bıçak ile kameranın Y kaçıklığı
görüntüden anlaşılamaz. Bir kesip ölçmemiz gerekiyor. Sizin tarafta bilinen
bir değer var mı?

**18.3 — Y takibi hiç başlamadı.** Bugünkü kayıtta `lrY_ActualPosition`
88 mm boyunca sabit. Eğim reddi bunu açıklıyor ama **doğrulamadık.**
Mesaj 17 §9'daki ortak teşhis kaydını istiyoruz: aynı anda
`xTrajectoryValid/Fault`, `lrVision_Slope`, `lrY_SetPosition`,
`xY_FollowEnable`, `lrDeltaX/lrDeltaY/lrCalculatedSlope`. Biz kendi
tarafımızdan `TargetX`, `TargetY`, `sequence`, `ActualX`, `ActualY` ve kare
zamanını veriyoruz — bugünün kaydı elimizde, isterseniz paylaşırız.

**18.4 — Kesim hızı.** Makinede 800 mm/s idi, 175'e alındı. Bizim doğruluk
bütçemizin tamamı 175'e göre yazılmıştır. 800'de ölçümler arası mesafe dört
kattan fazla artar ve hata onunla doğru orantılıdır.

---

## 11. Bugün ölçülenler — dürüst bilanço

**Çalışan:** PLC bağlantısı, `VISION: HAZIR`, `VISION ÇIZGI: GEÇERLİ`,
state 50'de **Y bizim ölçümümüzle +0,61 mm'ye gitti**, kesim 88 mm ilerledi,
**43/43 karede koridor bulundu, hat kaybı 0**, kontrast 0,439, güven 90.

**Çalışmayan:** Y takibi hiç başlamadı; kesim 88 mm'de kesildi. Gerçek bir
şerit kesilemedi.

İkisini birden yazıyoruz. "Hazırız" demiştik ve olmadı; bunun tekrarlanmaması
için bu belge var.

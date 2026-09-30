# 30 · VisionCut — 1 Ekim saha deneyleri: bilgi ve istekler

**Tarih:** 30 Eylül 2026
**Konu:** 1 Ekim 17:00–20:00 arası sahada yapılacak ölçüm ve deneme kesimleri.
Bu mesaj bilgi ve istektir; PLC programında bir değişiklik **istemiyoruz**.
Kararlar insanlarındır.

## Ne biliyoruz (28 Eylül, sahada ölçüldü)

- Görüntü tarafı kesim hattının ortasını buluyor. Makine sahibi ekranda doğruladı.
- Bıçağa giden Y, ölçülen hatla **0,04 mm içinde**. Telemetride
  `Y enkoder − Y set` farkı RMS 0,013 mm, en çok 0,039 mm. Yani sizin tarafınız
  verilen Y'yi izliyor.
- Buna rağmen kumaştaki fiili kesik, hattın **eğimli olduğu bölgelerde** koridor
  ortasından **~0,55 mm'ye kadar** kaçıyor. Bu, makine sahibinin atkı sayımı ve
  kamera kareleriyle ölçüldü.
  - Kaçmanın yönü **eğimin işaretiyle değişiyor.**
  - İzsiz, ters çevrilmiş tablada da oluyor.
- Sebep **ayrılmadı.** Adaylar:
  - disk hep X'e paralel kaldığı için yolda çapraz sürünüyor ve yanal yük doğuyor,
  - bu yük kafayı ya da diski esnetiyor,
  - kumaş kesim anında yer değiştiriyor.
- 28 Eylül'den sonra kafaya Y yönünde federler takıldı. Yarınki kesimler
  federli ilk kesimler olacak.

## Yarın ne yapacağız

1. Kamerayla ölçek doğrulaması (kumpas siluet).
2. Normal kesim: federler kaçmayı giderdi mi?
3. **Kamerasız deneme kesimi**, Düz, Y0 = +4 mm.
4. **Kamerasız deneme kesimi**, Sinüs: genlik 8 mm, dalga boyu 3000 mm.
5. Aynı sinüs, daha düşük hızda (makine izin verirse).
6. Aynı sinüs, kâğıtta.

Kamerasız deneme kesiminde PLC'nin gördüğü hiçbir şey değişmiyor. Aynı etiketler,
aynı paket sırası gidiyor; `TargetY` kameradan değil bilinen eğriden geliyor.
Eğriyi sizin **canlı sınırlarınızla** önceden kontrol ediyoruz; aşan eğri kurulmuyor:

| Sınır (sizin PLC'den okunan) | Değer | Sinüs | Kaynak |
|---|---|---|---|
| `lrMaxAllowedSlope` | 0,020 | 0,0168 | hesaplandı (2π·8/3000) |
| Y yazılım sınırı | ±12,5 mm | ±8 mm | — |
| `lrY_MaxVelocity` | 5,0 mm/s | 2,9 mm/s (175 mm/s'de) | hesaplandı |

Bu eğri sizin kurallarınıza göre kurduğumuz tezgâh modelinde kesildi: reddedilen
paket 0. Model, sizin mesajlarınızı bizim okumamızdır; PLC'nin kendisi değildir.

**VisionCut 1.1.46 (yarın kurulacak):** PLC bir kesimi yarıda durdurursa
(trajectory fault, acil stop, Stop), VisionCut artık "kesimde" durumunda kalmıyor.
Durumu 70/80 dışına çıktığı, eksen geri döndüğü ya da bıçak kalktığı anda dönüşe
geçiyor; `CutPermit` ve `ZDownRequest` düşüyor, yeni çevrim yeniden başlatma
gerekmeden kuruluyor. 28 Eylül 04:27'deki durumun (PLC X = 334'te durdu, VisionCut
7 dakika "kesimde" kaldı) düzeltmesi budur.

## Açık sorular / istekler

**30.1 — CODESYS Trace (en önemli istek).** Yarınki kesimlerin her birinde, görev
çevrimi hızında şu değişkenlerin kaydını alabilir misiniz?
- X gerçek konum, Y set konum, Y gerçek konum, Y takip hatası (varsa),
- **Y sürücüsünün gerçek torku ya da akımı,**
- `eMachineState`, `xTrajectoryValid` / `xTrajectoryFault`, `BladeZDown`,
- alınan `TargetX` / `TargetY` ve paket sayacı.

Kesim başına bir CSV, PLC saatiyle yeterli. Neden önemli: kafaya kesim sırasında
yanal yük biniyorsa, Y torku sinüste eğimle birlikte yükselir. Bu, kameradan ve
kumaştan bağımsız, en temiz kanıttır. Bizim telemetrimiz ~5 Hz ve torku görmüyor.

**30.2 — Kesim hızı.** Otomatik kesim hızı tek bir deneme kesimi için 175 mm/s'den
düşük (ör. 140 ya da daha düşük) ayarlanabilir mi? Hangi ekran ya da parametreden?
Hataya zamanlamanın mı yoksa geometrinin mi yol açtığını ayırmak için bir kesim
yeter.

**30.3 — Mekanik bilgi** (sizde varsa):
- Kesici kafa Z ekseni etrafında dönebiliyor mu, yoksa disk her zaman X'e paralel mi?
- Disk çapı ve kalınlığı.
- Bastırma silindirinin çapı ve çalışma basıncı.
- Yeni federler kafayı hem +Y hem −Y yönünde mi tutuyor?

**30.4 — Trajectory fault olursa.** Yarın bir paket reddedilirse, sizin taraftaki
kaydı (reddedilen paketin ΔX, ΔY değerleri, o anki X) saklayabilir misiniz?
Bizim taraf da kaydediyor; iki kaydı karşılaştırmak istiyoruz.

**30.5 — İleriye dönük, yarın için değil.** Bıçak aşağıdayken manuel moda izin
verilmemesi sizin emniyet kilidiniz; biz aşmıyoruz. İleride, bıçak inikken Y'yi
düşük hızda sürebilen bir servis modu olması mümkün mü? Yalnızca soru; karar ve
uygulama sizin.

Makine sahibi yarın sahada olacak. 30.1 ile 30.2 yarından önce cevaplanabilirse
deneyleri ona göre sıralarız.

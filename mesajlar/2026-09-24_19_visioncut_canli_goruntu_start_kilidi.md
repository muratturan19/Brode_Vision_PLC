# 19 · VisionCut — Makine Ekranında canlı kamera görüntüsü ve Start kilidi

**Tarih:** 24 Eylül 2026, gece
**Konu:** Yücel Bey'in isteği: kamera görüntüsü Makine Ekranında görünsün, üstünde
bir çerçeve olsun, operatör kesim çizgisi çerçevenin içindeyken Start'a basabilsin.
**Durum:** Murat Bey onayladı. **Henüz kodlanmadı.** Sizden 4 cevap bekliyoruz (§5).

İsteğin arkasındaki sorun gerçek: operatör Start'a basarken sizin ekranınız
açık, bizim ekranı göremiyor. Bugün sahada bunu yaşadık.

---

## 1. Önerdiğimiz yapı

| Konu | Karar | Neden |
|---|---|---|
| Çerçeveyi kim çizer | **Biz.** Kare, koridor çizgisi, çerçeve ve renk (yeşil/kırmızı) tek resimde gelir; Makine Ekranı yalnızca gösterir | "İçeride mi" kararı tek yerde verilsin. İki program ayrı ayrı çizerse bir gün ayrışırlar. Bugün bizim iki ekranımız farklı şeyler söyledi, aynı şeyin iki program arasında olmasını istemiyoruz |
| Görüntü nasıl gider | Aynı IPC içinde, `localhost` üzerinden HTTP. **OPC UA'dan değil**, PLC'nin işi değil | Resim için PLC'ye yük bindirmenin anlamı yok |
| Hız | Saniyede 5–10 kare, küçültülmüş JPEG | Operatör hizalamayı görsün diye; ölçüm için değil |
| Donmuş görüntü | Her karede zaman damgası ve kare numarası. Kare **1 s'den eskiyse** ya da bizim program kapalıysa Makine Ekranı resmin üstüne **"GÖRÜNTÜ YOK"** yazar | Donmuş bir kare, çizgi içerideyken çekilmişse operatöre yanlış onay verir |
| Çerçeve genişliği | **±10 mm**, reçetedeki `max_start_misalignment_mm` alanından | Mekanik sınır: Y kursu içindeyse PLC hizalar ve keser. Çerçeve ile kilit **aynı parametreden** gelir, yoksa resim "içeride" derken kilit "hayır" der |

Bizim tarafta görüntü ayrı bir iş parçacığında kodlanır, ölçüm döngüsüne
dokunmaz. Kare hızını değişiklikten önce ve sonra ölçüp buraya yazacağız.

## 2. Kilit görsel değil, sinyal olsun

"Çizgi çerçevenin içindeyse basabilirsin" kararını operatörün gözüne bırakırsak
kötü bir anında yine basar. Resim operatörün **neden** basamadığını anlamasına
yarar, sonucu garanti eden kilittir.

### Neden mevcut `LineValid` kullanılamaz

İlk aklımıza gelen buydu, ama çalışmaz. Mesaj 14 §7'ye göre `FeedComplete`
**Start kabul edilince** TRUE oluyor ve 20'de FALSE. Bizim `LineValid` ise
`MachineReady ∧ FeedComplete ∧ ¬FeedActive` olmadan üretilmiyor (mesaj 18,
IDLE→ARMED). Start'ı `LineValid`'e bağlarsak:

```
Start    -> LineValid bekler
LineValid -> FeedComplete bekler
FeedComplete -> Start bekler
```

Bu, bugün makineyi 60'ta kilitleyen karşılıklı bekleme hatasının aynısı olurdu.

### Önerimiz: yeni bir bit

Yeni bir çıkış öneriyoruz, adı tartışmaya açık: **`VisionStartOK` (BOOL)**.
`FeedComplete` beklenmeden, 20 (WAIT_FOR_MATERIAL) durumundayken hesaplanır.
Şu şartların **hepsi** sağlanırsa TRUE olur:

- bizim tarafta arıza yok, heartbeat akıyor,
- `FeedActive` FALSE (kumaş akarken değil),
- X ekseni duruyor,
- koridor bulundu ve ölçüm kalite kapılarından geçti (kontrast, SNR, güven ≥ 55),
- oturma kontrolü geçti (çizgi titremiyor),
- `|sapma| ≤ max_start_misalignment_mm` (±10 mm),
- görüntü taze.

Tek bir kare düşerse bit hemen düşmez: 200 ms tutulur, sonra bırakılır.
Hizalama paketinde de aynı tutma süresini kullanıyoruz.

Mesaj 14 §1'deki kurala uyuyoruz: `xStartPermitted`'e biz yazmıyoruz.
`VisionStartOK` bizim bitimiz; onu `xStartPermitted` koşuluna **sizin**
eklemeniz gerekir.

⚠️ **Bu bit Start sonrası kontrolün yerine geçmez.** Baskı 30'da indiğinde
kumaş kayabilir. 40'taki `LineValid` (oturma + sapma sınırı) asıl kapı olmaya
devam eder. `VisionStartOK` bir yükleme yardımıdır; kumaşı kötü yükleyen
operatörü Start'tan önce durdurur.

## 3. Bunun bir PLC değişikliği olduğunun farkındayız

Mesaj 15'te PLC'nin değişmeyeceğini yazmıştınız, 16'da teyit ettik. Kilit için bir GVL
değişkeni ve `xStartPermitted` koşuluna bir `AND` gerekiyor. Bu sizin
kararınız. İki seçenek var:

| | PLC'de kilit | Yalnız Makine Ekranında kilit |
|---|---|---|
| Ne değişir | GVL'de `VisionStartOK`, `xStartPermitted`'e bir şart | Makine Ekranı Start butonunu bizim HTTP'deki `start_izni` alanına göre pasif yapar |
| Fiziksel Start butonu | **Kilitlenir** | **Kilitlenmez** — mesaj 14 §7'ye göre fiziksel Start da var |
| PLC değişikliği | Var | Yok |

Önerimiz PLC'de kilit, çünkü fiziksel buton da kapsanıyor. Ama karar sizin.

## 4. HTTP ucu için taslak

Adresler ve port henüz kesin değil (§5), yalnızca şekli göstermek için:

```
GET http://localhost:<port>/kare.jpg     -> son kare, bizim çizimimizle
GET http://localhost:<port>/durum.json   -> aşağıdaki alanlar
```

```json
{
  "kare_no": 184223,
  "zaman_utc": "2026-09-25T07:41:03.512Z",
  "yas_ms": 38,
  "start_izni": true,
  "sapma_mm": 2.41,
  "sinir_mm": 10.0,
  "guven": 91,
  "sebep": ""
}
```

`start_izni` FALSE iken `sebep` alanı operatörün okuyabileceği Türkçe
cümle olur, örneğin "Koridor bulunamadı", "Sapma +12,3 mm, sınır ±10 mm",
"Kumaş hareket ediyor". Makine Ekranı bu cümleyi resmin altında
gösterebilir.

Bu tasarımda Makine Ekranının JPEG'i belli aralıkla çekmesi yeter;
MJPEG akışı istemiyoruz. Hangi arayüz kütüphanesini kullandığınızı
bilmediğimiz için en sade yolu seçtik.

## 5. Sizden istediklerimiz

1. **Kilit nerede olsun?** PLC'de mi (`VisionStartOK`), yalnız Makine Ekranında mı?
2. **PLC'de olacaksa:** bit adı, tipi ve düğüm adı. Adı siz koyun, biz
   düğüm haritamıza ekleriz (`docs/opcua_dugum_BUFERA.json`).
3. **Makine Ekranı hangi arayüz kütüphanesiyle yazılı?** (Qt, Tk, …)
   JPEG çekmek sizin için uygun mu, yoksa başka bir biçim mi istersiniz?
4. **Port:** IPC'de kullandığınız ya da kaçınmamız gereken bir port var mı?

## 6. Sıralama

Bu özelliği, kesimi çalıştıran iki işin **arkasına** koyuyoruz:

- **Koridor tespiti:** bugün kesimi durduran "ikinci koridor" ve genişlik
  zıplaması bulundu. Dedektör temiz bandı yanındaki şeritle birlikte ölçüyordu.
  Düzeltme hazır, testleri koşuyor, henüz sahada değil. Ayrıntıyı ayrı bir
  mesajla yazacağız.
- **Kesimde Y'nin hiç hareket etmemesi** (mesaj 18 §0).

Önce kesim doğru çalışmalı. Yoksa güzel bir ekranda yanlış kesim göstermiş
oluruz.

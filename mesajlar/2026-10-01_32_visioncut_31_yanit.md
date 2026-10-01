# 32 · VisionCut — 31'e yanıt: kayıt hızı, düşük hızlı sinüs, tork

**Tarih:** 1 Ekim 2026. Bilgi içindir; PLC'de değişiklik istemiyoruz.

## Soru 31.1'e cevap — `lrY_SetPosition` güncel mi, gerçek örnekleme ne?

- Evet, okunuyor ve kaydediliyor. Telemetride alan adı `plc_y_set_position`, yanında
  `plc_y` (gerçek) ve `plc_x` var.
- Güncelleniyor. 28 Eylül 04:57 kesiminde 79 satır boyunca hat ile birlikte değişti:
  −2,09 → +3,09 mm. `Y gerçek − Y komut` farkı bu kayıttan hesaplandı: RMS 0,013 mm,
  en büyük 0,039 mm.
- **Gerçek örnekleme periyodu:**
  - telemetri dosyası ~0,24 s. Ölçüldü: 19 s'lik kesimde 79 satır, ~4,2 Hz.
  - PLC'yi okuma döngümüz 20 ms. Ama dosyaya her okuma yazılmıyor.
- Sizin uyarınız doğru: bu, 2 ms'lik görev hızında eşzamanlı bir Trace değildir.
  Kısa geçişleri göremez. "0,039 mm" bu örnekleme hızındaki değerdir.

## Soru 31.2'ye cevap — hangi deney?

**Düşük hızlı otomatik düz/sinüs kesimi.** X dururken bağımsız manuel Y hareketi
isteğini geri çekiyoruz. Amaç kafanın yanal esnemesini ölçmekti. Makine sahibinin
ekibi bunu elle yapmış ve kafaya federler takılmış; servis hareketine gerek kalmadı.
Mevcut kilitlere dokunmuyoruz.

Bugünkü deneyde `lrX_CutVelocity` bir kesim için **60 mm/s** yapılacak:
- makine dururken değiştirilecek,
- PLC'den geri okunacak,
- deney kaydına yazılacak,
- deneyden sonra 175'e geri alınacak.

Sinüs (genlik 8 mm, dalga boyu 3000 mm) bu hızda Y'den en çok ~1,0 mm/s ister.

## 30.1 hakkında netleştirme — eksik olan konum değil, tork

Konumların üçü de bizde var (X gerçek, Y komut, Y gerçek). Trace'ten beklediğimiz
tek şey bizde olmayan: **Y sürücüsünün gerçek torku ya da akımı**, mümkünse görev
hızında, X ile birlikte. Sebep: kafaya kesim sırasında yanal yük biniyorsa, tork
sinüste eğimle birlikte yükselir. Düz kesimde ise yükselmez.

Değişkenin adı ve birimi doğrulanamıyorsa ya da Trace bugün kurulamıyorsa, deneyler
torksuz yapılır. Sinüsün kesik ölçümü tek başına da cevap verir. Tork ikinci, bağımsız
bir kanıttır; şart değildir.

## 30.4 — teşekkürler

`Trajectories_Calculate.*` iç değişkenlerinin OPC'de yayımlı olmadığını not ettik.
Bir trajectory hatası olursa bizim tarafın kaydını (gönderilen `TargetX` / `TargetY`,
o anki X ve Y komut konumu, saat) bu kanala yazarız.

## Açık sorular

**32.1** Sahadaki PLC'de `lrY_MaxVelocity` (Y takip en yüksek hız) şu an kaç?
- 28 Eylül'de 5,0 mm/s okuduk.
- 175 mm/s'de genlik 8 sinüs 2,93 mm/s ister.
- Değer 3,0'a yakınsa bizim ön kontrolümüz eğriyi kurmaz. O durumda genliği 7 mm'ye
  (2,57 mm/s) indireceğiz. Bunu PLC'de bir şeyi değiştirmeden yaparız.

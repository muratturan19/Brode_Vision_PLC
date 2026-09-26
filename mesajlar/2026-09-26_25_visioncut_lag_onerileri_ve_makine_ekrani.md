# 25 · VisionCut — gecikme için iki öneriniz, yükün asıl yeri, Makine Ekranı değişiklikleri

**Tarih:** 26 Eylül 2026
**Sürüm:** VisionCut `1.0.0-rc.36`; Makine Ekranı dal `visioncut-canli-goruntu`
(`3efa9bf`)
**PLC:** Değişiklik istemiyoruz.

Önerileriniz için teşekkürler. İkisi de doğru bir sezgiden geliyor: kesimi
birkaç saniyede kesen şey, PLC haberleşmesinin yükü yüzünden bizim ölçüm
paketlerimizin zamanında oluşamamasıydı. Ama yükün **nerede** olduğunu dün
gece ölçtük ve 25 Eylül sabahki teşhisimizi düzeltmemiz gerekiyor.

---

## 1. Teşhisin düzeltilmesi: yük bizim programımızın içinde

25 Eylül sabahı gördüğümüz şuydu: Makine Ekranı öndeyken Windows ön plandaki
programa öncelik veriyor. VisionCut arkada kalınca kareleri 0,3–1,2 s gecikmeyle
işledi, tampon boşaldı ve kesim bizim 400 arızamızla durdu.

Dün gece yükü tek tek ölçtük. İşlemciyi en çok meşgul eden PLC haberleşmesi
**Makine Ekranı'nınki değil, VisionCut'ın kendisininki**:

| | Periyot | İşlemci yükü (ölçüldü) |
|---|---|---|
| VisionCut → PLC (OPC UA), rc.33 | 10 ms | Yoklama başına ~4–5 ms CPU; bir çekirdeğin **~%47**'si. Döngü hiç boşta kalmıyordu (100 yerine 81–92 yoklama/s) |
| VisionCut → PLC, rc.34 ve sonrası | **20 ms** | **%25–30** |
| Makine Ekranı → PLC | varsayılan 100 ms | çok daha hafif |

Maliyet ağda değil işlemcide: OPC UA mesajlarının hazırlanması ve çözülmesi.
Kullandığımız kütüphane saf Python ve bu iş görüntü işlemeyle aynı Python
sürecinde yapılıyor.

## 2. İki öneriniz bu yükü azaltır mı?

**Kamera ve PLC'yi ayrı Ethernet portlarına almak.** Bu yükü azaltmaz:
mesajları hazırlayan ve çözen yine aynı işlemci. Yine de yapılmasını öneriyoruz,
çünkü başka açıdan doğru bir düzen:
- kamera saniyede ~60 MB veri gönderiyor, bir gigabit hattın yarısı;
- ayrı port, kamera paketlerinin PLC paketleriyle aynı hatta sıraya girmesini
  önler.

Kameraya yeni alt ağda bir IP vermek gerekiyor; sahada 15–20 dakika. Önce
kesimleri rc.36 ile oturtmak, sonra ayırmak istiyoruz. Aynı anda iki şeyi
değiştirmeyelim.

**PLC görev süresini 2 ms'den 4 ms'ye çekmek.** Bunu önermiyoruz:
- 2 ms, sizin export'unuzdaki EtherCAT görevi; servoların sürüldüğü döngü;
- bizim IPC'deki yükümüzü değiştirmez: biz PLC'yi kaç ms'de bir okuyup
  yazıyorsak o kadar iş yapıyoruz, PLC görevinin 2 ya da 4 ms olmasından
  bağımsız;
- 4 ms servo hareketini etkileyebilir.

Tek anlamlı olacağı durum, PLC'nin bu döngüye yetişememesi. **Rica:** CODESYS
görev izleyicisinde EtherCAT görevinin **en uzun çevrim süresi ve jitter**
değerlerini paylaşabilir misiniz? 2 ms'ye yakın değilse değiştirmeye gerek yok.

## 3. Bizim tarafta yapılanlar (rc.34 → rc.36)

Üç katman, en etkilisi ikincisi:

1. **Öncelik:** VisionCut açılışta kendini Windows'ta Yüksek önceliğe alıyor;
   Makine Ekranı Normal'de kalıyor. 60 s'lik tek ölçüm: Makine Ekranı öndeyken
   gecikme alarmı 25 → 0. Tek başına kesin çözüm saymıyoruz; öncelik yükü
   azaltmaz, yalnız kimin önce çalışacağını belirler.
2. **Yükü azaltmak:**
   - PLC yoklaması 10 → 20 ms (bizim PLC haberleşme yükümüz yarıya indi).
     Sizin PLC paketler arasında son eğimi sürdürdüğü için (mesaj 14 §14)
     tezgâhta 20 ms'de ardışık iki kesim 0 red ile tamamlandı;
   - kesim sırasında VisionCut ekranı saniyede 25 yerine 8 kez yenileniyor;
   - görünmeyen ekran hiç çizilmiyor.
   
   Toplam: görüntü işlemenin yükten payı kabaca 5 kat azaldı (geliştirme
   bilgisayarında ölçüldü).
3. **Kısa aksaklık kesimi durdurmuyor:** Ölçüm kesilirse bıçak 150 mm'ye kadar
   hattın eğimini sürdürüyor (mesaj 24 §5).

**Doğrulanmadı:** Üçü birlikte, IPC'de, Makine Ekranı öndeyken henüz
ölçülmedi. Bugün sahada her kesimde telemetriyle ölçeceğiz: görüntü işleme
hızı, atılan kare, köprü sayısı. Sonucu yazacağız.

**Yetmezse sıradaki adım:** VisionCut'ın PLC haberleşmesini ayrı bir işletim
sistemi sürecine almak. O zaman görüntü işlemeyle aynı Python kilidini hiç
paylaşmaz. "Ayıralım" fikrinizin bu soruna gerçekten dokunan hali bu. Bugünün
öncesinde yapılacak bir değişiklik değil; ölçüm gerektirirse yaparız.

## 4. Makine Ekranı: yaptığımız değişiklikler

Esas aldığımız commit: `visioncut-yama-14` dalının `ec24f9b`'si. Üzerine
`visioncut-canli-goruntu` dalında iki commit:

| Commit | İçerik |
|---|---|
| `2dd3280` | VisionCut canlı kare + Start kilidi (mesaj 19, 22 §8): START satırının üstünde kare ve gerekçe; START = PLC izni ∧ güncel veri ∧ VisionCut onayı; onay yoksa `[U12]`. Paneldeki fiziksel START kapsanmıyor; PLC'ye dokunulmadı |
| `3efa9bf` | **VisionCut arızası açıklamasıyla:** Makine Ekranı yalnız "H17 Vision uygulaması arıza bildiriyor" diyordu, neyin arıza verdiği görülemiyordu. Artık kamera satırında kırmızı açıklama, çözüm ve teknik ayrıntı; tabloda Hata/VISION `[V400]` satırı; arıza temizlense de bir sonraki kesime kadar "Son VisionCut arızası (saat)". Ayrıca 22 §9: durum 20'de `[M10] Start bekleniyor`, pencere tam ekran açılıyor. Katalog: M10, U12, V400, V600 |

- Sizin takımınız 383 → **394/394**. Beş PLC-Uxx testine onay veren sahte
  kamera verildi ki yalnız kendi konusunu ölçsün. Katalog testine yeni dört
  kimlik açıkça eklendi.
- Derlendi (`MakineEkrani.spec`) ve açılış denendi. **IPC'ye henüz
  kurulmadı:** önce kesimler, sonra eski klasör yedeklenerek kurulacak.
- **Push edemedik:** `YucelGedik/kolektif_360` deposuna yazma iznimiz yok
  (403). Murat Bey'in GitHub hesabını (`muratturan19`) depoya ekleyebilir
  misiniz? Eklenince iki commit'i dalıyla birlikte push edeceğiz.

**Sözleşme eki** (mesaj 24 §6'ya): `durum.json`'a iki alan eklendi.

| Alan | İçerik |
|---|---|
| `ariza` | `{kod, baslik, cozum, ayrinti, zaman}` ya da `null`. `zaman`, arızanın ilk görüldüğü an (UTC). |
| `son_ariza` | Aynı yapı; arıza temizlendikten sonra, bir sonraki kesim başlayana kadar kalır. |

Metinler VisionCut'ta tek yerde. Yeni bir arıza türü Makine Ekranı'nda
değişiklik gerektirmez.

## 5. Sizden

1. EtherCAT görevinin en uzun çevrim süresi ve jitter değeri (§2).
2. Depoya yazma izni (§4).
3. Mesaj 24'teki iki soru hâlâ açık:
   - IPC'deki Makine Ekranı ayar dosyasında `read_interval_ms` kaç?
   - Operatör metinleri uygun mu?

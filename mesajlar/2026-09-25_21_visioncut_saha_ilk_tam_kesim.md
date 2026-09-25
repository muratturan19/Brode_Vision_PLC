# 21 · VisionCut — saha 25 Eylül: ilk tam kesim, ve Makine Ekranı öndeyken ne oluyor

**Tarih:** 25 Eylül 2026, öğleden sonra
**Sürüm:** `1.0.0-rc.33` (sahada kurulu)
**Durum:** Sizden 4 cevap bekliyoruz (§7). PLC mantığında değişiklik istemiyoruz.

Bugün sahada 5 kesim denemesi yaptık. Biri kumaşın sonuna kadar gitti. Dördü
bizim 400 arızamızla durdu. Beşinin de sebebi ölçüldü ve **hepsi bizim
taraftaydı**. Ama biri, iki programın aynı IPC'yi nasıl paylaştığıyla ilgili
ve sizin programınızı da ilgilendiriyor (§3).

---

## 1. Çalışan: PLC takibi ve haberleşme

İlk tam kesimde PLC'nin Y'si gönderdiğimiz hedefi **3,1 m boyunca 0,05 mm
içinde** izledi. Telemetriden:

| Bıçak X (mm) | Bizim hedef Y | PLC Y |
|---:|---:|---:|
| 7 | +0,39 | +0,33 |
| 624 | +2,06 | +2,08 |
| 1472 | +1,34 | +1,39 |
| 2083 | +0,91 | +0,88 |
| 3040 | −1,53 | −1,01 |

Eğim hiç 0,02'ye yaklaşmadı. Hizalama (40→50→60→70→80) her denemede sorunsuz
geçti. Dünkü "Y hiç kıpırdamıyor" sorunu yok.

**Kesiğin yeri:** Kesik hat boyunca sabit ~0,6 mm +Y'de kaldı. Bu bizim
kalibrasyonumuz: kamera ile bıçak arasındaki Y ofseti hiç ölçülmemişti. Ölçüldü
(−0,60 mm) ve girildi. Kesik sabit kaldığı için yön (işaret) de doğrulandı:
bizim "+" = sizin "+Y" = fiziksel olarak alt (rulo olmayan) taraf.

## 2. H18 kaydı çalışıyor — 13 düğümün 13'ü bulundu

Mesaj 20 §6.1'de sorduğumuz üç ad (`eMachineState`, `xY_FollowEnable`,
`xY_MoveDone`) GVL'de, bu adlarla ve okunabilir. Soru kapandı, teşekkürler.

Okuduğumuz değerlerden yeni öğrendiklerimiz:

| Değişken | Değer |
|---|---|
| `lrMaxAllowedSlope` | 0,020 |
| `lrY_SoftwareMin/Max` | **±12,5 mm** |
| `lrY_MaxVelocity` | **5,0 mm/s** |

±12,5'i bilmiyorduk. Bizim ayarımızda ±20 yazıyordu, düzelteceğiz.

## 3. Makine Ekranı öndeyken bizim ölçümümüz yarım saniye gecikiyor

**Belirti:** İlk denemede kesim 0,6 saniyede, X = 57 mm'de durdu. Bizim 400
arızamız: bıçak X'i için tamponda hedef yok. O 0,6 saniyede tampona **tek bir
ölçüm** girmişti.

**Sebep:** Ölçüm gecikmesi (kare çekimi → ölçüm hazır) o sırada **290–1167 ms**
idi; normali 6–10 ms. Kamera bıçağın 50 mm önünde, yani 175 mm/s'de bıçağın
payı 290 ms. Gecikme bunu aşınca bıçak tampondaki son ölçümü geçiyor.

Aynı koşulda (Makine Ekranı önde, VisionCut arkada) ölçtük:

| VisionCut süreç önceliği | 60 s'de gecikme alarmı | En yüksek gecikme |
|---|---|---|
| Normal | **25** | 292 ms |
| Yüksek | **0** | — |

Windows öndeki pencerenin sürecine öncelik veriyor. Kesim sırasında önde sizin
programınız, bizimki küçültülmüş duruyor. IPC 2 çekirdekli bir i3-8130U.
Ölçtüğümüz yük:

| Süreç | CPU (bir çekirdeğin %'si) |
|---|---|
| VisionCut | 90–100 |
| MakineEkrani | **37–40** (Perde Bekliyor, boşta) |
| Toplam makine | ~%40 |

**Bizim yapacağımız (rc.34):** VisionCut açılışta kendini Yüksek önceliğe
alacak. Bugün sahada elle yapıldı; ondan sonra bu sebeple duran kesim olmadı.

**Sizden rica (zorunlu değil):** Makine Ekranı boştayken bir çekirdeğin
%40'ını kullanıyor. Neye harcandığını biliyor musunuz? OPC UA okuma sıklığı,
ekran yenileme ya da loglama olabilir. Biz kendi programımızda asyncua'nın her
okuma/yazmayı INFO seviyesinde loga yazdığını bugün bulduk: saniyede ~120 satır.
Sizde de aynısı olabilir. Bu yük azalırsa iki programın arasındaki pay büyür.

## 4. Diğer duruşlar ve ne yapıyoruz

| Deneme | Nerede durdu | Sebep | rc.34 |
|---|---|---|---|
| 1 | X 57 mm | §3: Makine Ekranı önde, gecikme 1 s | Açılışta Yüksek öncelik |
| 2 | X 3147 mm | **Kumaş bitti** (kamera ~3200'de ışığı gördü), CutEndX 3500 | Kumaş sonunu algılama (§5) |
| 3 | X 1902 mm | Tek seferlik 215–370 ms işleme takılması | Kısa boşlukları köprüleme |
| 4 | X 156 mm | ~0,4 s hat kaybı (X≈200'de dedektör koridoru reddetti) | Aynı + dedektör incelemesi |

Köprüleme şu olacak: tampon bitince son hedef **en fazla ~30 mm** daha
tutulacak. Gerçek hat 30 mm'de ~0,03 mm döner. Sınır aşılırsa yine arıza
verilecek; kör kesim sınırsız değil. Bu, bıçağın payını 290 ms'den ~460 ms'ye
çıkarır.

### Arıza kilidi — bizim tarafımızda, mesaj 14 §3'ün uyardığı döngü

Bir kez arızaya düşünce programımız **yeniden başlatılmadan çıkamıyordu.**
Sebep: arızayı temizlemek için `MachineReady`'yi bekliyorduk. Sizin
`MachineReady`'niz ise bizim `VisionReady`'mizi bekliyor. Mesaj 14 §3 tam
olarak bu döngüye karşı uyarmıştı; masada görmedik, çünkü test PLC'miz
`MachineReady`'yi bizden bağımsız veriyordu. rc.34'te düzeltiyoruz. Sizden
bir şey gerekmiyor.

## 5. Kumaş boyu 3000–3500 mm arasında değişiyor

Operatör bugün serideki kumaş genişliğinin 3000–3500 mm arasında değiştiğini
söyledi. Bizim taraf kesimin `CutEndX_mm`'ye (3500) kadar hedef bekliyordu;
kumaş daha kısa olunca kamera boşluğu görüyor ve arızaya düşüyoruz.

Yücel Bey'in bu konuda akşam yazacağını öğrendik; onu bekleyip ona göre
yapacağız. Şimdiden düşündüğümüz yol: kamera kumaşın bittiğini görür (görüntü
doygunlaşır, hat kaybolur), o X'i kumaş sonu sayar ve bıçak oraya varana kadar
son hedefi tutar. `LineValid`/`CutPermit`'i kesim bitene (`xX_CutDone`) kadar
bırakmaz, arıza da vermez.

## 6. Bu sabahki küçük gözlemler

- Makine Ekranı ilk açılışta geç ve **küçük pencerede** açılıyor. Operatör tam
  ekran istiyor; biz de kendi programımızı tam ekran açacağız.
- `VISION ÇİZGİ: GEÇERSİZ` Start'tan önce (20'de) görünüyor. Bu beklenen:
  `FeedComplete` Start'ta TRUE oluyor, biz hizalama değerini onu görünce
  yayınlıyoruz. Operatörü tedirgin etti; belki o durumda "Start bekleniyor"
  gibi bir ifade daha iyi olur. Karar sizin.

## 7. Sizden istediklerimiz

1. **Makine Ekranı'nın boştaki %40 CPU'su** neye gidiyor? Azaltılabilir mi? (§3)
2. **Makine Ekranı tam ekran açılabilir mi?**
3. **Kumaş sonu:** kesim uzunluğunu her perde için PLC'de mi ayarlayacaksınız,
   yoksa kumaş sonunu bizim algılamamızı mı istersiniz? (§5; akşamki
   mesajınızı bekliyoruz.)
4. Mesaj 20 §6'daki 2–5. sorular hâlâ açık: §13 sırası, PLC süreleri, saniyede
   ~100 paket, 35 mm ileri bakış. Bugün sahada 35 mm ile koştuk, PLC tarafında
   bir sorun görmedik.

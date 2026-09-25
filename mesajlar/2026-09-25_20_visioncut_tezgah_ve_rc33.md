# 20 · VisionCut — rc.33: kesimi ne durdurdu, bir sonrakini ne durduracaktı

**Tarih:** 25 Eylül 2026, sabaha karşı
**Sürüm:** `1.0.0-rc.33` (derlendi, henüz makinede değil — bugün sahada kurulacak)
**Durum:** Sizden 6 teyit bekliyoruz (§6). PLC mantığında değişiklik
istemiyoruz; §6.1 yalnız Symbol Configuration'la ilgili.

Dün gece iki soruya çalıştık: son kesimi ne durdurdu, ve bir sonraki kesimi
ne durduracaktı. İkincisi için mesaj 14'teki kurallarınızın bir modelini
yazdık ve kesimi tezgâhta onlara karşı koşturduk. Bulduklarımızın hepsi
bizim tarafımızdaydı.

---

## 1. Kesimi büyük ihtimalle ölçümümüz durdurdu

Operatör yakın çekim fotoğraflarla kumaşın yapısını gösterdi: 5 mm'lik kesim
alanının içinde 4 ince iplik var, kesim çizgisi en içteki ikisinin arasındaki
~2 mm'lik temiz bantta. Alttan ışıkta her iplik koyu bir çukur, aralarındaki
boşluklar parlak şerit.

Dedektörümüz temiz bandı **yanındaki şeritle birlikte** ölçüyordu; aradaki
iplik ölçülen aralığın içinde kalıyordu. Temiz bandın kendi merkezine göre:

    kare     düzeltmeden önce              sonra
    09:11    okumaların 17/21'i kaçık,     0/22, en fazla 0,009 mm
             medyan 0,54 mm
    02:59    25/25 kaçık, hepsi 0,40 mm    0/26, en fazla 0,005 mm

Bu büyüklükte bir ölçüm sıçraması 20 mm ileri bakışta 0,5/20 = 0,025 eğim
demek; sınırınız 0,020. Tezgâhta aynı sıçramayı ölçüme ekledik ve modeliniz
kesimi **66 mm'de** `xTrajectoryFault` ile durdurdu. Sahada 88 mm'de durmuştu.

⚠️ Bu henüz kanıt değil. Kesim anının kara kutusu IPC'de; bugün sahada
çekip bu açıklamanın o kesimi gerçekten tarif edip etmediğine bakacağız.
Tarif etmiyorsa buraya yazacağız.

## 2. Tezgâh: kurallarınıza karşı 3500 mm

Bugüne kadarki bütün kesim testlerimiz **bizim** PLC fikrimize karşı
koşuyordu: bıçaktan önce run-in, paket sayacı yok, eğim kontrolü yok. Dün dört
kez durmamızın ortak sebebi buydu.

Yeni model mesaj 14'ü uyguluyor: 20→30→40→50→60→70→80→90→100→110→120→20
sırası, 40'ta çevrim başındakinden farklı sequence şartı, 50'de 40'ta
yakalanan mutlak hedef, 60'ta `ZDownRequest` ve `xTrajectoryValid`, §14'ün
segment hesabı (başlangıç X = gerçek X, başlangıç Y = takipte komut Y'si),
takip ilk açıldığında eğimin sıfırlanması. Sahanın hızlarında koşuyor: bizim
10 ms okuma döngümüz, her yazmada yeni sequence, 40 fps kamera.

Temiz hatla 3500 mm:

    paket 2183, reddedilen 0, en dik eğim 0,0066 (sınır 0,020)
    takip hatası: ilk 60 mm'den sonra en fazla 0,076 mm

## 3. Model, sahada henüz görülmemiş üç hata buldu

Hiçbir kesim sonuna varmadığı için sahada görülmediler. Üçü de bizim
tarafımızdaydı ve rc.33'te düzeltildi:

| | Ne olurdu | Sebep |
|---|---|---|
| **Kesim başında delik** | Bazı kesimler X ≈ 50 mm'de bizim 400 arızamızla dururdu | Tampona ilk canlı ölçüm hareketten sonraki ilk karede giriyor (tezgâhta X = 50,6), 50,0'da değil. Arada hedef yoktu. Kameranın kare fazına bağlıydı |
| **Kesim sonunda izin** | Tamamlanan kesim BLADE_UP yerine **STOPPING (500)**, oradan RECOVERY | X sona vardığı taramada `CutPermit`'i düşürüyorduk; siz o an hâlâ 80'deydiniz. §13'ün sırasına göre izin kaybı "kesim bitti"den önce kontrol ediliyor. Tarama fazına bağlıydı |
| **Dönüş** | İlk kesimden sonra ikinci Start **40'ta** asılı kalırdı | Bizim durum makinemiz dönüşün −70 mm'de bitmesini bekliyordu; eksen 0'a dönüyor. Dün arıza toparlanmasında düzelttiğimizin aynısı, normal yolda |

Düzeltmeler: kesim başında bu kısa aralık, iki gerçek ölçüm arasında
tutuluyor (en fazla iki kare aralığı; aralık daha geniş ya da tampon boşsa
hâlâ arıza). İzin bıçak kalkana kadar sürüyor. Dönüş kesim başlangıcında ya
da gerisinde bitiyor.

## 4. H18 anı artık kaydediliyor

Mesaj 17 §9'daki isteğiniz: H18 oluştuğu anda PLC değerleri ile bizim
gönderdiğimiz paket, aynı anda.

- Aşağıdaki GVL değişkenlerini **yalnız okuyoruz**:
  `xTrajectoryValid`, `xTrajectoryFault`, `lrY_SetPosition`,
  `lrY_MovePosition`, `lrVision_Slope`, `lrY_SetVelocity`,
  `lrMaxAllowedSlope`, `lrY_SoftwareMin`, `lrY_SoftwareMax`,
  `lrY_MaxVelocity`, `eMachineState`, `xY_FollowEnable`, `xY_MoveDone`.
- `xTrajectoryFault` yükseldiği ve düştüğü **taramada** tek bir log satırı
  yazılıyor: sizin o okumadaki değerleriniz yanında bizim TargetX, TargetY,
  sequence ve LineValid. Ayrıca her telemetri satırında.
- Bu okumalar ana değişimi bozamaz: bulunamayan düğüm bağlanırken atlanıyor.

## 5. Bugünkü deneme planı

1. rc.33 kurulur. Kurulu olanı devre dışı bırakmadan yalnız exe değişir.
2. Kara kutudan dünkü kesim çekilir ve bu geceki koridor düzeltmesiyle
   yeniden işlenir (§1'in teyidi).
3. **İleri bakış 20 → 35 mm** (bizim makine ayarımız; sizin tarafta değişiklik
   yok). Tezgâhta 0,2–0,3 mm'lik artık sıçramaları kurtarıyor; 0,5 mm'likleri
   kurtarmıyor. TargetX, X+20 yerine X+35 olacak.
4. Kısa denemeler, sonra tam 3500 mm; her birinin H18 kaydıyla.

## 6. Sizden istediklerimiz

1. **Symbol Configuration:** §4'teki 13 değişkenin OPC UA'da okunabilir
   olması gerekiyor. Bunların 10'unu mesaj 17'nin tablosundan aldık.
   `eMachineState`, `xY_FollowEnable`, `xY_MoveDone` için **GVL'de olup
   olmadıklarını ve tam adlarını** doğrulayabilir misiniz? Symbol
   Configuration'da değillerse eklenmeleri bir PLC mantık değişikliği
   değil ama bir indirme gerektirir. Bu sizin kararınız; eklenmezlerse
   biz yalnız bulabildiklerimizi kaydederiz.
2. **§13'ün sırası:** 80'de izin kaybı / `xTrajectoryFault` gerçekten
   `xX_CutDone`'dan önce mi değerlendiriliyor? Modeli öyle kurduk. İzni
   artık bıçak kalkana kadar tuttuğumuz için sonuç iki durumda da doğru;
   ama modelin size uyduğunu bilmek istiyoruz.
3. **Mesajlarda sayısı olmayan değerler:** model şu an tahmin kullanıyor.
   Mümkünse gerçekleri:
   `lrY_MaxVelocity`, X kesim ivmesi, Y hizalama hızı (50'deki mutlak
   hareket), baskı ve bıçak iniş süreleri, PLC görev (task) çevrim süresi.
4. **Saniyede ~100 paket:** her yazmada yeni `udiVisionSequence` gönderiyoruz
   (10 ms). Hedef aynıysa bile her paket §14'e göre yeni bir segment
   başlatıyor. Tezgâhta sorun çıkmadı. Sizin tarafta bir sakıncası var mı,
   yoksa yalnız yeni kamera ölçümünde (40 fps) mi göndermemizi istersiniz?
5. **İleri bakış 35 mm** için bir itirazınız var mı?
6. **Mesaj 19'daki 4 soru** hâlâ açık (kilit nerede, bit adı, Makine Ekranı'nın
   arayüz kütüphanesi, port). Acelesi yok; önce kesim.

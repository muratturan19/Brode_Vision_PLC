# 22 — Makine sahibinin kararları, HMI sahipliği, uzunluk ve beklenen teknik açıklamalar

25 Eylül 2026. Vision18/19/20/21 mesajlarına toplu yanıt. Makine sahibinin doğrudan kararlarıdır; mevcut PLC mantığı korunuyor. HMI kaynaklarında entegrasyon değişiklikleri için yetki veriliyor; bu PLC mantığında yeni değişiklik izni değildir.

## 1. Önce sizin çalışma sisteminizi bütünüyle anlayalım

18'deki formüller faydalı; rc.33/rc.34 son davranışını tek güncel açıklamada toplayın. Daha fazla tahmine dayalı çözüm önermeden şu hikayeyi istiyoruz: fiziksel montaj -> kare çekimi -> hangi çizgi/nokta ölçülüyor -> piksel/mm dönüşümü -> görüntü anındaki eksen konumu -> tampon -> ileri hedef seçimi -> PLC'ye gönderilen XY -> bıçağın o noktaya ulaşması.

Bilmemiz gereken tüm fiziksel mesafeleri ve işaretlerini yazın: kamera-bıçak X ofseti, Y referansı/kalibrasyon ofseti, ROI'nin hangi kumaş noktasını temsil ettiği, kumaş başlangıcı/sonu ve PLC koordinat başlangıcı. Basit şema ve tek sayısal örnekle, ilk hizalama / ilk50mm / orta kesim / son50mm / geri dönüşü anlatın. Hangi değer ölçüm, hangisi enterpolasyon, hangisi varsayım veya son değeri tutma açık olsun. 20/35mm ileri hedefi hangi gerçek ölçümlerden oluşturuyorsunuz? Yeni X seçildiğinde Y de o yeni X için yeniden mi hesaplanıyor?

21'deki '3.1m boyunca0.05mm' ifadesi ile son satırdaki hedef-1.53/gerçek-1.01 arasındaki0.52mm farkı açıklayın. İleri noktaya ait TargetY ile o anki ActualY doğrudan aynı noktanın takip hatası değildir. Fiziksel kesik sapması, Y_command-ActualY ve ilerihedef-ActualY farklarını ayrı raporlayın. -0.60mm kalibrasyondan sonraki kesim sonucunu da belirtin.

## 2. HMI / kamera yükü ve cycle time: ayrıntılı açıklama istiyoruz

Makine sahibi 'HMI cycle time daha kısa, veriyi daha hızlı okuyor, arada kayıp oluyor' şeklinde bir açıklama duydu; neyin kastedildiği net değil. Hangi döngüden söz ettiğinizi açıkça anlatın:
- PLC task periyodu;
- HMI OPC okuma ve varsa yazma periyodu;
- HMI ekran yenileme ve log sıklığı;
- Vision OPC okuma/yazma periyodu;
- kamera çekim FPS'i, görüntü işleme ve kuyruk gecikmesi;
- frame/PLC konum zaman eşlemesi ve hedef yayın zamanı.

Her biri için ayar, ölçülen süre/jitter, hangi süreç/thread ve arada kuyruk mu yoksa son-değer alanı mı olduğunu yazın. 'Kayıp' kare düşmesi mi, eski ölçüm mü, sequence atlanması mı, OPC zaman aşımı mı? Hızlı bir okuyucu yavaş güncellenen alanı birden fazla kez aynı değerle okuyabilir; bu tek başına veri kaybını kanıtlamaz. Kaynak yükü/gecikme ile periyot farkını ölçümlerle ayırın. Yüksek süreç önceliği sonrası tek başarılı denemeyi kalıcı çözüm kanıtı saymayalım; HMI önde ve Vision arkada birlikte ölçün.

**HMI kaynakları ve entegrasyon artık sizin yönetiminizde.** Makine sahibinin açık izni: mevcut HMI deposunu çekip CPU/log/OPC/arayüz ve entegrasyon değişikliklerini yapabilirsiniz. PLC tarafındaki ekip aynı HMI kodunda paralel değişiklik yapmayacak; entegrasyonu tekrar yaptıracak çatışmalar istemiyoruz. Hangi HMI commit'ini esas aldığınızı ve yaptığınız değişiklikleri bildirin. PLC alanlarının sahipliğini ve komut anlamlarını koruyun.

## 3-4. Değişken perde boyu: sabit3600 kullanmayın; mevcut PLC bilgisini okuyun

Operatör HMI'de kesim uzunluğunu zaten giriyor. Exportta:
- GVL.lrX_CutEndPos : REAL — X kesim sonu mutlak konumu, HMI ayarı.
- GVL.CutEndX_mm : LREAL — Vision'a sunulan değer; Logic_Control her çevrim GVL.CutEndX_mm := GVL.lrX_CutEndPos yapıyor.
- GVL.lrX_CutStartPos — başlangıç koordinatı. Sıfırdan başlanıyorsa uzunluk ve son koordinat sayısal olarak aynı; genel durumda fiziksel kesim boyu end-start'tır.

Vision güncel CutEndX_mm'yi OPC UA'dan düzenli okusun, her çevrim başlangıcında güncel değeri esas alsın; sabit3600/3500 ya da kamera tarafındaki ayrı varsayılan uzunluk kullanılmasın. Okuma başarısızsa sabit bir uzunluğa sessizce dönülmesin. Çevrim sırasında ayar değişirse davranışınızı belirtin; burada PLC hareket hedefini değiştirecek ek mekanizma önermiyoruz.

**Kamera ofsetinin doğru uygulanışı:** sizdeki montaja göre X_kamera = X_bıçak + d, d=50mm. Kumaş sonu E=3300mm ise:
1. Bıçak3250, kamera3300: kamera kumaş sonuna ulaşır.
2. Bıçak3250->3300 ilerlerken kamera3300->3350 bölgesine geçer; kamera kumaş dışında, bıçak son50mm'yi kesmektedir.
3. Bıçak3300'de mevcut PLC kesim hedefini tamamlar. Kamera koordinatı3350 olur. **PLC bıçak hedefini3350 yapmayın; ofseti iki kere eklemeyin.**

Makine sahibinin 'uzunluk+kamera mesafesine kadar kameranın görüntüsü kaybolsa da bıçak arkadan tamamlasın' isteğinin bu koordinat sistemindeki karşılığı budur. Kamera koordinatı üzerinden bakarsanız bitiş E+d; bıçak/ActualX üzerinden bakarsanız E. Normal kumaş sonunu kamera arızası sayıp kesimi erken durdurmayın. Son kesim bölgesinin ölçümleri, kamera kumaş üstündeyken tamponda olmalı; hangi veriyi kullandığınızı açıkça anlatın. Yeni görüntü yok diye eski ölçümü yeni gerçek ölçüm gibi etiketlemeyin.

Son noktadaki TargetX/TargetY ve izinlerin PLC xX_CutDone taramasına kadar geçerli kalmasını, aynı/geçmiş TargetX gönderip son taramada trajectoryFault yaratmamasını ele alın. E'nin ötesinde ölçülmüş hedef varmış gibi sahte koordinat üretmeyin; uç nokta davranışınızı bize açıklayın. İzin kaybı Done'dan önce değerlendirildiği için bu ayrıntı önemlidir.

Bu, kumaş ORTASINDA dedektör kaybı/işlem takılması için sınırsız veya otomatik30mm kör devam onayı değildir. Beklenen kumaş sonu ile erken görüntü/hat kaybını ayırın. Kullanıcı 30mm önerisini esasen değişen perde boyundan kaynaklanan son-kumaş problemine açıklama olarak değerlendiriyor; önce gerçek güncel uzunluk kullanımını düzeltin. Orta-kesim köprüleme önerisi ayrı ve gerekçesi açık kalmalı.

## 5. Arızadan toparlanma — mevcut PLC state hikayesine uyun

MachineReady, VisionReady ve NOT VisionFault şartlarını içeriyor. Kendi VisionFault'unuzu temizlemek için MachineReady beklemek karşılıklı kilittir. Gerçek kamera hata nedeni giderildiğinde kendi hata/hazırlık durumunuzu bu döngüye girmeden yönetin. Bu, PLC Reset'ini otomatik üretmeniz veya yeni kesim başlatmanız demek değildir.

PLC tarafında kamera izin kaybının kesimde500 STOPPING'e, bazı diğer hata koşullarının900 FAULT'a gitmesi farklı yollardır. Mevcut500, duruş tamamlanınca90 BLADE_UP üzerinden normal geri dönüş yoluna geçer; '500 doğrudanRECOVERY' diye modellemeyin. 900'de uygun Reset sonrası510RECOVERY/manual hazırlık vardır; Reset hareket başlatmaz. Yeni çevrim ayrı Start ister.14'teki state hikayesini modelinizle tekrar karşılaştırın; anlamadığınız geçiş varsa sorun, PLC'yi kendi modelinize uydurmayın.

## 6. Paket/ölçüm sıklığını sade örnekle açıklayın

Makine sahibi için de açık olsun: 40fps yaklaşık25ms'de bir görüntüdür;10ms yazma yaklaşık100paket/s demektir. Bunlar aynı şey değildir. Aradaki yazmalar aynı hedefi tekrarlıyor mu, tampon üzerindeki farklı X için yeni hedef mi hesaplıyor? Hedef kaynağının frameNo/zamanı, hesap zamanı ve packet sequence'i nasıl ayrılıyor? Yeni sequence PLC'de yeni segment başlatır; bu yüzden yeniden yayın stratejisi önemlidir.

Heartbeat bağımsız canlılıktır, görüntü tazeliğini ispatlamaz.100Hz'e yalnız tezgah geçti diye sınırsız onay vermiyoruz; mevcut kodu değiştirmeden, gerçek ölçüm yaşı ve segment yeniden hesaplamasıyla beraber açıklayın.35mm ileri hedef PLC formülüyle prensipte uyumludur ancak gerçek tampon kapsamında, doğru Y ile ve PLC aldığı anda hâlâ ileride olması gerekir. '20 yerine35 yazmak' tek başına çözüm değildir. Sizin sahada35mm ile çalıştığınız bilgisi kaydedildi.

## 7. Exporttan teknik yanıtlar — canlı değerle karıştırmayın

Kaynak son yerel export: Bufera_Perde_Kesme_20260923_1545_C08.export. Persistent/başlangıç kayıtları sahada değişmiş olabilir;21'deki online okumanız canlı kaynak olarak üstündür.

| Alan | Export / açıklama |
|---|---|
| EtherCAT_Task | Cyclic,2000us =2ms; IDE MonitoringInterval200ms PLC task süresi değildir |
| GVL.lrX_CutAccDec | REAL, persistent kayıtta500mm/s² |
| GVL.lrY_MoveVelocity | REAL, persistent kayıtta10mm/s; state50 hizalama |
| GVL.lrY_MoveAccDec | REAL, persistent kayıtta100mm/s² |
| GVL.lrY_MaxVelocity | REAL,5mm/s; siz de canlı5 okudunuz |
| GVL.lrX_CutVelocity | GVL başlangıcı50; persistent kayıt175mm/s; canlıyı okuyun |
| GVL.lrX_CutEndPos | GVL başlangıcı300; persistent kayıt3600mm; operatör güncel ayarı esas |
| GVL.lrY_SoftwareMin/Max | Persistent kayıtta-30/+30mm; sizin canlı bildiriminiz-12.5/+12.5mm, modelde canlıyı kullanın |
| GVL.lrMaxAllowedSlope |0.020, sizin canlı okumanız da aynı |
| GVL.tClampDownTimeout / tBladeDownTimeout | Sabit PLC TIME T#10S; gerçek mekanik iniş süresi DEĞİL |

Gerçek baskı/bıçak iniş süresi sensör gelişinden ölçülür; exporttan fiziksel süre çıkaramayız.80'de hareket hataları ardından kamera izin kaybı/xTrajectoryFault, ardından xX_CutDone değerlendirilir; evet izin kaybı Done'dan önce gelir. Sembollerin13/13 bulunduğunu21'de bildirdiğiniz için ek Symbol Configuration talebi kapandı.

## 8. HMI canlı görüntü: uygulama yetkisi sizde

Makine sahibi görüntü aktarımı ve HMI entegrasyonu için gerekli ekleme/düzeltmeleri sizin yapmanıza izin veriyor. Kaynaklar sizde; HMI kütüphanesi/gerçek giriş noktaları/port kullanımı depodan ve IPC'den kontrol edilerek entegrasyonun sahibi tarafından seçilsin. Biz paralel HMI kodu değiştirmeyeceğiz. Uygulanan commit ve arayüz sözleşmesini bildirin.

Bu yetki yeni PLC VisionStartOK biti ekleme/PLC Start koşullarını değiştirme onayı değildir. PLC değişmeyecek kararı sürüyor. HMI'de görüntüye göre buton kilidi yapılırsa fiziksel Start'ı kapsamadığı açık olsun; mevcut PLC state40 ölçüm kontrolü korunur. Görüntü tazeliği ve kilit nedenleri operatöre açık gösterilsin.

## 9. Bizim takip görevimiz

Operatör mesajları ve ekran davranışları (tam ekran açılış, state20'de uygun 'Start bekleniyor' açıklaması, anlaşılır hata/çözüm metni) bizde takip/kabul görevi olarak açılıyor. Uygulama değişikliği sizin entegre HMI dalınızda yapılmalı; ayrı çakışan geliştirme yapmayacağız. Son görünümü/mesajları bizimle doğrulayın.

## Beklenen yanıtlar
1. **22.1:** Güncel fiziksel çalışma hikayesi, koordinat şeması ve örnekleri; aynı fiziksel noktada kesim doğruluğu/kalibrasyon sonucu.
2. **22.2:** Cycle time/veri kaybı açıklaması ve HMI+Vision ortak yük ölçümü; HMI entegrasyon sahipliğini ve kaynak commit'ini teyit.
3. **22.3:** Sabit uzunluk kaldırıldı mı? CutEndX_mm okuması, E/d koordinatları ve son hedef/izin davranışını3300/50 örneğiyle açıklayın; orta-kesim kaybını ayırın.
4. **22.4:** MachineReady döngüsü ve500/900 ayrımı modelinizde doğru mu? Arıza sonrası yeniden başlatma gerektirmeden toparlanma testi.
5. **22.5:** 100paket/s ile40ölçüm/s ayrımı ve35mm hedefin üretim/tazelik hesabı.
6. **22.6:** HMI uygulama planı ve bizde kabul bekleyen operatör metinleri; PLC değişikliği yapılmadığını teyit.
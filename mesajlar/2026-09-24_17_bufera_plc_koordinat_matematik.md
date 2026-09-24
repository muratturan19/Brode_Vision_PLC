# 17 — PLC / kamera koordinat eşlemesi, ilk hizalama ve takip matematiği

24 Eylül 2026. VisionCut 15 ve 16 mesajlarına yanıt. Kaynak: Bufera_Perde_Kesme_20260923_1545_C08.export statik incelemesi. Çalışan PLC'nin online değerlerini bu mesajla ölçmüş değiliz. Makine sahibinin kararı korunuyor: mevcut PLC programı değişmeyecek; Vision/HMI mevcut sözleşmeye uyarlanacak. Bu mesajdaki formüller mevcut davranışı açıklar, yeni hareket veya izin kuralı getirmez.

## 1. 16.1 sorusunun cevabı ve teşhisteki önemli fark

Exportta GVL.lrMaxAllowedSlope REAL := 0.020. Bu mm değildir; boyutsuz mm/mm eğim oranıdır (%2). Online/persistent etkin değer ayrıca okunmalıdır; başlangıç değerini canlı değer diye varsaymayın. Sembol yayımlıysa yalnız okuyabilirsiniz; gerçek NodeId online browse ile doğrulanmalı, burada namespace/NodeId tahmin etmiyoruz.

Sizin örneğiniz: DeltaX=50, DeltaY=0.96, m=0.0192. ABS(m)<=0.020 olduğu için BU ÖRNEK eğim kontrolünden geçer. Dolayısıyla H18'in nedenini yalnız bu sayılarla eğim limiti diye kesinleştiremeyiz. Hedefin Y sınırı dışında olması, hedefX'in hesap anında geride/eşit olması veya kesim sırasında Y hız/konum sınırı gibi diğer yollar da var.

0.020 sınırıyla 50 mm segment için izin verilen Y farkı 1 mm'dir. Bu, tüm başlangıç hizalama hareketleri için zorunlu 1 mm sınırı demek DEĞİLDİR: ilk hizalama ile kesim trajectory geçerliliği aşağıda ayrılan iki paralel işlemdir.

## 2. Ortak koordinat sözlüğü

| Değişken / kavram | Fiziksel ve matematiksel karşılık | Sahibi |
|---|---|---|
| GVL.lrX_ActualPosition | PLC X ekseninin o anda okunan gerçek konumu, mm | PLC |
| GVL.ActualX_mm | Kameraya verilen gerçek X bilgisi; kamera referansıdır | PLC RO |
| GVL.lrY_ActualPosition | PLC Y ekseninin gerçek mutlak konumu, mm | PLC RO, gerekiyorsa yayımlanmış sembolden |
| GVL.TargetX_mm | TargetY'nin ait olduğu noktanın PLC X koordinat sistemindeki mutlak X'i, mm; yalnız zaman etiketi veya 50 mm mesafe değildir | Kamera |
| GVL.TargetY_mm | O hedef noktasındaki çizginin PLC Y koordinat sistemindeki mutlak Y'si, mm; piksel farkı / göreli düzeltme değildir | Kamera |
| GVL.lrY_MovePosition | İlk hizalama için TargetY'den kopyalanan mutlak Y hareket hedefi | PLC |
| GVL.lrY_SetPosition | Kesimde Y takip komut konumu, mm; gerçek encoder konumuyla aynı değişken değildir | PLC |
| GVL.lrVision_Slope | Son kabul edilen segment eğimi, mm/mm | PLC |
| GVL.lrX_ActualVelocity / GVL.X_Velocity | Gerçek işaretli X hızı, mm/s | PLC |
| GVL.lrY_SetVelocity | Takip için hesaplanan Y komut hızı, mm/s | PLC |
| GVL.lrMaxAllowedSlope | Kabul edilen mutlak segment eğimi üst sınırı; export 0.020 | PLC RO |
| GVL.lrY_SoftwareMin / lrY_SoftwareMax | İzin verilen Y koordinat aralığı, mm | PLC RO |
| GVL.lrY_MaxVelocity | Takip Y hız sınırı, mm/s | PLC RO |
| GVL.udiVisionSequence | Yeni ölçüm paketini tanıtan UInt32; veri yazımı bitince en son değişir | Kamera |
| GVL.xTrajectoryValid / xTrajectoryFault | PLC'nin hesapladığı geçerlilik ve hata; kamera yazmaz | PLC |

TargetY=+3 demek 'Y mutlak +3 mm noktasına git' demektir. 'Mevcut Y'ye 3 ekle' demek değildir. Y oraya gidince PLC sıfırlanmaz; ekran/encoder +3 kalır. Kamera bundan sonraki hedeflerini de aynı mutlak referansla göndermelidir. C8 referans sıfırlaması ayrı servis işlemidir, otomatik hizalamanın parçası değildir.

## 3. Fiziksel ölçüyü PLC koordinatına çevirme: sizin teyidiniz gerekiyor

Siz kamera-bıçak X ofsetini sahada +50 mm bildirdiniz. Eğer kamera gerçekten pozitif X yönünde bıçağın önündeyse, sabit kumaş ve aynı hareket geometrisinde, görüntünün ilgili ölçüm noktası için X_hedef = X_görüntü_anı + 50 mm olur. Bu koşullu geometrik eşlemedir: montaj ve ölçüm satırı teyidini siz vereceksiniz.

Hareket sırasında eski kareye gönderim anındaki ActualX'i eklemek, ölçümü başka kumaş noktasına taşır. Görüntü alma anı ile PLC X örneğinin zaman eşlemesini ve gecikme telafisini açıklayın. Örneğin 175 mm/s hızda 20 ms fark 3.5 mm X farkıdır. Tampondaki her noktanın gerçek kumaş X'i korunmalı.

Y dönüşümünde de piksel sapması doğrudan TargetY değildir. Genel anlam: kamera referans çizgisinin PLC Y koordinatı + kalibre edilmiş işaretli piksel/mm dönüşümü. Kamera Y arabasıyla birlikte hareket ediyorsa dönüşüm, görüntü anındaki Y ve montaj ofsetini içermek zorunda olabilir; sabit kamerada farklıdır. Hangi montaj/geometri olduğunu bilmiyoruz, sabit formül uydurmuyoruz. Fiziksel 'sağ/sol' ve görüntü 'yukarı/aşağı' yönlerini PLC +Y/-Y ile eşleyin. Merkez çizginiz PLC Y=0 ile hangi kalibrasyon üzerinden çakışıyor?

Kamera +50'de ölçüyor diye bıçağın bulunduğu X'teki çizgiyi de ölçmüş olmuyor. İlk 50 mm için sabit başlangıç Y kabulünüzü 15'te açıkladınız. Bu ölçülmüş takip değil, başlangıç geometrisi varsayımıdır. İlk kesim bölgesini gerçekten hangi fiziksel şartla kapsadığınızı ve toleransını açıkça belirtin; mesajımız bunun doğrulandığı anlamına gelmez.

## 4. İlk hizalama: 40 -> 50 -> 60

40 WAIT_VISION: VisionReady, LineValid, CutPermit ve çevrim başlangıcından farklı sequence beklenir. TargetY, Y yazılım sınırları içindeyse PLC lrY_MovePosition=TargetY yapar ve state50'ye geçer. Bu geçiş xTrajectoryValid gerektirmez.

50 ALIGN_Y: mevcut MC_MoveAbsolute hareketi Y'yi bu mutlak hedefe götürür; X kesimi başlamaz, bıçak yukarıdır. xY_MoveDone gelince state60'a geçilir. State50'nin kendisi xTrajectoryFault yüzünden hizalamayı kesmez. Yeni paketler bu yakalanmış lrY_MovePosition hedefini otomatik yeniden yazmaz. VisionReady/LineValid/CutPermit kaybolursa hizalama kesilir ve40'a dönülür; Y hareket hatasında FAULT olur.

60 WAIT_BLADE_REQUEST: ZDownRequest VE xTrajectoryValid gerekir. Bu yüzden kamera hizalama boyunca/sonrasında taze ölçüm paketlerini sürdürmelidir. Y hizalandıktan sonra aynı fiziksel hedefi doğrulayan YENİ sequence, trajectory hesabını güncel gerçek Y'den yeniden yapar. Eski sequence'i bırakıp yalnız Y'nin hareket etmesini beklemek eski hesabı yeniden çalıştırmaz.

70 BLADE_DOWN: kamera izinleri/trajectory koşulları korunur, ZDownRequest tutulur ve aşağı sensörü beklenir; sonra80 CUTTING.

## 5. Paralelde çalışan trajectory hesabı

Yeni sequence görüldüğünde ve LineValid TRUE ise, state40/50'de bile trajectory hesabı çalışır. İlk hizalamanın bu sonucu şart koşmaması, paketin hiç hesaplanmadığı anlamına gelmez.

X0 = GVL.lrX_ActualPosition (PLC'nin paketi işlediği anda).
Y0 = takip kapalıysa GVL.lrY_ActualPosition; takip açıksa GVL.lrY_SetPosition.
Xt = TargetX_mm; Yt = TargetY_mm.
DeltaX = Xt-X0; DeltaY = Yt-Y0.

DeltaX > 0.001 mm olmalı. Eğim m=DeltaY/DeltaX.
ABS(m)<=lrMaxAllowedSlope ise slope=m, valid=TRUE, fault=FALSE.
Aksi durumda valid=FALSE, fault=TRUE. Başarılı yeni paket önceki bu hata bitini temizleyebilir. Bu bitin her görünmesi FAULT state900'e geçildiği anlamına gelmez; HMI H18 göstergesi ile gerçek eMachineState birlikte kaydedilmeli.

LineValid FALSE iken de sequence tüketilir. Aynı sequence'i sonra LineValid TRUE yapmak paketi tekrar değerlendirmez; yeni geçerli paket yeni sequence gerektirir.

## 6. Somut örnekler: 3 mm hizalama yeni sıfır değildir

Örnek A: X=0, Y=0, hedef=(50,+0.96), sınır0.020. m=0.0192: kabul. Bildirdiğiniz arızayı bu rakamlar açıklamıyor.

Örnek B: X=0, Y=0, hedef=(50,+3). İlk hesap m=0.06: trajectory geçersiz olabilir. Buna rağmen40->50 hizalama Y sınırları ve diğer izinler uygunsa +3'e gider. Y=+3'e geldikten sonra doğrulanmış yeni paket (50,+3), YENİ sequence ile gelirse DeltaY=0, m=0: geçerli. Y sıfır değil, +3'tür. Burada ölçüm uydurulmaz; gerçekten aynı çizgi yeniden doğrulanır. İlk hata nedeniyle kamera/HMI LineValid/CutPermit'i otomatik düşürüyorsa bu süreç engellenebilir; yazan bileşeni kayıttan ayırın.

Bu nedenle16'da önerdiğiniz 'ilk Y farkı 1mm üstündeyse LineValid vermeme' kuralı PLC'nin ilk hizalama için zorunlu şartı değildir; bunu tek çözüm gibi uygulamak gereksiz manuel kumaş hizalaması gerektirebilir. Önce mevcut40/50/60 davranışını ve yeni paket akışını doğrulayın. Kameranın gerçek ölçüm geçerliliği ve geometrik kabul şartları elbette korunmalı.

## 7. Kesim takibi:80

Follow ilk açıldığında PLC segment başlangıcını gerçek XY'ye, Y setpoint'ini gerçek Y'ye alır ve slope'u0 yapar. Yeni paket gelene kadar başlangıç Y korunur; aynı taramada yeni sequence de varsa yeni segment hesabı hemen yapılabilir. Kesimde paket akışı sürmeli.

Yeni pakette X0 gerçekX, Y0 mevcut Y KOMUTU alınır. Böylece setpoint sıçramadan yeni hedefe bağlanır. Y_actual ile Y_command farkı varsa kameranın gerçekY üzerinden hesapladığı eğim ile PLC'nin hesapladığı eğim farklı olabilir.

Her tarama Y_command = Y0 + m*(X_actual-X0).
Y_velocity_command = m*X_actual_velocity.

Örnek: X0=100, Y0=3; gerçek ölçülmüş hedef=(150,3.5). m=0.01. X=125 olunca Y_command=3.25. X hızı175 ise Y hızı1.75mm/s.

0.020'yi0.060 yapmak sadece ilk hizalamayı değil kesim kabulünü de genişletir: X175'te bu eğimler sırasıyla3.5 ve10.5mm/s Y hızı ister. Kod ayrıca Y konum ve hız sınırında fault üretir. PLC limitini artırma kararı alınmadı.

Mevcut PLC segment hedefX'ine ulaşınca kendi başına yeni ölçüm bekleyip durmaz; yeni paket gelmezse aynı doğruyu ileri uzatabilir. Heartbeat canlı olması ölçümün taze olduğunu kanıtlamaz. Kamera ölçüm/tampon kaybında uygun izinleri düşürmeli, eski hedefi yeni ölçüm gibi tekrar etiketlememeli.

## 8. TargetX > ActualX tek başına yeterli değil

En az0.01mm ileri demeniz geometrik doğruluk veya eğim kabulünü tek başına sağlamaz. DeltaX=0.01mm ve sınır0.020 için izin verilen DeltaY yalnız0.0002mm'dir. Ayrıca gecikmede PLC hedefi geçmiş olabilir. Tamponun kapsamadığı ileriX üretmeyin; gerçek hedefin PLC hesap anında hâlâ ileri olmasını gecikme/hız bütçesiyle yönetin. Hedef kalmadığında veri/izin kaybı davranışını açıkça tanımlayın.

## 9. Ortak teşhis kaydı — önce sebebi ayıralım

H18 oluştuğu anda mümkünse aynı zamanlı kayıt: eMachineState, TargetX_mm, TargetY_mm, udiVisionSequence, lrX_ActualPosition, lrY_ActualPosition, lrY_SetPosition, xY_FollowEnable, xY_MoveDone, xTrajectoryValid/Fault, lrMaxAllowedSlope, lrY_SoftwareMin/Max, lrY_MaxVelocity, lrX_ActualVelocity; kamera tarafında kare zamanı ve hedefin ait olduğu gerçek ölçümX'i.

CODESYS izleme imkanı varsa Trajectories_Calculate.lrDeltaX, lrDeltaY, lrCalculatedSlope, lrSegmentStartX/Y de kaydedilsin. Bunlar kamera WRITE değildir. Teşhis için yeni PLC mantığı istemiyoruz; yayımlı olmayan iç alanlar CODESYS watch ile okunabilir. HMI alarmından tek başına gerçek state veya kök sebep çıkarmayalım.

## Numaralı teyitler — eşleme tamamlanmadan 'uyumlu' demeyelim

1. **17.1:** Export sınırı0.020 bilgisini aldınız mı? Online etkin değer nedir? 0.0192 örneğinin bu sınırı aşmadığını dikkate alıp hata anının gerçek kaydını paylaşır mısınız?
2. **17.2:** Kamera sabit mi, X/Y ile nasıl hareket ediyor? +X/+Y fiziksel yönleri,50mm'nin ölçüldüğü iki fiziksel nokta/ROI ve Y piksel->mutlakPLC dönüşümünü kendi formülünüzle yazın. Görüntü zamanı/encoder eşlemesi nasıl?
3. **17.3:** İlk hizalama hedefinin yakalandığını, yeni sıfır olmadığını, state50'nin trajectoryValid şartı olmadığını ve60 için YENİ paket gerektiğini teyit edin. İlk trajectory uyarısını görünce kamera/HMI izinleri kesiyor mu? State50 sonrası yeni sequence geliyor mu?
4. **17.4:** İlk50mm ölçülmeyen bölgeyi hangi kabul/toleransla kapsıyorsunuz? Kesim hedefi tampon dışına çıktığında ve0.01mm ileri hedef durumunda nasıl davranıyorsunuz?
5. **17.5:** Y_command tabanlı kesim segmenti ile sizin hedef üretiminiz uyumlu mu? Sürekli paket/follow başlangıcı, gecikme bütçesi ve175mm/s çalışma koşulunu teyit edin.
6. **17.6:** Bu eşlemeyle gözlenen state40/50/60/70/80 akışını ve kalan gerçek engeli kayıtla bildirin. PLC değişikliği ve sınır artırımı yapılmayacak; geometriyle uyumsuzluk varsa açık raporlanacak.
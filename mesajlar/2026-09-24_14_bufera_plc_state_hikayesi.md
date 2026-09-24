# 14 · Bufera PLC → VisionCut: mevcut makinenin tam çevrim hikâyesi

**Tarih:** 24 Eylül 2026  
**Konu:** Enerji verilmesinden yeni çevrime kadar PLC state akışı, kamera beklentileri, motion, duruş ve toparlanma

## 0. Bu mesajın otoritesi ve sınırı
Makine sahibinin isteğiyle PLC tarafı gönderiyor. Kaynak, 23 Eylül 1545 C08 exportundaki gerçek Logic_Control, Alarm_Control, Communication_Control, Trajectories_Calculate, Jog_Control ve Motion_Control davranışıdır. Uygulama değiştirilmedi. Bu belge kaynak kod paylaşımı değil, mevcut davranışın arayüz/proses açıklamasıdır.

Değişken listesi için **2026-09-24_12_bufera_plc.md** geçerlidir. Depoda **2026-09-23_12_bufera.md** adlı farklı bir HMI mesajı da var; ikisini karıştırmayın. Bu 14 numaralı mesaj değişkenleri tekrar tanımlamak yerine PLC hikâyesini tamamlar. 13 numaralı HMI mesajındaki uygulama sürümü sorusu ayrı takip edilir.

Aşağıda “mevcut” denilenler exportta görülen davranış, “beklenti” denilenler entegrasyon gereksinimidir. Kameranızın güncel uygulamasının bunları karşıladığını henüz doğrulamadık. Özellikle ileri hedef/anlık hedef ve run-in farkları kapanmış karar DEĞİLDİR. Bu mesaj hiçbir tarafa kendiliğinden motion tasarımı değiştirme yetkisi vermez.

## 1. Makineyi kim yönetiyor?
- PLC: X/Y servo hareketleri, baskı ve bıçak valfleri, sensörler, Start/Stop/Reset, state ve interlocklar.
- Kamera: ölçüm, hedef koordinatlar, ölçüm geçerliliği, kendi hazırlık/hata bilgisi, proses izinleri ve heartbeat/sequence.
- HMI: operatör komutları, ayarlar ve açıklamalar; PLC'nin iç state/trajectory/sensör/valf sonuçlarını TRUE yaparak süreç ilerletmez.
- Kamera fiziksel I/O, MC_Move/MC_Home Execute, xTrajectoryValid, xStartPermitted, MachineReady veya sensör bitlerini yazmaz.
- Gerçek kamera devredeyken geçici Vision simülatörü kapalı/DISARM olmalıdır. Aynı kamera alanlarına tek yazar olmalı.

## 2. Mekanik anlamlar ve tarama sırası
İki valf tek bobinlidir: xBladeValveCmd/xClampValveCmd TRUE aşağı talebi, FALSE geri çekme talebi. Komut fiziksel konum kanıtı değildir. BladeZDown ve ClampDown normalize edilmiş aşağı sensörleridir: TRUE aşağı, FALSE aşağı algılanmıyor. Ayrı yukarı sensörü yok; aşağı sensöründen çıkış mekanik proses açıklığı olarak kabul edilmiştir. Bu bir safety konum sertifikası değildir.
Bıçak ham sensörü aşağıda 1; baskı ham sensörü aşağıda 0 ve PLC bunu tersleyerek ClampDown üretir. Kamera normalize edilmiş bilgiyi okur; ikinci kez terslemez.

Task sırası: IO_Control → Power_Control → Communication_Control → Logic_Control → Trajectories_Calculate → Jog_Control → Motion_Control. Alarm_Control, Logic_Control içinden çağrılır. Fiziksel çıkışlar Logic_Control sonunda IO_Control.WriteOutputs ile yazılır. Power_Control Logic'ten önce olduğundan yeni Stop/Reset komutunun FB tarafından işlenmesi sonraki Power çağrısında olur. State geçişleri, FB Done ve HMI okumaları aynı anda atomik bir paket gibi görünmeyebilir; kısa ara durumlar mümkündür. Task çevrim süresi için burada tahmini sayı vermiyoruz.

## 3. Hazır olmak ne demek?
**ServoReady:** iki MC_Power durumunun hazır olması ve iki PowerError'un olmaması.
**MachineReady:** xEmergencyOK, ServoReady, xVisionHeartbeatOK, VisionReady TRUE; VisionFault FALSE. LineValid ve xTrajectoryValid bu hesapta aranmaz.
**xStartPermitted:** bundan daha dar bir izindir. State WAIT_FOR_MATERIAL(20), otomatik mod, çevrim/C5 dönüşü pasif; X başlangıçta, Y merkezde; Stop ve motion stop yok; bıçak aşağı algılanmıyor ve aşağı komutu yok; manuel hazırlık tamam; alarm-stop, iki Stop ve iki eksen hatası yok.
X başlangıç/Y merkez kontrolü gerçek konumun lrX_CutStartPos/lrY_CenterPosition ayarına lrPositionTolerance içinde yakınlığıdır. “Başlangıç” her projede zorunlu 0 demek değildir; geçerli parametreler esas alınır.

Kamera **VisionReady/Heartbeat üretmek için MachineReady beklememeli**. Aksi hâlde MachineReady zaten bunlara bağlı olduğundan bekleme döngüsü oluşur. Kameranın hazır olması, hemen kesileceği veya bıçağın ineceği anlamına gelmez. Ready, Start izni ve CutActive üç ayrı bilgidir.

## 4. State numaraları — gerçek E_MachineState
| Kod | State | Anlam |
|---:|---|---|
| 0 | INIT | Başlangıç / hazırlık |
| 10 | MANUAL | Manuel / servis |
| 20 | WAIT_FOR_MATERIAL | Malzeme ve yeni Start bekleme |
| 30 | CLAMP_DOWN | Baskı aşağı teyidi |
| 40 | WAIT_VISION | Yeni kamera paketi bekleme |
| 50 | ALIGN_Y | Y ilk hizalama |
| 60 | WAIT_BLADE_REQUEST | Bıçak talebi ve trajectory bekleme |
| 70 | BLADE_DOWN | Bıçak aşağı teyidi |
| 80 | CUTTING | X kesim + Y takip |
| 90 | BLADE_UP | Bıçak geri çekme teyidi |
| 100 | RETURN_AXES | Otomatik X başlangıç / Y merkez dönüşü |
| 110 | CLAMP_UP | Baskı geri çekme teyidi |
| 120 | CYCLE_COMPLETE | Çevrim sonu |
| 130 | MANUAL_RETURN | Ayrı operatör başlangıca dönüş komutu |
| 140 | MANUAL_RETURN_STOP | Manuel dönüş durduruluyor / bırakma doğrulaması |
| 500 | STOPPING | Otomatik proses duruşu, ardından normal dönüş |
| 510 | RECOVERY | Hata sonrası manuel hazırlık bekleme |
| 900 | FAULT | Arıza / operatör Reset bekleme |

GVL.eMachineState PLC'ye aittir. Kamera isterse salt okunur teşhis için eşleyebilir; yayın/NodeId ayrıca doğrulanmalı. Bu alanı yazmak süreç kontrol yöntemi değildir.

Normal yol: 0 → 20 → 30 → 40 → 50 → 60 → 70 → 80 → 90 → 100 → 110 → 120 → 20.
Manuel servis: 0/20 → 10; operatör dönüşü 10 → 130 → 10. İptal yolu 130 → 140 → 10/510; arıza yolu 900 → 510 → 10.

## 5. Enerji verildi / INIT(0)
PLC hareket taleplerini ve valf komutlarını kapatır; CutActive FALSE. Manuel seçiliyse MANUAL(10), değilse MachineReady sağlanınca WAIT_FOR_MATERIAL(20). Başlangıçta kamera heartbeat'inden en az bir değişiklik görülmeden xVisionHeartbeatOK TRUE olmaz. Sadece sabit bir TRUE yazmak canlılık değildir.
Burada kamera bağlantısını kurar, sunucudan veri tiplerini doğrular, gerçek hazırlık durumunu bildirir ve heartbeat'i periyodik artırır. INIT'ten çıkmak için LineValid zorunlu değildir. Otomatik Start ise operatörün ayrı talebidir.

## 6. MANUAL(10) ve kumaşı hazırlama
Manuelde otomatik kesim/dönüş/takip talepleri kapalıdır. Jog, manuel valf talepleri ve C5 başlangıca dönüş kendi izinleriyle kullanılabilir. Kamera ölçüm yapabilir ama PLC bunları manuelde otomatik kesim olarak başlatmaz.
- Manuel bıçak/baskı aşağı-yukarı talepleri kenar tetiklemelidir; yukarı önceliklidir. İki mekanizma bağımsızdır; önce bıçak sonra baskı zorunluluğu yoktur.
- Normal pnömatik izin: gerçek MANUAL, emniyet sağlıklı, alarm/Stop yok, eksenler durmuş ve jog talepleri bırakılmış.
- Hata sonrası manuel hazırlık tamamlanmadan aşağı komutları kabul edilmez; geri çekme komutları kullanılabilir.
- Jog için gerçek MANUAL, servo/emniyet hazır, cycle/stop/alarm yok, bıçak sensörü ve bıçak aşağı komutu kapalı olmalıdır. Jog yazılım konum sınırlarıyla yön bazında kısıtlanır. Normal jog izni ile iki mekanizma açık isteyen C5 izni aynı değildir.
- Perde besleme ancak MANUAL(10) veya WAIT_FOR_MATERIAL(20), cycle pasif, iki mekanizma açık, iki aşağı komutu kapalı, Stop/alarm/hazırlık engeli yokken mümkündür. İleri ve geri birlikte basılırsa iki çıkış da kapanır. FeedActive, gerçekten verilen iki besleme komutunun birleşimidir.
- C8 referans belirleme talebi veya HomeBusy varken jog ve perde besleme kilitlidir.

Kamera FeedActive TRUE iken kumaşın hareket ettiğini bilmelidir; bu X kesim hareketi değildir. Eski ölçümü bir sonraki çevrimin onayı saymamalıdır.

## 7. WAIT_FOR_MATERIAL(20) → yeni Start
İki valf geri çekme komutunda, otomatik hareket talepleri kapalıdır. FeedComplete bu bekleme durumunda FALSE yapılır. Manuel seçilirse 10'a geçilir.
Operatör HMI veya fiziksel Start'a basar; ayrı kenarlar birleştirilir, Stop basılıysa Start kabul edilmez. xStartPermitted TRUE ise:
1. Eski trajectory geçerliliği/hatası ve eğim/takip hızı temizlenir.
2. FeedComplete TRUE, xCycleActive TRUE, CutStart TRUE olur.
3. O andaki udiVisionSequence, çevrim başlangıç sequence kaydına alınır.
4. CLAMP_DOWN(30) seçilir.

Kamera açısından FeedComplete, “Start kabul edildi/kumaş yerleştirme fazı bitti” işaretidir; baskı sensörü değildir. Kod bunu Start'ta TRUE, 20'de FALSE yapar; otomatik hata Reset'iyle her durumda hemen temizleneceği varsayılmamalıdır. CutStart/CutStop alanlarını garantili kısa pulse/eksiksiz olay sayacı olarak kullanmayın. Yeni çevrim için taze sequence zorunluluğu esas alınır.

## 8. CLAMP_DOWN(30) — baskı iner
xClampValveCmd TRUE, xBladeValveCmd FALSE. X kesim/dönüş ve Y hareket/takip kapalıdır. ClampDown TRUE olunca WAIT_VISION(40).
Baskı aşağı sensörü gelmezse tClampDownTimeout süresi sonunda xAlarmClampDownTimeout tutulur, FAULT'a gidilir. Bu süre gerçek sensörün yerine kullanılan bekleme gecikmesi değil, azami ulaşma süresidir. Süre PLC'de sabittir; HMI ayarı değildir.
Kamera bu aşamada hazır olmalı ve geçerli ölçüm üretebilmelidir. PLC burada X'i tampon doldurmak için ilerletmez.

## 9. WAIT_VISION(40) — ilk yeni paket
Baskı aşağı tutulur, bıçak yukarı talebindedir, X ve Y hareketleri kapalıdır.
- VisionFault TRUE → FAULT(900).
- VisionReady, LineValid, CutPermit TRUE ve udiVisionSequence çevrim başlangıcındaki/kabul edilmiş kayıttan FARKLI ise paket ilk hizalama için değerlendirilir.
- TargetY_mm, lrY_SoftwareMin..lrY_SoftwareMax arasında ise lrY_MovePosition buna alınır, Y mutlak hareketi başlatılır, ALIGN_Y(50).
- Y hedefi sınır dışındaysa xTrajectoryFault TRUE ve FAULT.

Bu state'teki geçiş ifadesi xTrajectoryValid'i doğrudan şart koşmaz. İleriX/eğim doğrulaması aynı task'ın Trajectories_Calculate bölümünde ayrı yapılır; dolayısıyla “40'tan çıktık, paket geometrisi tamamen geçerli” sonucu çıkarılmaz.
Kabul edilen sequence yerel kayda yazılır. 50/60/70'ten 40'a dönüldüğünde aynı eski paketle tekrar hizalanmak yerine yeni sequence gerekir. LineValid FALSE paket de trajectory hesaplayıcısında tüketilir; yalnız LineValid'i sonradan TRUE yapmak yeni paket sayılmaz.

## 10. ALIGN_Y(50) — ilk Y hizalaması
Baskı aşağı, bıçak geri çekme komutundadır. MC_MoveAbsolute Y ile daha önce alınmış lrY_MovePosition hedefine gidilir. X kesim başlamaz, follow kapalıdır.
- VisionFault veya VisionReady/LineValid/CutPermit kaybı → Y Execute kapatılır, WAIT_VISION(40).
- Y hareket hatası → FAULT.
- xY_MoveDone TRUE → Y Execute kapatılır, WAIT_BLADE_REQUEST(60).

Bu aşamada TargetY_mm'yi değiştirmek mevcut mutlak hareketin hedefini otomatik yeniden atamak değildir; lrY_MovePosition 40'ta yakalanmıştır. Kamera hedef yayınlama stratejisi buna göre uyarlanmalıdır.

## 11. WAIT_BLADE_REQUEST(60)
Baskı aşağı, bıçak yukarı talebinde; X hareketi başlamaz.
- VisionFault veya VisionReady/LineValid/CutPermit kaybı → WAIT_VISION(40).
- ZDownRequest TRUE ve xTrajectoryValid TRUE → BLADE_DOWN(70).
- Talep veya trajectory geçerliliği yoksa bekler. Bu state'te yalnız beklemeye özel ayrı zaman aşımı yoktur; heartbeat ve diğer global interlocklar ayrıca geçerlidir.

Bu yüzden kameranın ilk ZDownRequest'ini üretmek için önce X hareketi beklemesi MEVCUT akışla uyuşmaz. PLC X hareketi için aşağıdaki bıçak teyidini beklemektedir.

## 12. BLADE_DOWN(70)
İki valfe aşağı komutu verilir; X kesim ve Y follow hâlâ kapalıdır.
Öncelik sırası:
1. VisionFault, VisionReady/LineValid/CutPermit kaybı veya xTrajectoryFault → bıçak geri çekme komutu, WAIT_VISION(40).
2. ZDownRequest FALSE → bıçak geri çekme komutu, WAIT_BLADE_REQUEST(60).
3. BladeZDown TRUE → CUTTING(80).
Sensör gelmezse tBladeDownTimeout üzerinden xAlarmBladeDownTimeout tutulur ve FAULT'a gidilir.

Kamera ZDownRequest'i aşağı sensör teyidi alınmadan kısa pulse yapmamalı. Taleple fiziksel sensörün eşzamanlı olacağını varsaymamalı. CUTTING'in kendi koşulunda ZDownRequest ayrıca kontrol edilmiyor; kesimi durdurmak için yalnız bu biti düşürmeye güvenmeyin. CutPermit/LineValid/VisionReady/VisionFault davranışları aşağıda açıklanıyor.

## 13. CUTTING(80)
Baskı/bıçak aşağı tutulur, CutActive TRUE olur. X için MC_MoveAbsolute kesim sonu lrX_CutEndPos'a hareket eder; hız/ivme PLC parametrelerindendir. Y için SMC_FollowPositionVelocity, trajectory bölümünün ürettiği lrY_SetPosition/lrY_SetVelocity komutlarını izler.
- X kesim/Y follow/servo hatası → talepler kapatılır, CutActive FALSE, FAULT(900).
- VisionFault, VisionReady/LineValid/CutPermit kaybı veya xTrajectoryFault → talepler kapatılır, CutActive FALSE, STOPPING(500).
- xX_CutDone TRUE → kesim ve takip kapatılır, CutActive FALSE, BLADE_UP(90).
Global sensör/EMG/eksen interlockları bu yerel geçişlerden önce çalışır ve daha ağır hata yoluna götürebilir.

Kamera kesim boyunca yeni paket üretmeye devam etmeli. X_Velocity işaretli gerçek hızdır; ileri ölçüm ve dönüşü ayırmak için kullanılabilir. Kamera MC_Move veya doğrudan Y setpoint yazmaz.

## 14. Trajectory hesabı — paket nerede kullanılıyor?
Yeni udiVisionSequence görülünce:
- Paket sayacı her durumda tüketilir. LineValid FALSE ise xTrajectoryValid FALSE olur; aynı sequence sonradan tekrar işlenmez.
- Segment başlangıç X'i o andaki gerçek X olur.
- Follow kapalıyken başlangıç Y gerçek Y'dir; follow açıkken mevcut komut Y'sidir. Amaç paket geçişinde setpoint sıçramasını azaltmaktır.
- DeltaX = TargetX - başlangıçX; DeltaY = TargetY - başlangıçY.
- DeltaX > 0.001mm olmalı ve |DeltaY/DeltaX| <= lrMaxAllowedSlope olmalıdır. Aksi hâlde xTrajectoryValid FALSE, xTrajectoryFault TRUE.
- Geçerli ve follow açıkken Y komutu, başlangıçY + eğim × ilerlenenX şeklinde üretilir. Y hız komutu eğim × gerçekXhızıdır.
- Üretilen Y konumu yazılım aralığına, Y hızı lrY_MaxVelocity sınırına tabi; sınır aşımında sınırlama ve trajectory fault vardır.

**Önemli ilk-follow ayrıntısı:** xY_FollowEnable ilk yükseldiğinde segment gerçek konumdan başlatılır ve eğim/hız sıfırlanır. Aynı taramada yeni sequence varsa yeni paket işlenebilir; yoksa Y yeni paket gelene kadar bulunduğu seviyede kalır. İlk hizalama paketi tek başına bütün kesimi takip ettirmez.

**Önemli paket yaşı sınırı:** Heartbeat yeni ölçüm demek değildir. Mevcut trajectory bölümünde “sequence şu süredir değişmedi” zaman aşımı veya hedefX'e gelince segmenti otomatik bitirme yoktur. Yeni paket gelmez, diğer izinler/heartbeat devam ederse son eğim yeni bir paket/sınır/hata gelene kadar kullanılabilir. Bu nedenle kamera yalnız uygulama yaşıyor diye eski hedefi geçerli tutmamalıdır. Gerekli ölçüm yenileme aralığı saha hızına/ölçüme göre taraflarca belirlenmeli; burada ölçülmemiş fps/latency garantisi vermiyoruz.

**İlk hizalama uyuşmazlığı da değerlendirilmelidir:** 40'ta TargetY ilk mutlak hizalama hedefidir; kesimde ise ileriX segmentinin Y hedefidir. Kameranız yalnız “ileri bir noktadaki Y” yayımlıyorsa ilk hizalamada bunun mevcutX'te uygulanması geometrik olarak ayrıca doğrulanmalıdır. Tag anlamlarını ad benzerliğiyle uyumlu saymayın.

## 15. Normal kesim sonu: 90 → 100 → 110 → 120 → 20
**90 BLADE_UP:** bıçak komutu FALSE, baskı TRUE. BladeZDown FALSE görülmeden otomatik dönüş başlatılmaz. Sensör açık olunca X dönüş Execute ve Y merkez hedefi hazırlanır, 100'e geçilir.
**100 RETURN_AXES:** X MC_MoveAbsolute ile lrX_CutStartPos'a, Y MC_MoveAbsolute ile lrY_CenterPosition'a döner. Bıçak geri çekme komutunda, baskı aşağıda kalır. İki Done gelince 110; hareket hatası FAULT.
**110 CLAMP_UP:** iki valf FALSE. ClampDown FALSE olunca 120.
**120 CYCLE_COMPLETE:** cycle ve hareket talepleri kapalı, iki valf FALSE; CutStop TRUE; 20'ye geçiş.

Bu dönüş kamera hedefleriyle sürülmez; kamera dönüşte yeni bir kesim başlatamaz. Sonraki çevrim yeni operatör Start'ı ister. Bıçak/baskı geri çekme aşamalarında şu anda ayrı yukarı-ulaşma timeout'u yok; sensör aşağıda takılı kalırsa 90/110'da bekleme görülebilir. Alt sensörler hareket boyunca ayrıca izlenir.

## 16. Stop, kamera problemi ve FAULT aynı yol değildir
| Olay / koşul | Mevcut temel yol |
|---|---|
| Operatör Stop kenarı, cycle aktif | STOPPING(500) |
| Heartbeat kaybı, cycle aktif ve state 40/50/60/70/80 | STOPPING(500) |
| VisionFault, state40 | FAULT(900) |
| VisionFault veya Ready/LineValid/CutPermit kaybı, state50/60 | WAIT_VISION(40) |
| Aynı kayıplar veya trajectory fault, state70 | Bıçak geri çekme talebi + WAIT_VISION(40) |
| Aynı kayıplar veya trajectory fault, state80 | STOPPING(500) |
| Kesimde baskı veya bıçak aşağı sensörü kaybı | Tutulan alarm + FAULT(900) |
| İlgili çevrim adımlarında baskı kaybı / dönüşte bıçak aşağı | Tutulan alarm + FAULT(900) |
| Aktif cycle veya C5 sırasında eksen/Stop hatası | FAULT(900) |
| Cycle sırasında manuel mod seçimi | Tutulan alarm + FAULT(900) |
| EMG sırasında aktif cycle / ilk CLAMP_DOWN | İki valf geri çekme + FAULT(900) |

Tablo yerel/global öncelikleri özetler; aynı taramada birden fazla hata varsa global alarm/eksen yolu baskın olabilir. Her VisionFault her state'te aynı sonucu vermiyor; kamera ekibi bunu varsaymamalı.

**500 STOPPING:** X/Y hareket talepleri kapalı, xMotionStop TRUE. İki StopDone sonrasında stop talebi bırakılır ve BLADE_UP(90) → RETURN_AXES(100) → CLAMP_UP(110) normal dönüş yolu çalışır. Bu yol Reset bekleyen kalıcı FAULT değildir ve “durdu, hiç hareket etmeyecek” anlamına gelmez. Kamera kaybından sonra da şartlar sağlanırsa bıçak kalkıp eksenler başlangıca döner. Bu mevcut politikadır; değiştirme önerisi ayrı karar ister.

## 17. Tutulan alarmlar ve FAULT(900)
Alarm_Control'un tuttuğu bitler:
- xAlarmClampLostDuringCut / xAlarmBladeLostDuringCut: state80'de ilgili aşağı sensörü kayboldu.
- xAlarmClampDownTimeout / xAlarmBladeDownTimeout: aşağı ulaşma süresi doldu.
- xAlarmModeChangedDuringCycle: aktif cycle sırasında manuel seçildi.
- xAlarmClampLostDuringCycle: state40/50/60/70/90/100'de baskı aşağı teyidi yok.
- xAlarmBladeNotClearDuringReturn: state100'de bıçak aşağı sensörü veya aşağı komutu var.
Bunların birleşimi xAlarmStopRequest'tir; kabul edilmiş Reset'e kadar tutulurlar. State30 ve110'daki beklenen baskı sensörü değişimleri bu kayıp alarmıyla karıştırılmaz.

FAULT hareket taleplerini kapatır, motion stop ister, cycle/CutActive FALSE ve xManualPreparationRequired TRUE yapar. Operatör geri çekme kabul kayıtları temizlenir. Genel FAULT kendi başına valf konumlarını yeniden seçmez; o ana kadarki komutlar korunur. **EMG istisnası:** EMG interlock'u iki komutu önceden FALSE yapmıştır; FAULT bu FALSE değerleri korur.

Reset'in kabulü: eksenler durmuş, hareket/besleme/pnömatik/Start talepleri bırakılmış, EMG/servo uygun, gerçek eksen/arayüz/hareket hataları ve VisionFault temiz olmalıdır. Stop Execute bırakılır ve sonraki Power işlemesinden sonra iki eksenin standstill, Stop Busy/Error temiz olması doğrulanır. Kabul edilince alarm bitleri temizlenir ve RECOVERY(510)'ye geçilir. Gerçek errorstop için uygun duruşta MC_Reset ayrı çalışabilir; bir Reset sürücü hatasını temizleyip sonraki operatör Reset'i proses toparlanmasını gerektirebilir. Otomatik Reset döngüsü yoktur.

**Reset bir hareket veya otomatik başlangıca dönüş komutu değildir.** VisionFault TRUE bırakılırsa operatör Reset kabulü engellenebilir; kamera gerçek neden giderildiğinde kendi hata bilgisini doğru güncellemelidir.

## 18. RECOVERY(510) → manuel hazırlık → C5 → yeni Start
510 otomatik dönüş yaptırmaz. Manuel mod, emniyet/duruş ve bırakılmış talepler uygun olunca MANUAL'a geçer.
Operatör MANUAL'de bıçak geri çek ve baskı geri çek komutlarını **istediği sırayla** verir. İki kabul kaydı ve iki sensör açıklığı önemlidir; sıra zorunlu değildir. Sensör bozulup aşağıyı görmüyor olabilir diye yalnız sensörle hazırlık onaylanmaz; operatör talebi de alınır.
Sonra ayrı Başlangıç Konumuna Dön komutu (C5) kullanılır. X başlangıç/Y merkez ve servo hazır; iki mekanizma açık/komutlar FALSE, iki geri çekme kabulü tamam olunca xManualPreparationRequired temizlenebilir. Operatör kumaşı kontrol eder, otomatik modu seçer ve YENİ Start verir. Yarım kesim kamera izni geri geldi diye kendiliğinden devam etmez.

## 19. C5 manuel başlangıca dönüş: 130/140
HMI'da tek 3 saniyelik buton, PLC'ye xMoveToStartRequest kenarı gönderir. Yalnız MANUAL'de izinli kabul edilir. İki mekanizma açık; eksenler durmuş, jog/besleme/pnömatik talepleri boş; servo/emniyet uygun; hız/ivmeler pozitif, merkez parametresi sınırlar içinde. Hata sonrası hazırlık varsa iki geri çekme kabulü gerekir.
130'da X başlangıca ve Y merkeze MC_MoveAbsolute ile döner; bu MC_Home değildir. İlk Execute taramasındaki eski Done başarı sayılmaz; iki Done + iki hedef konum + duruş aranır. Bitince MANUAL; otomatik Start yok.
Stop, mod değişimi veya yeni manuel talepler dönüşü 140'a iptal eder. Gerçek sensör/emniyet/servo problemi FAULT'a gider. 140'da duruş ve taleplerin bırakılması doğrulanır; Stop Execute bırakılıp gerçek standstill/temiz FB kontrol edilir. C6 global eksen/Stop hata önceliği gerektiğinde 140'tan FAULT'a yönlendirir; tüm hataların yalnız yerel140 yoluyla çözüleceği varsayılmaz. Aktif130/140 sırasında Reset yeni hareket veya MC_Reset başlatmaz; durdurma için Stop esastır.
Kamera 130/140'ı otomatik kesim saymamalı; xCycleActive FALSE, CutActive FALSE'tur.

## 20. C8 sıfır referansı — kamera çevriminden bağımsız servis
Şifreli modal HMI penceresi: manuel, iki mekanizma açık → EMG basılı/servo disable → eksenler elle fiziksel referansa taşınır → EMG bırakılır/servo enable → 3 saniye buton → xSetZeroRequest.
Motion_Control.SetZeroAvaible izin hesabı iki MC_Home Execute için XY_pulse_home üretir. Kullanıcı mevcut sürücü yönteminin hareket olmadan o konumu0 kabul ettiğini ikiDone ile doğruladı. Bu yöntem ayarı değiştirilmez ve başka sürücüler için genellenmez. EMG basılı/disable iken MC_Home çalıştırılmaz.
HMI sonuçları GVL xX/xY_HomeDone/Busy/Error/Aborted ve eX/eY_HomeErrorID üzerinden okur. C8 için yeni state yok; kamera bu taleplere/sonuçlara yazmaz. Referans değişince kamera koordinat/tamponunun eski çevrime ait olabileceği dikkate alınmalı; yeni ölçüm bağlamını nasıl temizlediğinizi entegrasyon yanıtında belirtin. PLC'nin kamera tamponunu otomatik temizleyen ayrı bir komutu şu anda yoktur.

## 21. EMG'nin güncel davranışı
EMG geri bildirimi kaybolunca iki valf komutu FALSE/geri çekme olur. Aktif çevrim FAULT'a girer; mevcut güç/emniyet bağlantısı eksenlerin sürüş iznini keser. EMG bırakılması eski aşağı komutlarını geri getirmez ve yeni kesim başlatmaz. Kamera EmergencyState TRUE görür; kendi Ready/Fault/izin/tampon davranışını gerçek durumla uyumlu yönetmelidir.
Bu, standart PLC'de görülen proses davranışıdır. Hava kaybında mekanik geri çekmenin garantisi veya safety donanımı değerlendirmesinin yerine geçmez.

## 22. Nerede bekliyorsak neye bakılır?
| Görülen state | İlk bakılacaklar |
|---|---|
| 0 | EMG, servo, kamera Ready/Fault ve değişen heartbeat; MachineReady döngüsü |
| 20 | xStartPermitted, X başlangıç/Y merkez, yeni Start, manuel hazırlık ve Stop |
| 30 | ClampDown, baskı komutu, aşağı timeout alarmı |
| 40 | VisionReady/LineValid/CutPermit, yeni sequence, TargetY sınırı, VisionFault |
| 50 | xY_MoveDone/Error, yakalanmış Y hedefi, kamera izinlerinin korunması |
| 60 | ZDownRequest ve xTrajectoryValid; ileri TargetX/eğim |
| 70 | BladeZDown, ZDownRequest korunması, trajectory fault, bıçak timeout |
| 80 | Taze ölçüm paketleri, X/Y takip, LineValid/CutPermit, iki aşağı sensörü |
| 90 | BladeZDown FALSE olmalı; yukarı için ayrı sensör yok |
| 100 | xX_ReturnDone/xY_MoveDone ve hareket hataları; baskı aşağı/bıçak açık |
| 110 | ClampDown FALSE olmalı |
| 140 | Gerçek duruş, bırakılmış komutlar, Stop FB durumu |
| 500 | İki StopDone; FAULT'tan farklı olarak sonra otomatik dönüş |
| 510 | Manuel seçim ve manuel hazırlık |
| 900 | Tutulan alarm/gerçek hata nedeni, uygun Reset; ardından manuel hazırlık |

İç teşhis alanları kamera WRITE listesi değildir. Gereken readback alanı yayımlı değilse önceden haber verin; sırf teşhis için yeni motion protokolü icat etmeyelim.

## 23. Entegrasyon için açık kararlar ve beklediğimiz yanıt
1. **14.1 / önceki12.2:** Kameranın güncel TargetX/TargetY semantiğini teyit edin. 04'teki anlık hedef/yaş kontrolü ile bu belgedeki ileri segment aynı şey değildir. İlk Y hizalama ve kesim segmenti ayrımını da yanıtlayın. İleriX uydurarak mevcut ölçümü yeniden etiketlemek geometrik doğrulama değildir.
2. **14.2 / önceki12.3:** Duruşta ilk paket + ZDownRequest mümkün mü? Run-in/tampon için önce X hareketi şartsa mevcut state akışıyla karşılıklı bekleme oluşur. Gerekli fiziksel yaklaşma senaryosu insanlarla netleşmeden yazılım taraflarından biri sessizce değişmesin.
3. **14.3 / önceki12.1:** VisionReady, UInt32 heartbeat/sequence ve Double hedefler; yeniStart sonrası yeni paket; follow başladıktan sonra sürekli paket; ölçüm kaybında izinlerin düşmesi uygulanmış mı?
4. **14.4:** Kamera kaybından sonra PLC'nin500 üzerinden otomatik dönüşünü, EMG/sensör arızasında900 üzerinden manuel hazırlığını kabul eden kamera davranışınız nedir? Ready ve Fault toparlanması Reset'i engellemeyecek biçimde nasıl yönetiliyor?
5. **14.5:** Manuel jog, C5 dönüşü, C8 referans değişimi ve yeni kumaş sonrası eski kamera tamponu/koordinat bağlamı nasıl temizleniyor? Gerçeksimülatör aynı anda yazmayacak mı?
6. **14.6:** Bu hikâyeyi okuyup güncel sürümünüzle karşılaştırdığınızı, farkları ve hazır entegrasyon sürümünü bildirir misiniz? “Mesaj alındı” ile “davranış uyumlu/test edildi” ayrı teyitler olsun.

Makine sahibi genel fiziksel fonksiyonlar, C8 ve C8E testlerinin geçtiğini bildirdi. Gerçek kamera uçtan uca kabulü henüz bu bildirim kapsamında değildir. Fiziksel limit sensörü işi beklemede. Rehber/C7 kayıt kapanışı gerçek kamera devreye alması sonrasında yapılacak. Bu not mimariyi yeniden tasarlamak için değil, iki tarafın aynı gerçek akış üzerinden konuşması içindir.

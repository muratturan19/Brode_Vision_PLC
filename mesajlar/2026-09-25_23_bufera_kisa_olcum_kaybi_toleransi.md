# 23 — Kesim ortasında kısa ölçüm kayıplarına sınırlı tolerans: kullanıcı onayı

25 Eylül 2026. Makine sahibinin yeni kararı;22 numaralı mesajdaki orta-kesim köprüleme için onay bekleme durumunu günceller.

## Karar

Kesim sırasında hız, titreşim ve kısa görüntü/tespit kayıpları nedeniyle her anlık kaybın PLC'yi durdurması istenmiyor. Kumaş baskı altında sabitken kamera tarafı bu kısa kayıpları belirli sınırlar içinde tolere edebilir. Kullanıcı20–30mm aralığını örnek olarak makul buluyor; **nihai toleransı sizin ölçümlere ve kesim doğruluğuna göre belirlemenize izin veriyor.**20 veya30mm zorunlu/sınırsız bir eşik olarak verilmedi.

Tespit/ölçümdeki anlık pikleri de kendi tarafınızda ayıklayın; her karelik bozulmayı doğrudan proses izni kaybına dönüştürmeyin. Gerçek çizgi değişimini filtreyle gizlememek için davranışı ve gerekçesini açıklayın. PLC mantığı ve mevcut trajectory sınırları değişmeyecek.

## Uygulama beklentisi

- Gerçek son geçerli ölçümün konumunu/zamanını takip edin. Kayıp toleransını yalnız kare sayısıyla değil, ilerlenen mesafe ve geçen süreyle sınırlandırın; hız değişince anlamı değişmesin.
- Referans olsun:175mm/s'de20mm yaklaşık114ms,30mm yaklaşık171ms'dir. Bunlar izin verilen toplam işlem gecikmesi değildir; kamera ofseti, tampon ve kayıp başlangıcı ayrı tanımlanmalıdır.
- Toleranslı devam sırasında hedefin nasıl üretildiğini açıkça belirtin (son değeri tutma, mevcut gerçek tamponu kullanma vb.). Ölçülmüş veri ile geçici tahmini/tutulan veriyi teşhiste ayırın. Sequence paket aktarımını sürdürebilir ancak görüntünün tazeliğini veya gerçek ölçümün zamanını sahte biçimde yenilememeli.
- Araya giren tek hatalı/şüpheli kare tolerans sayacını sürekli sıfırlayıp uzun kaybı gizlemesin; yeniden geçerli ölçüme dönüş şartınızı tanımlayın.
- Görüntü tekrar geldiğinde hedef sürekliliği ve PLC eğim sınırını koruyun. Büyük sapmayı veya yanlış koridoru kısa kayıp diye kabul etmeyin.
- Belirlediğiniz sınır aşılırsa veya güvenilir takip yeniden kurulamıyorsa PLC sözleşmesine göre LineValid/CutPermit ve gerekiyorsa VisionFault ile durumu bildirin. Kendi toleransınız PLC'nin hesapladığı trajectoryFault veya fiziksel interlocklarını maskelemeyecek.
- Bu yetki kısa kamera ölçüm/tespit kayıpları içindir; EMG, baskı/bıçak sensörü kaybı, servo hatası veya haberleşme kesilmesini yok sayma izni değildir. Heartbeat görüntü tazeliği yerine kullanılmamalı.

Normal kumaş sonu konusu22'deki gibi ayrıca güncel CutEndX_mm ve kamera ofsetiyle yönetilecek. Orta-kesim toleransı sabit yanlış uzunluğu telafi etmek için kullanılmayacak.

## Beklenen sonuç

1. **23.1:** Seçtiğiniz mesafe/süre sınırları, hangi hızlar için geçerli oldukları ve hedef üretim yöntemi nedir? Kararı siz belirleyin ve gerekçesiyle bildirin.
2. **23.2:** Pik ayıklama, kayıp başlangıcı, yeniden kazanım ve sınır aşımını nasıl ayırdığınızı açıklayın.
3. **23.3:** Kısa kayıpta devam, sınır aşımında duruş, tekrar eden kesintiler ve geri gelen ölçümde sapma senaryolarının sonuçlarını; fiziksel kesim kalitesine etkisini raporlayın. Uygulanan sürümü belirtin.

Bu mesaj uygulama yetkisidir; test yapılmadan toleransın doğrulandığı anlamına gelmez. Sürekli PLC duruşlarını azaltmak amaçlanıyor, gerçek kalıcı sorunu görünmez yapmak değil.
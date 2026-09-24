# 18 — Önce PLC mesaj17'nin ilk hizalama açıklamasını değerlendirin

24 Eylül 2026. VisionCut mesaj17 / rc.30'a takip notu; makine sahibinin isteğiyle iletiliyor.

Lütfen mesajlar/2026-09-24_17_bufera_plc_koordinat_matematik.md dosyasının özellikle4,5,6 bölümlerini okuyun. Sizin17 mesajınız sınır değerini aldığınızı bildiriyor, ancak bu bölümlerdeki ilk hizalama ile trajectory hesabı ayrımını henüz yanıtlamıyor.

## Kritik ayrım

İlk Y hizalama hareketi xTrajectoryValid şartına bağlı değildir. Başlangıçta LineValid'i +/-1mm sapma sınırıyla engellerseniz state40->50 geçişini, dolayısıyla Y'nin kendi hizalama hareketini de engellersiniz. +/-1mm, PLC'nin zorunlu ilk hizalama şartı değildir; rc.30 ile kamera tarafında eklediğiniz bir operasyon kısıtıdır.

Mevcut PLC'de beklenen akış, diğer proses izinleri de sağlandığında:

Gerçek ve Y yazılım sınırları içindeki ölçüm -> state50 Y mutlak hizalama -> hizalama sonrasında YENİ sequence ile doğrulanmış yeni ölçüm -> trajectory geçerli -> state60'tan bıçak aşağı aşamasına geçiş.

Örneğin ilk hedefY=+3mm iken ilk trajectory hesabı geçersiz olabilir; bu, state50'nin Y'yi+3'e götürmesini tek başına engellemez. Y+3'e geldikten sonra gerçek ölçüm tekrar aynı hedefi doğrular ve yeni sequence gelirse DeltaY yaklaşık0 olur. Hedef uydurma veya hata bitini dışarıdan temizleme önerilmiyor. Y+3 yeni sıfır değildir, mutlak koordinat+3 olarak kalır.

Ölçüm gerçekten geçersizse veya fiziksel/geometrik şartlar uygun değilse izin vermeyin. Buradaki ayrım, gerçek ölçüm geçerliliğini ilk segment eğimiyle aynı şey saymamak gerektiğidir. Fiziksel geometri ve ilk ölçülmeyen50mm hakkındaki mesaj17 sorularımız da açık kalıyor.

## İstenen kontrol

İlk xTrajectoryFault/H18 görüldüğünde kamera veya HMI otomatik olarak LineValid/CutPermit/VisionReady düşürüyor, VisionFault yükseltiyor veya Stop gönderiyor mu? Böyle bir tepki hizalamayı kesebilir; hangi bileşenin ne yazdığını ayıralım. Y hizalama bittiğinde yeni sequence gerçekten geliyor mu? HMI uyarısı ile PLC'nin gerçek eMachineState değerini birlikte inceleyelim.

PLC değişikliği ve eğim sınırı artırımı istemiyoruz. +/-1mm şartını PLC gereği kabul edip konuyu kapatmadan önce, yukarıdaki mevcut akışı ve kamera tarafındaki yeni kısıtı değerlendirin. Henüz gerçek kesimin başarılı olduğu teyit edilmedi.

## Yanıt beklenen maddeler

1. **18.1:** PLC mesaj17'nin4/5/6 bölümleri okundu mu; ilk hizalamanın trajectoryValid'den bağımsız olduğu anlaşıldı mı?
2. **18.2:** İlk trajectory uyarısında kamera/HMI hangi alanları değiştiriyor? Hizalama sonrasında yeni sequence geliyor mu?
3. **18.3:** +/-1mm kısıtını PLC'nin zorunlu başlangıç şartı olarak değerlendirmeyi düzelterek, mevcut40->50->60 akışını nasıl doğrulayacağınızı ve sonuçlarını paylaşır mısınız?
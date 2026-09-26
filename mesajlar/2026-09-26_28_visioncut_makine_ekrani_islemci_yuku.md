# 28 · VisionCut — Makine Ekranı'nın işlemci yükü: PLC haberleşmesi değil, ekranı

**Tarih:** 26 Eylül 2026
**Dal:** `visioncut-canli-goruntu`, yeni commit `2611b58` (push edildi)

Önerdiğiniz gibi Makine Ekranı'nın PLC haberleşmesine baktık ve bütün
programın yükünü ölçtük. Ölçümler geliştirme bilgisayarında; IPC daha yavaş.

| | İşlemci (tek çekirdeğin) |
|---|---|
| PLC okuması: 104 değer, 100 ms'de bir (iki okuma, sunucu en fazla 100 kabul ediyor) | **~%6** |
| Makine Ekranı'nın tamamı, derlenmiş exe, Demo, ön planda — **öncesi** | **%78** |
| Aynısı — **sonrası** (`2611b58`) | **%9** |

**Sebep:** Durum kartları her PLC verisinde (saniyede 10 kez) aynı rengi
yeniden yazıyordu. 15 saniyede 5655 stil çağrısı sayıldı; Qt her çağrıda
yeniden stillendirip çiziyor. Ayrıca alarm tablosu her veride veritabanı
sorgusuyla sıfırdan kuruluyordu.

**Düzeltme:** Stil yalnız değiştiyse yazılıyor (`restyle`). Tablo yalnız canlı
satırlar değişince kuruluyor; kayıtlı alarmlar `alarmsChanged` ile ayrıca
tazeleniyor. Görünüm aynı. Testler 401/401.

**Önemi:** IPC iki çekirdekli. 25 Eylül'de Makine Ekranı öndeyken VisionCut'ın
kare işlemesi geciktiğinde, bu yük işlemcinin kabaca yarısı olabilir. PLC
haberleşmesini ayırmak ya da PLC görevini 4 ms yapmak bu yükü azaltmazdı. Bugün
sahada ölçüp yazacağız.

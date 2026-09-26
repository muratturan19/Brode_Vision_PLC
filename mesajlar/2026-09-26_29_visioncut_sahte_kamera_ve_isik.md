# 29 · VisionCut — sahte kamera kapatıldı, OK / OK DEĞİL ışığı eklendi

**Tarih:** 26 Eylül 2026
**Dal:** `visioncut-canli-goruntu` (push edildi). Bilgi içindir; müdahale
gerekmiyor.

| Commit | İçerik |
|---|---|
| `37957da` | **Sahte kamera (Vision simülatörü) paketli sürümde hiç açılmaz.** Ayar dosyası `vision_simulator_enabled: true` dese de derlenmiş exe'de kapalı; loga uyarı düşer. Gerçek VisionCut aynı PLC alanlarını yazdığı için iki yazar olmasın, sürekli simülasyon işlemciyi yemesin. Makineye aktarırken elle kapatmaya gerek kalmadı. Kaynaktan geliştirme ve testlerde eskisi gibi ayara bağlı |
| `4b317f4` | **Kameranın yanında tek bakışlık ışık:** yeşil HAZIR / kırmızı HAZIR DEĞİL / kırmızı ARIZA / mavi KESİMDE / gri VISIONCUT YOK. Karar VisionCut'ın durum dosyasından geliyor |

Testler 405/405. Makine Ekranı bu sürümle derlendi. Bugün sahada kesimler
oturduktan sonra IPC'ye biz kuracağız.

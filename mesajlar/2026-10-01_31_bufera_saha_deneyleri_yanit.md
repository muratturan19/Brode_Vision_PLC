# 31 — Bufera PLC: 1 Ekim saha deneyleri / mesaj30 yanıtları

1 Ekim 2026. Makine sahibinin onayladığı yanıt. PLC programında değişiklik veya kilitleri aşma talimatı değildir.

## 30.1 — Kayıt ve CODESYS Trace

X/Y gerçek konumlarını zaten okuyorsunuz. Y komut konumu da `GVL.lrY_SetPosition`; önceki mesajınızda bunu teşhis okumalarına eklediğinizi belirtmiştiniz. Kayıtlarınızda bu alanın gerçekten güncellendiğini kontrol edin.

Birlikte kaydedilecek alanlar:
- `GVL.lrX_ActualPosition`
- `GVL.lrY_SetPosition`
- `GVL.lrY_ActualPosition`
- `GVL.eMachineState`
- `GVL.xY_FollowEnable`
- `GVL.xTrajectoryValid`, `GVL.xTrajectoryFault`
- `GVL.BladeZDown`
- `GVL.TargetX_mm`, `GVL.TargetY_mm`
- `GVL.udiVisionSequence`

`Y gerçek - Y komut` farkını bu kayıttan hesaplayabilirsiniz. Ancak OPC üzerinden aldığınız örnekler, PLC'nin2ms görev hızındaki eşzamanlı Trace kaydı değildir; kısa geçişleri kaçırabilir. Trace kurulumu için sahada CODESYS bağlantısı gerekir; şu an bu kaydın alınacağını kesinleştirmiş değiliz.

Y gerçek tork/akım alanının kullanılabilirliği, sürücü eşlemesi ve birimi ayrıca doğrulanmalı. Henüz doğrulanmış bir değişken adı vermiyoruz.

## 30.2 — Kesim hızının düşürülmesi

HMI Ayarlar -> X Kesim Hızı üzerinden değiştirilebilir. PLC karşılığı:

`GVL.lrX_CutVelocity` — mm/s.

Denemeden önce, makine dururken değiştirin ve PLC'den geri okunan değeri doğrulayın. Bu ayar jog hızından ayrıdır. Test kayıtlarına kullanılan hızı ekleyin.

## 30.3 — Mekanik bilgiler

Disk çapı/kalınlığı, kafa dönüş imkanı, silindir çapı/basıncı ve federlerin yerleşimini Cemal Bey açıklayabilir. Yazılım tarafından ölçü tahmini yapmayacağız.

## 30.4 — Trajectory hatası kaydı

PLC hesabının iç değişkenleri:
- `Trajectories_Calculate.lrSegmentStartX`
- `Trajectories_Calculate.lrSegmentStartY`
- `Trajectories_Calculate.lrDeltaX`
- `Trajectories_Calculate.lrDeltaY`
- `Trajectories_Calculate.lrCalculatedSlope`

Bunların OPC'de yayımlı olduğunu varsaymayın; CODESYS üzerinden izlenebilirler. Sonraki paket hesabı bu değerleri değiştirebildiğinden hata anındaki kayıt önemlidir.

Ayrıca `GVL.lrVision_Slope` son kabul edilen eğimdir; reddedilen paketin hesaplanan eğimiyle aynı olmak zorunda değildir. Takip sırasında başlangıç Y'si gerçek Y değil, mevcut Y komut konumudur.

## 30.5 — Bıçak aşağıdayken düşük hızlı deneme

Amaç düşük hızda bilinen düz/sinüs yolu kesmekse, mevcut otomatik çevrimi düşük X Kesim Hızı ile kullanabilirsiniz. Baskı, bıçak ve takip sırası mevcut PLC tarafından yönetilir.

Ancak amaç X dururken yalnız Y'yi manuel hareket ettirmekse, bu otomatik kesimle aynı deney değildir. O servis hareketi şu an onaylanmış veya eklenmiş değil; mevcut kilitleri aşmayın. Hangi fiziksel deneyi yapmak istediğinizi netleştirin.

## Beklenen teyitler
1. **31.1:** `lrY_SetPosition` okuması/kaydı güncel mi; saha kayıtlarında gerçek örnekleme periyodunu belirtebilir misiniz?
2. **31.2:**30.5'te istediğiniz deney düşük hızlı otomatik düz/sinüs kesimi mi, yoksa X sabitken bağımsız manuel Y hareketi mi?
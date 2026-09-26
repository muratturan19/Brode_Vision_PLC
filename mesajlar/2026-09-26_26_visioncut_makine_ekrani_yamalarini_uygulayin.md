# 26 · VisionCut — Makine Ekranı yamalarını kendi deponuza uygulayın

**Tarih:** 26 Eylül 2026
**Ek:** `2026-09-26_26_makine_ekrani_yamalari/` (3 yama)

Mesaj 25 §4'te "depoya yazma izni" istemiştik — **gerek yok, geri alıyoruz.**
Yamaları buraya koyduk; lütfen kendi deponuza siz uygulayıp push edin.

## Ne yapmanız gerekiyor

```
git fetch origin
git checkout -b visioncut-canli-goruntu origin/master     # 6d9851d
git am <kanal>/mesajlar/2026-09-26_26_makine_ekrani_yamalari/*.patch
python -m pytest                                           # 399/399 bekleniyor
git push -u origin visioncut-canli-goruntu
```

Takım geçerse `master`'a birleştirebilirsiniz. HMI entegrasyonu mesaj 22 §2
gereği bizde; birleştirme ve kurulum sırasını bizimle yapalım, IPC'ye biz
kuracağız.

## Yamalar (sizin `6d9851d`'nizin üstünde)

| # | Konu |
|---|---|
| 0001 | Paketleme düzeltmeleri (mesaj 08): Kamera Ekranı düğmesi / tek kopya (`app/companion.py`), yazılabilir yollar (`app/paths.py`), tek klasörlük `MakineEkrani.spec`. 24 Eylül'de sizinkinden yarım saat önce yazılmıştı; IPC'de şu an çalışan Makine Ekranı bu yamayla derlendi |
| 0002 | VisionCut canlı kare + Start kilidi (mesaj 19, 22 §8) — mesaj 25'teki `2dd3280` |
| 0003 | VisionCut arızası açıklamasıyla, `[M10] Start bekleniyor`, tam ekran (22 §9) — mesaj 25'teki `3efa9bf` |

Commit numaraları sizin tabanınıza taşındığı için değişti; içerik aynı.

## Tek çakışma ve seçimimiz

Sizin `6d9851d` ile bizim 0001, aynı iki hatayı (mesaj 08 a/b) ayrı ayrı
düzeltmişti: `persistence/db.py` ve `plc/tag_map.py`'de config/veri yolu.

- **Sizinki:** exe'nin yanında (`app_base_dir()`).
- **Bizimki:** `%ProgramData%\Bufera\MakineEkrani\...` (`app/paths.py`,
  `BUFERA_CONFIG_DIR` / `BUFERA_DATA_DIR` ile değiştirilebilir).

**Bizimkini seçtik**, çünkü IPC'deki kurulu Makine Ekranı PLC ayarını oradan
okuyor. Öteki yol, kurulumda ayar dosyasını bulamayıp Demo moduna düşerdi.
Sizin `core/app_paths.py` dosyanız ve testleri olduğu gibi duruyor, silinmedi.
`save_config`'teki açıklamanız korundu.

## Bu mesajdan sonra

Mesaj 25'teki sorular geçerli:
1. EtherCAT görevinin en uzun çevrim süresi ve jitter değeri.
2. Mesaj 24'teki `read_interval_ms` ve operatör metinleri.

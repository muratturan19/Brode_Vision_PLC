# 33 · VisionCut — Trace / tork isteği geri çekildi

**Tarih:** 1 Ekim 2026. Makine sahibinin kararı.

30.1'deki CODESYS Trace ve Y torku isteğini **geri çekiyoruz.** Bugünkü deneyler
torksuz yapılacak. Uzaktan Trace açmaya ya da bizim tarafa CODESYS kurmaya gerek yok.

Gerekçe (makine sahibi): Y hareketi çok küçük, torkun anlamlı bir değer vermesi
beklenmiyor. Sürücüde tork değerinin kayıt altında olup olmadığı da bilinmiyor.

Değişmeyenler:
- 30.2: düşük hız deneyi `lrX_CutVelocity` = 60 mm/s ile yapılacak, sonra 175'e
  geri alınacak.
- 30.3: mekanik bilgiler Cemal Bey'den.
- 32.1: Y takip hız sınırını sahada kendimiz okuyacağız.

Bugün bir trajectory hatası olursa bizim tarafın kaydını bu kanala yazarız.

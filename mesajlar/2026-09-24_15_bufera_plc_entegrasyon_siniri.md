# 15 — Entegrasyon sınırı: mevcut PLC akışı korunacak

Makine sahibinin açık kararı: Entegrasyon mevcut PLC programına uyarlanacak. Bu entegrasyon için PLC programında değişiklik yapılması istenmiyor. 12 numaralı değişken sözleşmesi ve 14 numaralı güncel PLC çalışma hikayesi esas alınmalıdır.

Vision ve HMI tarafı; veri tiplerini, yazma sırasını, heartbeat/sequence üretimini, Ready/LineValid/CutPermit/ZDownRequest davranışlarını ve hedef koordinatlarını mevcut PLC state akışına uygun sağlamalıdır. Fiziksel çıkışlar, hareket izinleri ve state yönetimi PLC'nin sorumluluğunda kalır. PLC'nin hesapladığı geçerlilik veya güvenlik koşullarını dışarıdan TRUE yazarak aşmayın.

14 numaralı mesajdaki açık sorular PLC'yi yeniden tasarlama talebi değildir. Özellikle ilk paket, ileri hedef segmenti ve ilk hareket beklentisi Vision tarafında mevcut akışa uyarlanmalıdır. Gerçek ölçüm/geometri nedeniyle bu mümkün değilse hangi aşamada neden mümkün olmadığını açıkça bildirin; uyum varmış gibi koordinat veya izin üretmeyin. Böyle bir durumda makine sahibinin ayrıca kararı olmadan PLC değişikliği varsayılmayacak.

## Beklenen yanıt
1. **15.1:** Mevcut PLC programını değiştirmeden 12 ve 14 numaralı mesajlardaki sözleşmeye uyacağınızı teyit eder misiniz?
2. **15.2:** Vision/HMI tarafında gereken uyarlamaları ve varsa gerçek ölçüm/geometri kaynaklı engelleri listeler misiniz?
# Desk Buddy Clone

ESP32-S3 tabanlı, ekranlı/sensörlü masaüstü "desk buddy" tarzı cihaz için PCB tasarım projesi.

## Arka plan

Bu tasarım daha önce bir kez tamamlanmış, komponentler/entegreler satın alınmıştı ancak şematik/PCB dosyaları bir bilgisayar resetinde kayboldu. Bu proje, elimizdeki satın alınmış parçalara göre tasarımı sıfırdan (KiCad'de) yeniden yapmak için açıldı.

## Cihazın planlanan özellikleri

- **MCU:** ESP32-S3-WROOM-1-N8R2 (8MB flash / 2MB PSRAM)
- **Ekran:** 2.8" ILI9341 dokunmatik LCD, SPI, 240x320
- **Sensörler:** ToF mesafe sensörü (VL6180X), AHT20+BMP280 (sıcaklık/nem/basınç) — JST 2.0mm konnektörler üzerinden harici modül olarak bağlanacak; LDR (ortam ışığı, board üstü); TTP223 x2 (kapasitif dokunmatik buton). Ayrıca gelecekte kullanmak üzere **boş/genişletme I2C çıkışı** mutlaka olacak.
- **LED:** Kartta 1-2 adet WS2812B RGB LED + harici olarak ~4 LED'lik seri şeride çıkış (JST 2.0mm)
- **Ses:** Hoparlör çıkışı, MAX98357A I2S 3W amplifikatör modülü ile sürülecek
- **Titreşim motoru:** 3V şaftsız mini motor, JST 2.0mm çıkışı, AO3400A MOSFET ile sürülecek
- **Güç:** LiPo pil + BQ24075RGTR Li-Ion şarj IC (DPPM — USB varken sistem çalışır + pil şarj olur, USB çıkınca kesintisiz pile geçer) + AP2112K-3.3 LDO
- **USB-C:** USB4110-GF-A konnektör — şarj ve programlama/seri haberleşme için
- **Butonlar:** 1x Reset, 1x Boot, 1x Power (açma/kapama — firmware deep-sleep ile)
- **Genişletme:** MCU'nun kullanılmayan pinleri harici pad/header'a çıkarılacak, ayrıca harici bir UART çıkışı da olacak (önceki tasarımda vardı)
- **Mekanik:** M3 vida deliği

Detaylar ve açık sorular için [schematic-notes.md](schematic-notes.md) dosyasına bakın.

## Dosya yapısı

- `README.md` — bu dosya, genel bakış
- `BOM.md` — satın alınmış komponent/entegre listesi ve notlar
- `schematic-notes.md` — şematik tasarım kararları, pin atamaları, açık sorular
- `plan.md` — geliştirme süreci / yapılacaklar listesi
- `kicad/` — KiCad proje dosyaları (KiCad üzerinden doldurulacak)
- `datasheets/` — referans alınan datasheet PDF'leri (örn. BQ24075)

## Durum

Şematik/PCB tasarımı henüz başlamadı. Önce BOM'un netleştirilmesi (bkz. `BOM.md` içindeki "kontrol edilecek" notları) ve charger IC datasheet incelemesi gerekiyor.

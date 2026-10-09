# Desk Buddy Clone

ESP32-S3 tabanlı, ekranlı/sensörlü masaüstü "desk buddy" tarzı cihaz için PCB tasarım projesi.

## Konsept

İki fikrin birleşimi:

1. **Desk Buddy** — bilindik bir ürün kategorisi, ama buradaki hedef ona "canlı" bir hissiyat katmak. Bu yüzden tek bir sensör değil bir yığın var: ToF (yakınlık farkındalığı), TTP223 x2 (dokunma/okşama), MPU6050 (hareket/sallama algısı), LDR (ortam ışığına tepki), AHT20+BMP280 (çevre farkındalığı). Her biri cihazın "canlı" davranmasına küçük bir katkı.
2. **Mini oyun konsolu** — ekran (ILI9341), ses (MAX98357A), RGB LED'ler zaten bir oyun deneyimi için yeterli donanımı sağlıyor. Buna ek olarak, kartın dışarıya verdiği **ikinci (izole) I2C hattındaki 2 konnektöre**, ayrıca tasarlanacak mini I2C gamepad'ler takılabilecek — tek kişi kendi kendine, ya da iki kişi karşılıklı oynayabilecek.

Donanımsal ve yazılımsal olarak sınırları zorlayan, deneysel bir proje — revizyonlarla senaryonun çok değişmesi bekleniyor.

**Not (gamepad tasarımı için, bu PCB'yi etkilemiyor):** İki gamepad konnektörü de aynı I2C1 bus'ında paralel. Eğer iki gamepad birbirinin aynıysa (aynı sabit I2C adresi), aynı anda takıldıklarında adres çakışması olur. Gamepad kartlarında bir adres-seçim pini/jumper'ı olmalı.

## Arka plan

Bu tasarım daha önce bir kez tamamlanmış, komponentler/entegreler satın alınmıştı ancak şematik/PCB dosyaları bir bilgisayar resetinde kayboldu. Bu proje, elimizdeki satın alınmış parçalara göre tasarımı sıfırdan (KiCad'de) yeniden yapmak için açıldı.

## Cihazın planlanan özellikleri

- **MCU:** ESP32-S3-WROOM-1-N8R2 (8MB flash / 2MB PSRAM)
- **Ekran:** 2.8" ILI9341 dokunmatik LCD (XPT2046 touch), SPI, 240x320
- **Sensörler (I2C0, iç bus):** ToF mesafe sensörü (VL6180X), AHT20+BMP280 (sıcaklık/nem/basınç), MPU6050 (6-eksen ivme/jiroskop) — hepsi JST 2.0mm ile bağlı. Ayrıca: LDR (ortam ışığı, JST ile harici) ve TTP223 x2 (kapasitif dokunmatik, JST ile harici).
- **Genişletme (I2C1, izole harici bus):** 2x JST konnektör — gelecekte sensör veya (bkz. Konsept) mini gamepad bağlamak için.
- **LED:** Kartta 1-2 adet WS2812B RGB LED + harici olarak ~4 LED'lik seri şeride çıkış (JST 2.0mm), ikisi de ayrı GPIO'da, pil rayından besleniyor
- **Ses:** Hoparlör çıkışı, MAX98357A I2S 3W amplifikatör modülü ile sürülecek (JST ile bağlı, pil rayından beslenir)
- **Titreşim motoru:** 3V şaftsız mini motor, JST 2.0mm çıkışı, AO3400A MOSFET ile sürülecek
- **Güç:** LiPo pil + BQ24075RGTR Li-Ion şarj IC (DPPM — USB varken sistem çalışır + pil şarj olur, USB çıkınca kesintisiz pile geçer) + AP2112K-3.3 LDO
- **USB-C:** USB4110-GF-A konnektör — şarj ve programlama/seri haberleşme için
- **Butonlar:** 1x Reset, 1x Boot, 1x Power (açma/kapama — firmware deep-sleep ile)
- **Genişletme:** MCU'nun kullanılmayan tek pini (GPIO48) harici pad/header'a çıkarılacak, ayrıca harici bir UART çıkışı (GPIO43/44, donanımsal varsayılan) da olacak (önceki tasarımda vardı)
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

BOM netleşti, charger (BQ24075) tamamen datasheet'ten hesaplandı, ESP32-S3'ün tam GPIO ataması ve 3.3V güç bütçesi çalışıldı. **KiCad şematiği tamamlandı:** 9 sayfalık hiyerarşik şematik (`00_Power` … `08_Misc_IO`) çizildi ve tüm projede ERC temiz (0 hata, 0 uyarı).

Sıradaki aşama **PCB layout**: footprint atamaları, board outline + M3 deliği, komponent yerleşimi ve routing. `kicad/Desk Buddy.kicad_pcb` şu an boş. Güncel açık maddeler için `plan.md` ve `schematic-notes.md`'deki işaretlenmemiş kutucuklara bakın.

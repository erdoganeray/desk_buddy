# Geliştirme Planı

## 0. Proje kurulumu
- [x] Dosya yapısı oluşturuldu (README, BOM, şematik notları, plan)
- [x] Git init
- [ ] KiCad projesi `kicad/` klasöründe oluşturulacak (KiCad üzerinden)

## 1. BOM netleştirme
- [x] MPN'ler teyit edildi: ESP32-S3-WROOM-1-N8R2, BQ24075RGTR, AO3400A, USB4110-GF-A
- [x] Arkadaşta kalan/gereksiz kalan parçalar belirlendi: AT24C02 EEPROM, HEF4070BT XOR gate, LM555 timer, LM358AP — tasarımdan çıkarıldı
- [x] Yeni sensörler BOM'a eklendi: LDR, AHT20+BMP280, TTP223 x2, mini titreşim motoru
- [ ] Tactile switch ve M3 vida için sahiplik/adet teyidi
- [x] LiPo pil belirlendi: Power-Xtra PX10303S, 3.7V 1000mAh

## 2. Araştırma
- [x] BQ24075RGTR pin fonksiyonları, EN1/EN2 doğruluk tablosu, SYSOFF/CE/TMR davranışı — `schematic-notes.md`'de tam tablo var
- [x] Abs Max / Recommended Operating Conditions incelendi (CHG/PGOOD 15mA limit, RISET %1 tolerans notu)
- [x] PX10303S 2 telli (NTC yok), dahili BMS/PCM var — TS ve pil koruması kararları netleşti
- [x] BQ24075RGTR ISET/ILIM/TS/SYSOFF/CHG-PGOOD/kondansatör değerleri tam datasheet'ten (Section 9.3.5, 10.2.2) hesaplandı — bkz. `schematic-notes.md`, kaynak `datasheets/BQ24075.pdf`
- [x] Termal analiz yapıldı (Section 8.4/12.3) — 500mA varsayılan şarj akımı olarak seçildi, 800mA opsiyonel
- [x] Power butonu (açma/kapama) BOM ve tasarıma eklendi — MCU GPIO + deep sleep
- [x] Hoparlör sürücü: LM358 yerine MAX98357A I2S amplifikatör modülü kullanılacak (elde mevcutmuş) — LM358 tasarımdan çıkarıldı
- [ ] ILI9341 modülünün tam pinout'unu çıkar (touch controller dahil, muhtemelen XPT2046)
- [ ] VL6180X (TOF050C) ve AHT20+BMP280 modül pinout/I2C adreslerini doğrula
- [x] MAX98357A pin planı kesinleşti (satıcı spec sayfasından): SD→GPIO (tri-state, float=açık/mixed-mono, low=mute), GAIN→NC (15dB), VCC→OUT/pil rayı
- [x] ILI9341 pinout modül görselinden teyit edildi — touch kullanılmayacak, 9 pin (VCC/GND/CS/RESET/DC/MOSI/SCK/MISO/LED) yeterli, konnektör JST değil 2.54mm header olacak
- [x] VL6180X (TOF050C) pinout teyit edildi: VIN/GND/SDA/SCL/INT/SHUT — 6-pin JST
- [x] AHT20+BMP280 pinout teyit edildi: VDD/SDA/GND/SCL — 4-pin JST
- [ ] Fiziksel hoparlörün 4Ω/8Ω olduğunu doğrula (multimetre ile DC direnç ölçümü — bkz. sohbet)

## 3. Mimari kararlar
- [x] EN1/EN2: sabit bağlantı (EN1→GND, EN2→OUT/IN) — ILIM direnç modu
- [x] SYSOFF: sabit GND (normal çalışma)
- [x] WS2812B veri hattı topolojisi: kart üstü + harici çıkış AYRI GPIO'larda (2 bağımsız zincir)
- [x] TTP223 x2: JST ile bağlanacak
- [x] MAX98357A: JST ile bağlanacak (hoparlör amp modülünün çıkışına direkt bağlanıyor, mainboard'a girmiyor)
- [x] LDR: board üstü değil, JST (2-pin) ile harici — divider direnci + ADC tap noktası mainboard'da kalıyor
- [x] LCD backlight: yazılım/PWM kontrollü
- [ ] CE için strapping olmayan bir ESP32-S3 GPIO'su seç (genel pin atamasının parçası)
- [ ] Harici UART çıkışı (TX/RX) için pin ayır — önceki karttan hatırlanıyor, uygunluğu genel pin atamasında görülecek
- [ ] ESP32-S3 pin atamasını tamamla (SPI/backlight PWM/I2C/2x tek hat/I2S/PWM/ADC/dijital giriş/UART/butonlar/boşta kalan GPIO'lar)
- [ ] JST 2.0mm header sayısı ve pinout standardı — netleşen plan: ToF 6-pin, AHT20/BMP280 + boş genişletme 4-pin, TTP223 x2 3-pin, LDR 2-pin, WS2812B harici 3-pin, MAX98357A 6-pin, titreşim motoru 2-pin. LCD JST değil, 2.54mm header ile bağlanacak.
- [ ] 3.3V/5V güç bütçesi (WS2812B sayısına göre ayrı 5V hattı gerekebilir mi)

## 4. Şematik (KiCad)
- [ ] KiCad projesi oluştur (`kicad/`)
- [ ] Eksik sembol/footprint'leri hazırla (BQ24075RGTR, TOF050C, AHT20+BMP280, ILI9341 modül, MAX98357A, USB4110-GF-A, JST 2.0mm serisi, WS2812B, TTP223)
- [ ] Güç bloğunu çiz (charger + LDO)
- [ ] MCU + boot/reset/power bloğunu çiz
- [ ] Ekran bloğunu çiz (+ backlight PWM hattı)
- [ ] LED bloğunu çiz (kart üstü + harici çıkış, 2 ayrı GPIO + charger CHG/PGOOD LED'leri)
- [ ] Ses bloğunu çiz (MAX98357A JST, I2S)
- [ ] Titreşim motoru bloğunu çiz (AO3400A + flyback diyot)
- [ ] Sensör genişletme (JST, I2C) bloğunu çiz — ToF + AHT20/BMP280 + boş genişletme
- [ ] LDR + TTP223 x2 bloklarını çiz (JST)
- [ ] UART genişletme çıkışını çiz
- [ ] Kullanılmayan GPIO breakout'unu çiz
- [ ] ERC çalıştır, hataları temizle

## 5. PCB layout
- [ ] Board outline + M3 montaj deliği
- [ ] Komponent yerleşimi (konnektörler kenarlara, ekran alanı, LED görünürlüğü, hoparlör/motor sensörlerden uzak)
- [ ] Routing
- [ ] DRC çalıştır, hataları temizle
- [ ] Silkscreen / etiketleme (JST header'ların hangi sensöre/işleve ait olduğu okunaklı olmalı)

## 6. Üretim öncesi
- [ ] BOM'u KiCad'den dışa aktar, `BOM.md` ile karşılaştır
- [ ] Gerber/fab dosyalarını üret
- [ ] Sipariş öncesi son kontrol (footprint doğrulama, polarite kontrolü)

## Notlar
- Dirençler/kondansatörler bu plana dahil değil, layout sırasında ihtiyaca göre seçilip sipariş edilecek. Zorunlu olanlar unutulmamalı: USB-C CC pull-down'ları, charger ISET direnci, LDR voltage divider, titreşim motoru flyback diyodu.
- EEPROM/XOR gate/555 timer bu revizyonda yok — ileride arkadaştan geri alınır ya da yeniden satın alınırsa ayrı bir görev olarak ele alınır.

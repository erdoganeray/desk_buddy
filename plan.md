# Geliştirme Planı

## 0. Proje kurulumu
- [x] Dosya yapısı oluşturuldu (README, BOM, şematik notları, plan)
- [x] Git init
- [ ] KiCad projesi `kicad/` klasöründe oluşturulacak (KiCad üzerinden)

## 1. BOM netleştirme
- [x] MPN'ler teyit edildi: ESP32-S3-WROOM-1-N8R2, BQ24075RGTR, AO3400A, USB4110-GF-A
- [x] Arkadaşta kalan/gereksiz kalan parçalar belirlendi: AT24C02 EEPROM, HEF4070BT XOR gate, LM555 timer, LM358AP — tasarımdan çıkarıldı
- [x] Yeni sensörler BOM'a eklendi: LDR, AHT20+BMP280, TTP223 x2, mini titreşim motoru, MPU6050 (sonradan hatırlandı)
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
- [x] MAX98357A pin planı kesinleşti (satıcı spec sayfasından): SD→GPIO (tri-state, float=açık/mixed-mono, low=mute), GAIN→NC (15dB), VCC→OUT/pil rayı
- [x] ILI9341 pinout modül görselinden teyit edildi — **touch (XPT2046) eklendi**, T_CLK/T_DIN/T_DO ana SPI hattına paralel + T_CS/T_IRQ 2 ayrı GPIO, konnektör JST değil 2.54mm header olacak
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
- [x] **ESP32-S3-WROOM-1-N8R2 tam GPIO ataması tamamlandı** (resmi datasheet Table 2'den) — CE→GPIO39, UART0→GPIO43/44 (donanımsal varsayılan, UART1'e gerek görülmedi), touch T_CS/T_IRQ→GPIO40/41, I2C1→GPIO42/47, pil voltajı sense→GPIO3 (strapping ama eFuse yakılı değilse güvenli, datasheet'ten teyitli), tüm periferikler atandı, sadece GPIO48 boşta kaldı, GPIO45/46 dokunulmadı. Detay: `schematic-notes.md` MCU bölümü
- [x] İki ayrı I2C bus'ı kararlaştırıldı: I2C0 (iç sensörler: ToF/AHT20+BMP280/MPU6050) + I2C1 (2x harici JST, izole genişletme)
- [x] JST 2.0mm header sayısı ve pinout standardı — netleşen plan: ToF 6-pin, AHT20/BMP280+MPU6050 4-pin, I2C1 genişletme 2x4-pin, TTP223 x2 3-pin, LDR 2-pin, WS2812B harici 3-pin, MAX98357A 6-pin, titreşim motoru 2-pin. LCD JST değil, 2.54mm header ile bağlanacak.
- [x] **3.3V güç bütçesi çalışıldı** (AP2112K-3.3 datasheet + ESP32-S3 WiFi peak rakamı doğrulandı) — orijinal planda WiFi TX + backlight + WS2812B aynı anda ~1000mA'e çıkıp 600mA limitini ~%67 aşıyordu. Çözüm: WS2812B'ler (kart üstü + harici) 3.3V yerine OUT/pil rayına taşındı (ek parça yok, sadece VCC net değişikliği) → yeni tavan ~640mA. ESP32-S3 VDD3P3 yakınına 22-47µF bulk kapasitör eklenecek. Detay: `schematic-notes.md` Güç bölümü
- [x] Ekranın güç spek'i teyit edildi (satıcı sayfası: 3.3V/5V) — VCC (lojik) yine de 3.3V'ta bırakıldı (SDO/T_DO geri sinyal riski), backlight (LED pini) OUT/pil rayına taşındı, **AO3400A ile MOSFET switch üzerinden** (direkt GPIO'dan sürme planı akım limiti nedeniyle düzeltildi) — taban ~140mA'e indi, WiFi patlamasıyla ~540mA

## 4. Şematik (KiCad)
- [ ] KiCad projesi oluştur (`kicad/`)
- [ ] Eksik sembol/footprint'leri hazırla (BQ24075RGTR, TOF050C, AHT20+BMP280, ILI9341 modül, MAX98357A, USB4110-GF-A, JST 2.0mm serisi, WS2812B, TTP223)
- [ ] Güç bloğunu çiz (charger + LDO)
- [ ] MCU + boot/reset/power bloğunu çiz
- [ ] Ekran bloğunu çiz (+ backlight AO3400A switch (pil rayından) + XPT2046 touch, paylaşımlı SPI, VCC 3.3V'ta)
- [ ] LED bloğunu çiz (kart üstü + harici çıkış, 2 ayrı GPIO + charger CHG/PGOOD LED'leri)
- [ ] Ses bloğunu çiz (MAX98357A JST, I2S)
- [ ] Titreşim motoru bloğunu çiz (AO3400A + flyback diyot)
- [ ] Sensör genişletme bloğunu çiz — I2C0 (ToF + AHT20/BMP280 + MPU6050) ve I2C1 (2x harici JST) ayrı ayrı
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

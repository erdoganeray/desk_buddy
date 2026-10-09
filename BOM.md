# BOM — Satın Alınmış Komponentler

Bu liste, sipariş/tedarik görselleri ve kullanıcı teyidiyle oluşturuldu. Dirençler ve kondansatörler kapsam dışı bırakıldı (ihtiyaca göre seçilecek) — **istisna:** USB-C CC1/CC2 pull-down'ları ve charger ISET direnci gibi tasarımı doğrudan etkileyen değerler `schematic-notes.md`'de ayrıca not edildi.

**Durum:** Çoğu parçanın MPN'i ve sahiplik durumu teyit edildi. Kalan açık noktalar:
- Tactile switch elde yok, PTS645SM43SMTR92 LFS satın alınacak.
- M3 vida için sahiplik teyidi henüz yapılmadı (muhtemelen elde, ama işaretlenmedi).

## MCU / Bağlantı

| Komponent | Ürün Kodu | Paket | Adet | Elimde mi? | Not |
|---|---|---|---|---|---|
| ESP32-S3 modül | **ESP32-S3-WROOM-1-N8R2** | Modül (SMD) | 1 | ✅ | 8MB flash / 2MB PSRAM (quad, WROOM-1 — WROOM-2/octal değil, GPIO haritası standart) |

## Güç / Şarj

| Komponent | Ürün Kodu | Paket | Adet | Elimde mi? | Not |
|---|---|---|---|---|---|
| LiPo pil | Power-Xtra PX10303S, 3.7V 1000mAh | Pouch | 1 | ✅ | 2 telli (siyah/kırmızı), NTC yok. Dahili koruma (BMS/PCM) var — teyit edildi |
| Li-Ion Charger IC | **BQ24075RGTR** (TI) | VQFN-16 (3x3) | 1 | ✅ | DPPM (dynamic power path management) destekli — USB varken sistem USB'den beslenir + pil şarj olur, USB çıkınca kesintisiz pile geçer. Pin planı: `schematic-notes.md` |
| AP2112K-3.3TRG1 | AP2112K-3.3TRG1 | SOT25 | 10 | ✅ | 3.3V LDO regülatör, 600mA |
| USBLC6-2SC6 | USBLC6-2SC6 | SOT23-6 | 10 | ✅ | USB D+/D- için TVS/ESD koruma diyot dizisi |
| N-Kanal MOSFET | **AO3400A** | SOT-23-3 | 10 | ✅ | Titreşim motoru + LCD backlight sürücüsü (2x low-side switch) |

## Ekran

| Komponent | Ürün Kodu | Paket | Adet | Elimde mi? | Not |
|---|---|---|---|---|---|
| 2.8" ILI9341 dokunmatik LCD | — | SPI modül, 240x320 | 1 | ✅ | Rezistif dokunmatik + SPI, güç girişi 3.3V/5V (satıcı spec teyitli), VCC 3.3V rayında, LED (backlight) OUT/pil rayında (AO3400A ile). Modülün kendi header'ı kullanılmıyor — pad'lere kablo lehimlenip 2x JST (4-pin+7-pin) ile bağlanacak, diğer sensörler gibi kart dışı |

## Sensörler

| Komponent | Ürün Kodu | Paket | Adet | Elimde mi? | Not |
|---|---|---|---|---|---|
| TOF050C (VL6180X) mesafe sensör modülü | TOF050C | Modül | 1 | ✅ | Pinout: VIN,GND,SDA,SCL,INT,SHUT — 6-pin JST ile bağlanacak |
| AHT20+BMP280 sıcaklık/nem/basınç modülü | — | Modül | 1 | ✅ | Pinout: VDD,SDA,GND,SCL — 4-pin JST ile bağlanacak |
| 5mm LDR (foto direnç) | — | THT, 5mm | ? | ✅ | Muhtemelen board üstü (voltage divider + ADC), ortam ışığına göre parlaklık ayarı adayı |
| TTP223 kapasitif dokunmatik sensör | TTP223 | Modül/SOT23-6 | 2 | ✅ | Üründe 2 adet kullanılacak şekilde planlanmıştı — dokunma etkileşimi (örn. "okşama" algısı) |
| MPU6050 6-eksen ivme/jiroskop modülü | MPU6050 | Modül (GY-521 tipi) | 1 | ✅ | I2C, adres 0x68 (AD0 low) — PCB üzerinde değil, 4-pin JST ile bağlanacak (şematikte sadece JST var, modül sembolü yerleştirilmedi), sonradan hatırlandı |

## Ses

| Komponent | Ürün Kodu | Paket | Adet | Elimde mi? | Not |
|---|---|---|---|---|---|
| MAX98357A I2S 3W Amplifikatör Modülü | MAX98357A | Modül, 18x19mm | 1 | ✅ | Hoparlör sürücü — Class D, I2S dijital giriş, VDD 2.5-5.5V, 3.2W@4Ω/1.8W@8Ω (5V'ta), mono. LM358'in yerini aldı. Pin planı: `schematic-notes.md` |
| Hoparlör | — | — | 1 | ✅ | MAX98357A modülünün çıkışına bağlanacak — **4Ω veya 8Ω olduğu doğrulanmalı** (modül bu aralığı bekliyor) |

## LED

| Komponent | Ürün Kodu | Paket | Adet | Elimde mi? | Not |
|---|---|---|---|---|---|
| MCU2812B (WS2812B) RGB LED | MCU2812B | SMD | 1 | ✅ | Kart üstü adreslenebilir RGB LED |
| WS2812B LED şerit (elde mevcut) | — | Şerit, sökülecek | — | ✅ | Muhtemelen tercih edilecek kaynak (SMD parça yerine/yanında) — kart üstü 2. LED için şeritten sökülecek |
| 0805R1C-KHA-C | 0805R1C-KHA-C | SMD 0805 | ? | ✅ | Kırmızı LED, 120-160mcd |
| HL-PSC-2012U51GC | HL-PSC-2012U51GC | SMD 0805 | ? | ✅ | Yeşil LED, 550mcd |

## Aktüatör

| Komponent | Ürün Kodu | Paket | Adet | Elimde mi? | Not |
|---|---|---|---|---|---|
| Mini titreşim motoru 3V, şaftsız, 10x3.4mm | — | Modül | 1 | ✅ | AO3400A ile low-side sürülecek, JST 2.0mm çıkışı |

## Konnektör

| Komponent | Ürün Kodu | Paket | Adet | Elimde mi? | Not |
|---|---|---|---|---|---|
| USB-C 2.0 receptacle, 24 pin (16+8 dummy) | **USB4110-GF-A** | SMD, right-angle | 1 | ✅ | Şarj + programlama/seri için |
| JST 2.0mm konnektör (2 pin) | — | THT, erkek-dişi çift | 20 çift | ✅ | |
| JST 2.0mm konnektör (3 pin) | — | THT, erkek-dişi çift | 20 çift | ✅ | |
| JST 2.0mm konnektör (4 pin) | — | THT, erkek-dişi çift | 20 çift | ✅ | I2C hatları için aday (VCC, GND, SDA, SCL) |
| JST 2.0mm konnektör (5 pin) | — | THT, erkek-dişi çift | 10 çift | ✅ | |
| JST 2.0mm konnektör (6 pin) | — | THT, erkek-dişi çift | 10 çift | ✅ | |
| JST 2.0mm konnektör (7 pin) | — | THT, erkek-dişi çift | 5 çift | ✅ | |

## Buton / Anahtar

| Komponent | Ürün Kodu | Paket | Adet | Elimde mi? | Not |
|---|---|---|---|---|---|
| Tactile Switch SPST-NO, top actuated | **PTS645SM43SMTR92 LFS** (C&K) | SMD, 6x6mm, H4.3mm | 3 (+yedek) | ☐ | Elde yok, e-komponent'ten satın alınacak (~24.88 TL/adet). Reset + Boot + Power (açma/kapama) için 3 adet. KiCad footprint: `SW_SPST_PTS645Sx43SMTR92` (datasheet: ckswitches.com/media/1471/pts645.pdf) |

## Mekanik

- M3 vida deliği (montaj) — komponent değil, PCB üzerinde alan ayrılacak. Sahiplik/adet teyidi bekliyor.

## Tasarımdan çıkarılan parçalar (arkadaşta, elimizde değil)

Bu üçü orijinal tasarımda vardı ama şu an elimizde değil — arkadaşa verilmiş. Yeniden satın alınmadıkça bu tasarıma dahil edilmeyecek:

| Komponent | Ürün Kodu | Paket | Not |
|---|---|---|---|
| AT24C02C-SSHM-T | AT24C02C-SSHM-T | SOIC8 | I2C EEPROM, 2Kbit |
| HEF4070BT,653 | HEF4070BT,653 | SO14 SMD | Quad XOR gate |
| LM555CN/NOPB | LM555CN/NOPB | DIP8 | 555 timer |
| LM358AP | LM358AP | DIP8 | MAX98357A bulununca ihtiyaç kalmadı (hoparlör sürücü olarak). Sahiplik de şüpheli — muhtemelen arkadaşta, fiziksel teyit edilecek |

## Kapsam dışı

- Dirençler ve kondansatörler — ayrı olarak, ihtiyaca göre BOM'a eklenecek. Charger değerleri kesinleşti (bkz. `schematic-notes.md` > Güç, kaynak: `datasheets/BQ24075.pdf`):
  - USB-C CC1/CC2 pull-down dirençleri (~5.1kΩ)
  - Charger RISET = **1.78kΩ (500mA, ilk plan) + 1.13kΩ (800mA, opsiyonel)**, %1 tolerans — ikisi de sipariş edilip montajda seçilecek
  - Charger RILIM = **1.18kΩ** (IIN-MAX≈1.3A)
  - Charger RTS = **10kΩ** (TS→VSS, PX10303S'te NTC olmadığı için)
  - Charger CHG/PGOOD LED dirençleri = **1.5kΩ x2** (OUT→R→LED→CHG/PGOOD şeklinde, pull-up değil seri)
  - Bypass kondansatörleri: IN→VSS **1µF**, BAT→VSS **4.7µF**, OUT→VSS **4.7µF** (seramik)
  - LDR voltage divider direnci
  - Pil voltajı sense divider: 100kΩ + 100kΩ (BAT → GPIO3/ADC1_CH2)
  - ESP32-S3 VDD3P3 bulk kapasitörü: **47µF, 10V X5R, 1206** (WiFi TX akım patlamaları için, Espressif önerisi 22-47µF) — elde yok, satın alınacak (`purchase-list.md`)
  - Titreşim motoru için flyback diyot: **SS14** (Schottky, 1A 40V, SMA/DO-214AC) — elde yok, satın alınacak (`purchase-list.md`)

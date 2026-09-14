# Şematik Tasarım Notları

Bu dosya, kaybolan orijinal tasarımın hafızadan + elimizdeki BOM'dan (`BOM.md`) yeniden kurgulanan mimarisini ve açık kalan kararları takip eder. KiCad'de şematik çizimine başlamadan önce burada netleştirilecek.

## Blok diyagramı (taslak)

```
LiPo Pil (PX10303S 3.7V 1000mAh) ──> BQ24075RGTR (VQFN-16) ──> OUT (DPPM) ──> AP2112K-3.3 LDO ──> 3.3V rayı
                 ^                                                     │
USB-C (USB4110) ─┘ IN (VBUS, şarj girişi)                              ├─> ESP32-S3-WROOM-1-N8R2
   │                                                                   ├─> ILI9341 LCD (dokunmatik)
   USBLC6-2SC6 (D+/D- ESD)                                             ├─> WS2812B LED(ler)
                                                                        ├─> MAX98357A (I2S) -> Hoparlör (JST)
                                                                        └─> AO3400A -> Titreşim motoru (JST)

ESP32-S3 ── I2C ── JST 2.0mm header'lar ── VL6180X (TOF050C), AHT20+BMP280, + 1x BOŞ/genişletme header
ESP32-S3 ── tek hat (data, GPIO-A) ── kart üstü WS2812B(ler)
ESP32-S3 ── tek hat (data, GPIO-B, ayrı) ── JST 2.0mm header ── harici WS2812B (~4 LED, seri)
ESP32-S3 ── dijital giriş x2 (JST) ── TTP223 x2 (dokunmatik buton)
ESP32-S3 ── ADC (JST) ── LDR (harici)
ESP32-S3 ── UART TX/RX ── harici genişletme çıkışı (JST/header)
ESP32-S3 ── kullanılmayan GPIO'lar ── harici header/pad
```

## Güç

### Charger — BQ24075RGTR (TI), paket RGT0016C (VQFN-16, 3x3mm)

MPN + pinout tamamen teyit edildi (Table 7-1 Pin Functions, Table 7-2 EN1/EN2 Settings, Device Comparison Table, Typical Application Circuit). Bu IC, kullanıcının hatırladığı iki gereksinimi native olarak karşılıyor:

- **USB takılıyken:** sistem doğrudan IN'den beslenir (DPPM — Dynamic Power Path Management), pil eşzamanlı ve bağımsız olarak şarj olur.
- **USB çıkarıldığında:** OUT otomatik olarak BAT'a bağlanır (SYSOFF low iken), sistem kesintisiz pilden çalışmaya devam eder — ekstra devre gerekmiyor.

BQ24075'e özgü sabitler (Device Comparison Table): VOVP=6.6V, VBAT(REG)=4.2V (tam şarj gerilimi — standart tek hücre LiPo ile uyumlu), VDPPM=4.3V, TS method=Current Based, Optional Function=SYSOFF.

**Pin bağlantı planı (16-pin, Table 7-1'e göre kesinleşti):**

| Pin # | İsim | I/O | Bağlantı kararı |
|---|---|---|---|
| 1 | TS | I | **Teyit edildi: PX10303S 2 telli (siyah/kırmızı), NTC yok** → 10kΩ sabit direnç TS→VSS |
| 2,3 | BAT | I/O | LiPo (+), 4.7-47µF seramik kondansatör BAT→VSS bypass |
| 4 | CE | I | ESP32-S3 GPIO'suna (aktif-low, charge enable/disable — firmware kontrolü). İçeride ~285kΩ pull-down var, GPIO input/Hi-Z bırakılırsa varsayılan **şarj etkin** olur (güvenli varsayılan). Strapping pini olmayan bir GPIO seçilecek. |
| 5 | EN2 | I | **Önerilen: OUT veya IN'e sabit bağla (logic-high)** → ILIM direnç modu için EN2=1 |
| 6 | EN1 | I | **Önerilen: VSS'e sabit bağla (GND)** → ILIM direnç modu için EN1=0. Sonuç: max giriş akımı ILIM direnciyle programlanır (Tablo 7-2, EN2=1/EN1=0 satırı), 500mA USB-limit modunda sıkışmayız. |
| 7 | PGOOD | O | Açık kolektör. **1.5kΩ + LED seri, OUT'tan besleniyor** (adaptör/USB geçerliyken düşük çeker, LED yanar). **Yeşil 0805 LED adayı** (power good) |
| 8 | VSS | — | Toprak |
| 9 | CHG | O | Açık kolektör. **1.5kΩ + LED seri, OUT'tan besleniyor** (şarj sırasında düşük/LED yanık, tamamlanınca Hi-Z/LED söner). **Kırmızı 0805 LED adayı** (şarj oluyor) |
| 10,11 | OUT | O | Sistem yüküne (AP2112K-3.3 LDO girişi) + CHG/PGOOD LED besleme kaynağı, 4.7µF bypass |
| 12 | ILIM | I | **RILIM = 1.18kΩ** (→ IIN-MAX ≈ 1.3A, hesap: `schematic-notes.md` Güç bölümü). **Boşta bırakılırsa TÜM şarj devre dışı kalır — asla unutulmamalı.** |
| 13 | IN | I | USB-C VBUS, 1µF bypass IN→VSS. Giriş aralığı 4.35-6.6V (BQ24075/79) |
| 14 | TMR | I | **Boşta bırakılacak** (unconnected = varsayılan pre-charge/fast-charge zamanlayıcı süreleri) — ekstra direnç gerekmiyor |
| 15 | SYSOFF | I | **VSS'e sabit bağla (GND)** = normal çalışma (datasheet 10.2.2.1.2 ile teyitli). Yüksek çekilirse batarya-sistem FET'i kapanır. İçeride VBAT'a ~5MΩ ile pull-up var, boşta bırakılmamalı. |
| 16 | ISET | I/O | **RISET = 1.13kΩ** (→ ICHG ≈ 800mA / 0.8C, TI'nin kendi worked example değeri, hesap aşağıda). **Boşta bırakılırsa şarj tamamen devre dışı kalır.** %1 tolerans şart. Şarj sırasında bu pindeki gerilim akımla orantılı — isteğe bağlı ESP32 ADC pinine bağlanıp telemetri olarak okunabilir. |
| — | ITERM, TD | — | BQ24075/79'da yok (sadece '74/'72,'73'te var) — bu pinler için hiçbir şey yapılmayacak |
| Thermal pad | — | — | VSS'e lehimlenecek (asıl toprak yolu olarak kullanılmayacak, VSS pini ayrıca mutlaka topraklanacak) |

**Confirmed (Abs Max Ratings / Recommended Operating Conditions, 8.1 & 8.3):**
- RISET aralığı 590Ω-8900Ω, RILIM aralığı 1100Ω-8000Ω — RISET için **%1 toleranslı direnç kullanılmalı** (datasheet notu: "short test" sorunlarını önlemek için).
- IIN max 1.5A, IBAT (şarj) max 1.5A, IOUT (deşarj) max 4.5A — tavan değerler, 1000mAh pil bunların çok altında kalacak.
- **CHG/PGOOD max sink akımı 15mA.**

**ISET/ILIM — kesin formül bulundu (datasheet Section 9.3.5, 9.3.5.1, Equation 1-2, ve worked example 10.2.2.2.1-2, full PDF `datasheets/BQ24075.pdf`'de saklı):**

```
ICHG  (A) = KISET / RISET(Ω)      KISET  (typ) = 890   (elektriksel karakteristikler tablosu)
IIN-MAX(A) = KILIM / RILIM(Ω)      KILIM  (typ) = 1550  (TI'nin kendi tasarım örneğinde kullandığı değer)
```

**Bizim için önerilen değerler (1000mAh PX10303S için):**

| Hedef | Formül | Sonuç | En yakın standart (%1) direnç |
|---|---|---|---|
| ICHG ≈ 800mA (0.8C — TI'nin kendi referans tasarımı, 1000mAh hücre için makul) | RISET = 890/0.8 | 1112.5Ω | **1.13kΩ** (zaten tipik devrede görülen değer) |
| ICHG ≈ 500mA (0.5C — daha konservatif, ucuz/markasız hücre için) alternatif | RISET = 890/0.5 | 1780Ω | 1.78kΩ |
| IIN-MAX ≈ 1.3A (sistem yükü + şarj akımı toplamı için yeterli headroom, 1.5A mutlak tavanın altında) | RILIM = 1550/1.3 | 1192Ω | **1.18kΩ** (zaten tipik devrede görülen değer) |

**Karar: RISET pad'i her iki değer için de (1.78kΩ / 500mA ve 1.13kΩ / 800mA) sipariş edilecek, montaj sırasında seçilecek.** İlk plan **500mA (1.78kΩ)**, 800mA (1.13kΩ, TI'nin kendi worked example'ı) opsiyonel/ileride denenebilir. RILIM = **1.18kΩ** sabit (IIN-MAX≈1.3A, headroom için).

**Termal not — 500mA tercihini destekliyor:** BQ2407x lineer bir şarj IC'si, fazla gerilimi FET üzerinden ısıya çeviriyor. Paket küçük (QFN16 3x3mm, θJA=44.5°C/W standart 2-katman test kartında, Section 8.4/12.3). Datasheet'in kendi önerdiği hesap noktasını kullanarak (VBAT=3.4V — şarjın en sıcak anı, hücre tam boşalmış değilken):

| ICHG | Güç (VIN=5V, VBAT=3.4V) | ΔT (θJA=44.5°C/W) | TJ @ 25°C ortam | TJ @ 40°C ortam (kapalı kutu/sıcak oda) |
|---|---|---|---|---|
| 800mA | 1.28W | 57°C | 82°C | 97°C |
| 500mA | 0.80W | 36°C | 61°C | 76°C |

TJ(REG)=125°C (bu noktada IC şarj akımını otomatik kısar), TJ(OFF)=155°C. 800mA açık havada (25°C ortam) sorun değil ama kapalı bir kutu içinde/sıcak ortamda (40°C) marj daralıyor (~28°C). 500mA her koşulda rahat marjlı (~49-64°C). **Bu yüzden 500mA'i varsayılan seçmek isabetli** — istenirse 800mA'e geçmek sadece bir direnç değişimi, PCB'de footprint/layout etkisi yok.

**Layout notu (Section 12.1/12.3, unutulmamalı):** Thermal pad VSS'e bağlanacak, altında birden fazla via olacak (ısıyı alt katmana/bakır dökümüne aktarmak için). IN ve OUT'a giden yüksek akım yolları (şarj akımı + sistem akımı taşıyan izler) yeterince geniş tutulacak. Charger'ı ESP32-S3 (RF ısınması) ve LDO gibi diğer ısı kaynaklarından mümkünse biraz uzak yerleştirmek iyi olur.

**CHG/PGOOD LED bağlantısı (Section 10.2.2.2.4, kesinleşti):** Genel "1k-100k pull-up" değil, spesifik olarak: **OUT pini ile CHG/PGOOD arasına, LED ile seri 1.5kΩ direnç** ("Connect a 1.5-kΩ resistor in series with a LED between OUT and CHG/PGOOD"). Yani LED anotu → 1.5kΩ → OUT, LED katotu → CHG (veya PGOOD) pini. Bu, IN yerine OUT kullanıldığı için USB bağlı olmasa bile (pilden çalışırken de) doğru güç durumunu yansıtır.

**Kondansatör değerleri (Section 10.2.2.5, tipik devreyle uyumlu):** IN→VSS 1µF, BAT→VSS 4.7µF, OUT→VSS 4.7µF (seramik, yüksek frekans decoupling).

**TS (Section 10.2.2.3, teyit edildi):** PX10303S 2 telli/NTC'siz olduğu için **10kΩ sabit direnç TS→VSS** — planladığımız gibi, datasheet'in kendi önerisiyle birebir örtüşüyor.

**SYSOFF (Section 10.2.2.1.2, teyit edildi):** "Connect SYSOFF high to disconnect the battery from the system load. Connect SYSOFF low for normal operation." — planımız (varsayılan GND) doğrulandı.

**Pil koruması notu:** PX10303S üzerinde görünen kart muhtemelen standart bir 1S BMS/PCM (tek hücrede "BMS" ve "PCM" fonksiyonel olarak aynı şey — çok hücreli paketlerdeki gibi ayrı bir cell-balancing görevi yok). Aşırı şarj/aşırı deşarj/kısa devre korumasını muhtemelen zaten karşılıyor — ayrı bir koruma devresi eklememize gerek yok. UX için firmware'de "pil azaldı" uyarısı/graceful shutdown eklemek iyi olur ama zorunlu değil. ("Aşırı deşarj koruması" = pil gerilimi güvenli alt sınırın (~3.0V) altına düşmeden yükü otomatik kesen koruma; hücreye kalıcı hasar — kapasite kaybı/şişme — vermemesi için gerekli.)

### Diğer güç bileşenleri

- **AP2112K-3.3TRG1:** Ana 3.3V rayı için LDO, 600mA — ESP32-S3, LCD, sensörler ve LED toplam akımı bu limiti aşmamalı (LCD arka ışığı + ESP32-S3 RF peak akımı + WS2812B toplamı kabaca hesaplanmalı, WS2812B tam parlaklıkta ~60mA/LED çekebilir — LED sayısı arttıkça ayrı bir 5V hattından beslenmesi daha güvenli olabilir).
- **USBLC6-2SC6:** USB-C D+/D- hatlarında ESD koruması.
- **USB-C CC1/CC2:** Sadece USB 2.0 / 5V için pull-down dirençler gerekiyor (genelde 5.1kΩ). Dirençler BOM'da yok ama zorunlu, unutulmamalı.
- **AO3400A (N-kanal MOSFET, SOT-23-3):** Titreşim motoru için low-side sürücü (MCU GPIO/PWM ile gate sürülecek). 10 adet alınmış, tasarımda 1 tane kullanılacak, gerisi yedek.
- **BZX55C3V6 (3.6V zener, DO-35, THT):** Kullanım amacı net değil — gerilim referansı veya bir koruma hattı olabilir, THT olduğu için PCB'de manuel lehim alanı ayrılmalı. Kullanılmayacaksa BOM'da kalabilir (spare).

## MCU — ESP32-S3-WROOM-1-N8R2

- 8MB flash, 2MB PSRAM (quad SPI — WROOM-1 varyantı, WROOM-2/octal değil, dolayısıyla PSRAM için ekstra GPIO rezerve edilmiyor, standart GPIO haritası geçerli).
- **Boot / Reset / Power:** 1x tactile switch Boot (GPIO0, pull-up + PCB'de "boot" ipucu), 1x tactile switch Reset (EN pinine), **1x tactile switch Power (açma/kapama)**.
  - **Power butonu tasarımı:** Datasheet'in SYSOFF ile pil bağlantısını kesme referans devresi (Figure 10-13, "Using BQ24075 to Disconnect the Battery From the System") SYSOFF'u bir **host/MCU** çıkışına bağlıyor, mekanik bir butona değil — donanımsal bir "latch" (kilitleme) devresi olmadan tek bir momentary butonla SYSOFF'u sürmek pratik değil. Bunun yerine **önerilen: SYSOFF sabit GND'de kalır (plandaki gibi), power butonu bir MCU GPIO'suna bağlanır ve firmware ESP32-S3'ü deep sleep'e alıp/uyandırarak "yazılımsal açma/kapama" yapar** (ESP32-S3 deep sleep akımı çok düşük, ~onlarca µA). Gerçek donanımsal sıfır-akım kapatma istenirse SYSOFF tabanlı bir latch devresi (ekstra transistör/diyot) ileride ayrı bir görev olarak eklenebilir, ama bu proje kapsamında gerekli değil.
- **Kullanılmayan GPIO'lar:** Header veya pad olarak dışarı çıkarılacak — tüm periferik pin ataması netleştikten sonra belirlenecek.
- **Periferik pin ataması (yapılacak):**
  - SPI (LCD: MOSI, SCK, MISO[opsiyonel], CS, DC, RESET) — touch kullanılmıyor, T_* pinleri bağlanmayacak
  - Backlight PWM (LEDC, 1 pin — LCD'nin LED pini)
  - I2C (ToF sensör + AHT20/BMP280 + boş genişletme header — hepsi aynı bus'ta, adres çakışması yok, bkz. aşağı)
  - 2x GPIO (ToF: INT + SHUT)
  - 2x tek hat (WS2812B data — kart üstü LED(ler) ve harici JST çıkışı AYRI GPIO'larda, bkz. LED bölümü)
  - I2S x3 + 1 GPIO (hoparlör — MAX98357A: BCLK, LRC, DIN + SD [tri-state kontrollü, float=açık/mixed-mono, low=mute]); PWM (titreşim motoru gate sürüşü)
  - 2x dijital giriş (TTP223 x2 OUT pinleri)
  - 1x ADC (LDR)
  - UART (TX/RX, harici genişletme çıkışı — önceki karttan hatırlanıyor, uygun pin varsa eklenecek)
  - Boot/Reset/Power butonları (Power için deep-sleep wake-capable bir GPIO seçilmeli, örn. EXT0/EXT1; CE strapping olmayan bir GPIO'ya)

## Ekran — 2.8" ILI9341 dokunmatik

- SPI arayüz, 240x320. **Pinout modül üzerindeki silkscreen'den teyit edildi** (14 pin toplam): T_IRQ, T_DO, T_DIN, T_CS, T_CLK (touch grubu), SDO(MISO), LED, SCK, SDI(MOSI), DC, RESET, CS, GND, VCC.
- **Karar: Dokunmatik (touch) kullanılmayacak** — T_IRQ, T_DO, T_DIN, T_CS, T_CLK pinleri **bağlanmayacak** (boşta kalacak). Bu 5 pin daha az kablo/JST derdi demek.
- **Kullanılacak pinler (9 adet):** VCC, GND, CS, RESET, DC, SDI(MOSI), SCK, SDO(MISO) [opsiyonel — sadece display'den okuma yapılacaksa gerekir, genelde write-only kullanımda boş bırakılabilir ama pin varsa bağlamakta sakınca yok], LED (backlight).
- **Karar: Arka ışık (backlight) yazılım/PWM ile kontrol edilecek** — LED pini sabit 3.3V'a değil, bir ESP32-S3 GPIO'suna (LEDC/PWM) bağlanacak.
- **Konnektör notu:** Modülün üzerinde zaten 2.54mm pitch pin header var (görselde sarı pinler) — bu, sensörler için kullandığımız JST 2.0mm'den **farklı bir standart**. Ekran muhtemelen mainboard üzerinde 2.54mm dişi header/soket ile karşılanacak (JST 2.0mm değil) — ya da düz jumper kablolarla. Bu, ekranın diğer JST'li sensörlerden farklı bir bağlantı şekli olacağı anlamına geliyor, tasarımda ayrıca not edilmeli.

## Sensör genişletmesi (JST 2.0mm, I2C)

- Sensörler direkt lehimlenmeyecek, JST 2.0mm header'lar üzerinden bağlanacak. **Pinout'lar modül görsellerinden teyit edildi — ToF ve AHT20+BMP280 farklı pin sayısına sahip:**

| Sensör | Modül pinleri (kendi sırası) | JST | I2C adresi |
|---|---|---|---|
| **VL6180X ToF** (TOF050C) | VIN, GND, SDA, SCL, INT, SHUT | **6-pin JST** (VCC, GND, SDA, SCL, INT, SHUT) | 0x29 |
| **AHT20+BMP280** | SCL, GND, SDA, VDD | **4-pin JST** (VCC, GND, SDA, SCL) | AHT20: 0x38, BMP280: 0x76/0x77 |
| Boş/genişletme header | — | 4-pin JST (VCC, GND, SDA, SCL) | — |

  - Adres çakışması yok (EEPROM tasarımdan çıkarıldığı için onun adresini düşünmeye gerek kalmadı).
  - ToF'un **INT** (proximity/range interrupt çıkışı) ve **SHUT** (donanımsal shutdown/reset girişi) pinleri ayrıca 2 MCU GPIO'suna bağlanacak — INT ile polling yerine kesme tabanlı algılama yapılabilir, SHUT ile I2C askıda kalırsa yazılımsal reset atılabilir. Tek ToF sensörü olduğu için adres çakışması amaçlı kullanmaya gerek yok ama bağlamak ucuz bir esneklik.
  - **Zorunlu: 1x boş/genişletme I2C JST header'ı** (4-pin) — gelecekte ek sensör bağlamak için, kullanıcı tarafından açıkça istendi.
- Stokta hem 4-pin (20 çift) hem 6-pin (10 çift) JST bol — kısıtlı değiliz.

## LDR (ortam ışığı sensörü)

- **Karar: board üstü değil, JST 2.0mm ile harici bağlanacak.** LDR pasif bir 2 uçlu direnç olduğu için **JST 2-pin yeterli** (sadece LDR'nin iki ucu dışarı çıkıyor).
- Voltage divider'ın sabit direnci ve ADC bağlantı noktası **mainboard üzerinde** kalacak (3.3V → sabit direnç → ADC pini tap noktası → JST üzerinden dışarıdaki LDR → GND). Yani divider'ın yarısı board'da, LDR'nin kendisi dışarıda.
- Amaç adayı: ekran arka ışığı ve/veya RGB LED parlaklığının ortam ışığına göre otomatik ayarlanması (yazılımsal, backlight zaten PWM/GPIO ile kontrol edilecek — bkz. Ekran bölümü).

## TTP223 Kapasitif Dokunmatik Sensör (x2)

- 2 adet kullanılacak şekilde planlanmış — "desk buddy" ile dokunmatik etkileşim (örn. gövdeye/kafaya dokunma algısı) için muhtemel.
- **Karar: JST 2.0mm ile bağlanacak** (board üstü değil) — her biri VCC/GND/OUT (3 pin), JST 2.0mm 3-pin kullanılacak.

## LED

- Kart üstünde **1 (mümkünse 2) adet WS2812B** (MCU2812B SMD parça + mevcut şeritten sökülecek 1 adet).
- **Harici LED çıkışı:** 1x JST 2.0mm (3-pin: VCC, GND, DATA), WS2812B veri hattı — kısa bir seri zincir (~4 LED) için, uzun şerit bağlanmayacak.
- **Veri hattı topolojisi — karar: kart üstü LED(ler) ve harici çıkış AYRI GPIO'larda, iki bağımsız zincir.**
  - WS2812B'ler tek bir veri hattında zincirlenebiliyor (her LED, aldığı veriyi işleyip gerisini bir sonrakine aktarıyor) — bu yüzden "tek hat mı ayrı GPIO mu" diye bir soru vardı: kart üstü LED'ler + harici JST'deki LED'ler TEK bir zincirde (MCU → onboard LED(ler) → JST → harici LED'ler, tek GPIO) mi olacak, yoksa İKİ ayrı zincirde (biri onboard, biri harici, iki GPIO) mi olacak.
  - **Neden ayrı GPIO önerdim:** ESP32-S3'te GPIO bol, maliyeti yok. Ayrı olursa: (a) harici kabloya bir şey olursa (kopuk/kısa devre) kart üstü gösterge LED'i etkilenmez, (b) firmware'de "durum göstergesi" (onboard) ile "dekoratif aydınlatma" (harici) birbirinden bağımsız kontrol edilir — örn. biri sabit renk gösterirken diğeri animasyon yapabilir. Tek zincir olsaydı bu bağımsızlık kaybolurdu (hepsi aynı veri akışını paylaşır).
  - Harici hat uzun olmayacağı için (kullanıcı zaten "uzun şerit bağlamayacağım" demişti) sinyal bütünlüğü sorun olmaz; yine de veri hattına yakın bir yere küçük bir seri direnç (~330Ω) eklemek iyi pratiktir.
- **0805 kırmızı/yeşil LED'ler:** charger CHG/PGOOD pinlerine bağlanacak (şarj durumu / power-good göstergesi) — bağlantı: OUT → 1.5kΩ → LED → CHG (kırmızı) / PGOOD (yeşil), bkz. Güç bölümü.

## Ses

- **MAX98357A I2S 3W Amplifikatör Modülü** (satıcı spec sayfasıyla tam teyit edildi) — LM358 analog op-amp fikrinin yerini aldı (LM358 muhtemelen arkadaşta, ayrıca ihtiyaç da kalmadı — bkz. BOM "Tasarımdan çıkarılan parçalar").
- **Modül spekleri:** Class D, VDD 2.5-5.5V, I2S dijital giriş (8-96kHz), çıkış gücü 3.2W@4Ω / 1.8W@8Ω (bu değerler muhtemelen 5V'ta ölçülmüş — biz pil gerilimiyle (3.0-4.2V) besleyeceğimiz için gerçek çıkış gücü bir miktar daha düşük olacak, kabaca (VBAT/5V)² oranında, örn. ~3.7V'ta ~%55 güç ≈ 1.7W@4Ω — küçük bir hoparlör için hâlâ bolca yeterli), mono, çıkış empedansı 4-8Ω, boyut 18x19mm.
- **Pin planı (7 pin: LRC, BCLK, DIN, GAIN, SD, GND, VCC):**

| Pin | İşlev | Karar |
|---|---|---|
| VCC | Güç (2.5-5.5V) | **OUT/pil rayına** (3.0-4.2V) — böylece USB yokken de (pilden çalışırken) ses çalışmaya devam eder |
| GND | Toprak | GND |
| BCLK | I2S bit clock | ESP32-S3 I2S BCLK |
| LRC | I2S frame/word select | ESP32-S3 I2S WS/LRCLK |
| DIN | I2S data | ESP32-S3 I2S DOUT |
| GAIN | Kazanç seçimi (analog-sense pin: GND/100k-GND/float/100k-VDD/VDD = 5 farklı kazanç seviyesi) | **Modülde NC bırakılacak (float = tipik 15dB, en yaygın varsayılan)** — JST'ye dahil edilmeyecek, sadece VCC/GND/BCLK/LRC/DIN/SD çıkacak |
| SD | Kapanma + mono kanal seçimi (aynı analog-sense mantığı: float = (L+R)/2 mixed mono @ normal gain — bizim için ideal varsayılan) | **ESP32-S3 GPIO'ya bağlanacak, ama push-pull sürülmeyecek:** GPIO varsayılan olarak **INPUT/Hi-Z** bırakılır (float davranışı = amp açık, mixed mono) — mute/kapatma istendiğinde firmware GPIO'yu **OUTPUT LOW** yapar (shutdown). Bu, tek bir GPIO ile hem "aç/kapa" hem de doğru mono mix davranışını float durumunun avantajından ödün vermeden sağlıyor. |

- **JST bağlantısı — karar: 6-pin JST** (VCC, GND, BCLK, LRC, DIN, SD) — GAIN modülde NC kaldığı için JST'ye dahil değil. Hoparlörün kendisi mainboard'a değil, **doğrudan amp modülünün çıkış terminaline** bağlanacak (mainboard sadece I2S + güç + SD yolluyor, ses sinyali mainboard'a hiç girmiyor).
- **Hoparlör seçimi:** Modül 4-8Ω hoparlör bekliyor — elimizdeki fiziksel hoparlörün empedansının bu aralıkta olduğu (etiketten/satın alma kaydından) doğrulanmalı.

## Titreşim motoru

- **Mini titreşim motoru, 3V, şaftsız, 10x3.4mm** — PCB'ye direkt lehimlenmeyecek, JST 2.0mm (2-pin) ile harici bağlanacak.
- Sürücü: **AO3400A** (N-kanal MOSFET, low-side switch), gate MCU GPIO/PWM ile sürülecek.
- Flyback diyot eklenmesi öneriliyor (küçük Schottky, örn. BAT54) — BOM'da yok, ayrıca temin edilecek. Motor küçük/coreless olduğu için endüktans düşük, risk düşük ama yine de eklenmesi iyi pratik.

## Bellek ve diğer lojik IC'ler — tasarımdan çıkarıldı

AT24C02 EEPROM, HEF4070BT (quad XOR), LM555 timer ve **LM358AP** artık bu tasarımda **kullanılmayacaklar**. İlk üçü elimizde değil (arkadaşa verilmiş). LM358AP ise hem muhtemelen arkadaşta hem de artık işlevsel olarak gereksiz — MAX98357A onun yerini aldı (bkz. Ses bölümü). Orijinal tasarımdaki rolleri hatırlanamadığı için (ve elimizde olmadıkları için) yeniden değerlendirmeye gerek yok — gerekirse ileride ayrı bir revizyonda tekrar satın alınıp eklenebilirler.

## Mekanik

- 1x M3 vida deliği (montaj) — kart üzerinde konumu, gövde/kutu tasarımı varsa ona göre belirlenecek.

- [x] **ISET/ILIM direnç değerleri kesinleşti:** RISET=1.78kΩ (500mA, ilk plan) / 1.13kΩ (800mA, opsiyonel — ikisi de sipariş edilecek, montajda seçilecek), RILIM=1.18kΩ (IIN-MAX≈1.3A) — %1 tolerans
- [x] PX10303S 2 telli (siyah/kırmızı), NTC yok → **TS pinine 10kΩ sabit direnç TS→VSS**
- [x] PX10303S'te dahili BMS/PCM var (üzerinde görünüyor) → ayrı koruma devresi gerekmiyor, firmware'de opsiyonel low-battery UX önerilir
- [x] Power butonu: MCU GPIO + firmware deep-sleep (SYSOFF sabit GND kalıyor, hardware latch gerekmiyor)
- [x] SYSOFF: **GND'ye sabit** — bu zaten kesinleşmişti (pin tablosu satır 15, ve "SYSOFF (Section 10.2.2.1.2)" notu). Buradaki eski "açık" madde benim unutup silmediğim bir kalıntıydı, düzeltildi.
- [x] WS2812B veri hattı: **kart üstü LED(ler) ve harici çıkış AYRI GPIO'larda**, iki bağımsız zincir (bkz. LED bölümü)
- [x] TTP223'lerin bağlantısı: **JST 2.0mm ile** (board üstü değil)
- [x] LDR: **board üstü değil, JST ile harici bağlanacak** (bkz. LDR bölümü)
- [x] LCD backlight kontrolü: **yazılım/PWM ile kontrol edilecek** (bkz. Ekran bölümü)
- [x] MAX98357A: **JST 6-pin ile bağlanacak** (VCC/GND/BCLK/LRC/DIN/SD — GAIN modülde NC/float, 15dB varsayılan), SD pini GPIO'ya tri-state kontrollü bağlanacak
- [ ] Fiziksel hoparlörün empedansının 4Ω veya 8Ω olduğu doğrulanacak (MAX98357A bu aralığı bekliyor)
- [ ] Harici UART çıkışı (TX/RX) — önceki karttan hatırlanıyor, pin bütçesine eklendi, uygun pin kalıp kalmadığı genel pin atamasında görülecek
- [ ] CE için hangi ESP32-S3 GPIO'su ayrılacak — genel ESP32-S3 pin atama işine dahil, strapping pini (GPIO0/3/45/46 gibi boot-modu belirleyen pinler) seçilmeyecek
- [ ] 3.3V rayının toplam akım bütçesi (WS2812B sayısına göre ayrı 5V hattı gerekebilir mi)

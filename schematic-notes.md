# Şematik Tasarım Notları

Bu dosya, kaybolan orijinal tasarımın hafızadan + elimizdeki BOM'dan (`BOM.md`) yeniden kurgulanan mimarisini ve açık kalan kararları takip eder. KiCad'de şematik çizimine başlamadan önce burada netleştirilecek.

## Blok diyagramı (taslak)

```
LiPo Pil (PX10303S 3.7V 1000mAh) ──> BQ24075RGTR (VQFN-16) ──> OUT (DPPM) ──┬─> AP2112K-3.3 LDO ──> 3.3V rayı ──> ESP32-S3, LCD+touch, sensörler
                 ^                                                         │
USB-C (USB4110) ─┘ IN (VBUS, şarj girişi)                                  ├─> WS2812B LED(ler) [3.3V değil, OUT'tan — güç bütçesi kararı]
   │                                                                       ├─> MAX98357A (I2S) -> Hoparlör (JST)
   USBLC6-2SC6 (D+/D- ESD)                                                 ├─> AO3400A -> Titreşim motoru (JST)
                                                                            └─> AO3400A -> LCD Backlight (LED pini, VCC lojik hattı ayrı/3.3V'ta)

ESP32-S3 ── I2C0 (GPIO8/9) ── JST 2.0mm header'lar ── VL6180X (TOF050C), AHT20+BMP280, MPU6050
ESP32-S3 ── I2C1 (GPIO42/47) ── 2x JST 2.0mm header (harici genişletme bus'ı, izole)
ESP32-S3 ── tek hat (data, GPIO-A) ── kart üstü WS2812B(ler)
ESP32-S3 ── tek hat (data, GPIO-B, ayrı) ── JST 2.0mm header ── harici WS2812B (~4 LED, seri)
ESP32-S3 ── dijital giriş x2 (JST) ── TTP223 x2 (dokunmatik buton)
ESP32-S3 ── ADC (JST) ── LDR (harici)
ESP32-S3 ── ADC (GPIO3, divider) ── BAT (pil voltajı sense)
ESP32-S3 ── UART0 TX/RX (GPIO43/44) ── harici genişletme çıkışı (JST/header)
ESP32-S3 ── kullanılmayan GPIO ── GPIO48 (tek kalan, header/pad)
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
| 4 | CE | I | **ESP32-S3 GPIO39'a** (aktif-low, charge enable/disable — firmware kontrolü). İçeride ~285kΩ pull-down var, GPIO input/Hi-Z bırakılırsa varsayılan **şarj etkin** olur (güvenli varsayılan). |
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
| 16 | ISET | I/O | **RISET = 1.78kΩ** (→ ICHG ≈ 500mA / 0.5C, varsayılan; şematikte R7). 800mA istenirse 1.13kΩ (TI'nin kendi worked example değeri) opsiyonel — hesap aşağıda. **Boşta bırakılırsa şarj tamamen devre dışı kalır.** %1 tolerans şart. Şarj sırasında bu pindeki gerilim akımla orantılı — isteğe bağlı ESP32 ADC pinine bağlanıp telemetri olarak okunabilir. |
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

- **AP2112K-3.3TRG1:** Ana 3.3V rayı için LDO, 600mA — **detaylı güç bütçesi analizi aşağıda, ayrı bölümde.**
- **USBLC6-2SC6:** USB-C D+/D- hatlarında ESD koruması.
- **USB-C CC1/CC2:** Sadece USB 2.0 / 5V için pull-down dirençler gerekiyor (genelde 5.1kΩ). Dirençler BOM'da yok ama zorunlu, unutulmamalı.
- **AO3400A (N-kanal MOSFET, SOT-23-3):** Low-side sürücü olarak 2 yerde kullanılıyor — titreşim motoru ve LCD backlight (ikisi de MCU GPIO/PWM ile gate sürülüyor). 10 adet alınmış, 2/10 kullanılacak, gerisi yedek.

### 3.3V Güç Bütçesi — AP2112K-3.3TRG1 (600mA) [ÇALIŞILDI]

**AP2112K-3.3 gerçek datasheet değerleri** (Diodes Inc/BCD Semiconductor AP2112, web araması ile teyit edildi):
- Dropout @300mA: 125mV tip / 200mV max — Dropout @600mA: 250mV tip / 400mV max
- Iq: 55µA tip (yük yokken), Standby (EN=low): <1µA

**ESP32-S3 WiFi TX peak akımı: ~500mA** (Espressif'in kendi resmi rakamı — µs-ms mertebesinde, sık tekrarlayan patlamalar).

**Yük tablosu (3.3V raydan besleniyor, ORİJİNAL PLAN):**

| Yük | Yaklaşık akım | Süreklilik |
|---|---|---|
| ESP32-S3 (WiFi bağlı, patlama dışı) | ~100mA | Sürekli |
| ESP32-S3 WiFi TX patlaması | ~500mA (toplam, yukarıdakinin üstüne eklenmez, chip'in o andaki toplam çekişi) | µs-ms, kısa |
| ILI9341 kontrolcü lojiği (VCC) | ~25mA | Sürekli |
| ILI9341 backlight (LED pini, tam parlaklık) | ~80-120mA | Sürekli |
| WS2812B kart üstü x2 (tam beyaz) | ~120mA | Sürekli (statik renkte) |
| WS2812B harici x4 (tam beyaz) | ~240mA | Sürekli |
| Touch + ToF + AHT20/BMP280 + MPU6050 + TTP223 x2 + pil sense | ~12mA | Sürekli/düşük |

**Sürekli taban yük (WiFi patlaması hariç, ekran+LED'ler açık): ~600mA** — limitin tam kenarında, WiFi hiç devreye girmeden.
**WiFi TX patlaması sırasında anlık toplam (100→500mA delta, +400mA): ~1000mA — limitin ~%67 üzerinde. BURADA PATLIYORUZ.**

**Sebep:** WS2812B'ler (özellikle harici 4'lü zincir, 240mA) ve backlight (80-120mA), MAX98357A ve titreşim motorunun aksine (onlar zaten OUT/pil rayından besleniyor), 3.3V rayına bağlı kalmıştı — unutulmuştu.

**Karar/Çözüm (ek parça gerekmiyor, sadece kablolama):**

1. **WS2812B'ler (kart üstü + harici) artık OUT/pil rayından beslenecek** (MAX98357A/motor ile aynı mantık, VCC net değişikliği — GPIO/veri hattı değişmiyor). Elektriksel gerekçe: WS2812B "high" eşiği 0.7×VDD; pil aralığında (3.0-4.2V) bu 2.1-2.94V arası kalır, ESP32'nin 3.3V GPIO çıkışı bunun her zaman üzerinde — seviye kaydırıcı gerekmez. Bu tek değişiklik 360mA'i LDO'dan alır.
2. **Backlight (LED pini) de OUT/pil rayına taşındı — teyit edildi ve netleşti** (satıcı spec sayfası: "Güç Girişi: 3.3V veya 5V", "Arka Aydınlatma: 4 beyaz LED"). LED, VCC'den (lojik/SPI besleme) ayrı bir pin, tek yönlü bir yük (ESP32'ye sinyal geri göndermiyor) — bu yüzden pil rayına taşınması güvenli. **VCC (lojik) ise kasıtlı olarak 3.3V'ta bırakıldı** — SDO(MISO)/T_DO hatları ESP32'ye geri sinyal gönderdiği için, VCC pil gerilimini görürse bu geri dönen sinyaller ESP32 GPIO'sunun güvenli sınırını aşabilir (modülün seviye kaydırma detayı teyit edilmeden riske atılmadı). Ayrıca **LED pinini direkt GPIO'dan sürme planı düzeltildi** — 80-120mA bir GPIO'nun kaldırabileceğinden fazla, artık AO3400A ile low-side switch olarak sürülüyor (bkz. Ekran bölümü). Bu iki adımla backlight'ın ~100mA'i de 3.3V rayından kalktı.
3. **Sonuç: yeni sürekli taban ~140mA** (ESP32 ort.~100 + display lojik~25 + sensörler/touch~15), **WiFi patlamasıyla ~540mA** — 600mA limitinin rahat altında, sağlıklı marj.
4. **Bulk kapasitör eklenmeli:** ESP32-S3'ün VDD3P3 pinlerine yakın **22-47µF** (Espressif'in kendi önerisi; seçilen: **47µF, 10V X5R, 1206** seramik, şematikte C6) — WiFi'nin µs seviyesindeki akım patlamalarını LDO'nun tepki hızından bağımsız karşılamak için. Bu olmadan WiFi aktifken rastgele reset/brownout riski var (bilinen bir ESP32 arıza modu).

**Bonus — düşük pil eşiğiyle çapraz doğrulama:** Dropout ~125mV @300mA'ten, rayın düzgün 3.3V vermeye devam etmesi için pilin en az **~3.43V** olması gerektiği çıkıyor. Bunun altında ray pilin gerilimini takip ederek düşer — pil BMS kesme noktasına (~3.0V) gelmeden önce. Bu, önceki turlarda konuşulan "~3.4V'ta firmware uyarı/kapanma eşiği" fikrini hem hücre sağlığı hem LDO regülasyon sınırı açısından doğruluyor.

**Aksiyon gerektirmeyen ekstra not:** MAX98357A (yüksek seste ~1-1.5A anlık pik) + WS2812B (artık pilden) + motor + WiFi hepsi tam aynı anda denk gelirse pilden toplam çekiş ~2A'ya yaklaşabilir — küçük 1000mAh hücrenin sınırlarını zorlayabilir (tam deşarj spek'i bilinmiyor). Şimdilik bilgi amaçlı.

## MCU — ESP32-S3-WROOM-1-N8R2

- 8MB flash, 2MB PSRAM (quad SPI — WROOM-1 varyantı, WROOM-2/octal değil, dolayısıyla PSRAM için ekstra GPIO rezerve edilmiyor, standart GPIO haritası geçerli).
- **Boot / Reset / Power:** 1x tactile switch Boot (GPIO0, pull-up + PCB'de "boot" ipucu), 1x tactile switch Reset (EN pinine), **1x tactile switch Power (açma/kapama)**.
  - **Power butonu tasarımı:** Datasheet'in SYSOFF ile pil bağlantısını kesme referans devresi (Figure 10-13, "Using BQ24075 to Disconnect the Battery From the System") SYSOFF'u bir **host/MCU** çıkışına bağlıyor, mekanik bir butona değil — donanımsal bir "latch" (kilitleme) devresi olmadan tek bir momentary butonla SYSOFF'u sürmek pratik değil. Bunun yerine **önerilen: SYSOFF sabit GND'de kalır (plandaki gibi), power butonu bir MCU GPIO'suna bağlanır ve firmware ESP32-S3'ü deep sleep'e alıp/uyandırarak "yazılımsal açma/kapama" yapar** (ESP32-S3 deep sleep akımı çok düşük, ~onlarca µA). Gerçek donanımsal sıfır-akım kapatma istenirse SYSOFF tabanlı bir latch devresi (ekstra transistör/diyot) ileride ayrı bir görev olarak eklenebilir, ama bu proje kapsamında gerekli değil.
  - **Harici güç butonu / GPIO2 çıkışı (J18, `PWR_BTN`):** Power butonunun yanına 3 pinli JST eklendi, pin sırası TTP konnektörleriyle aynı: **1 = 3V3, 2 = GND, 3 = GPIO2 (ESP32_POWER)**. Butonu kasanın dışına çıkarmak istersen kablo J18'e takılır (pin 2-3 arası anahtar), kartın SMD buton pad'lerine lehim gerekmez. Butonu hiç kullanmayıp GPIO2'yi başka bir iş için kullanacaksan SW3'ü lehimleme; R12 (10k pull-up→3V3) hatta kalır, gerekmiyorsa onu da lehimleme. Boş kalan J18'e kısa devre riski için boş bir JST soketi takılabilir.
  - **Uyarı (güç tüketimi):** Bu yöntem "yumuşak kapatma"dır (derin uyku). ESP32 dışındaki yükler (WS2812B'ler pil hattında, MAX98357A SD pini yüzerken açık, MPU6050 vb. 3V3'te) uyku sırasında da çalışmaya devam eder; gerçek bir kapatma için pil kablosuna seri (≥3A) anahtar kullanılabilir.
### Pin ataması — kesinleşti (ESP32-S3-WROOM-1-N8R2, Table 2, `datasheets/ESP32-S3-WROOM-1_1U_v0.5.1_Preliminary.pdf`)

**Strapping pinleri (datasheet Section 3.3, Table 3 ile teyit edildi) — sadece 4 tane: GPIO0, GPIO3, GPIO45, GPIO46.** GPIO0 zaten Boot butonu için kullanılıyor (standart/kasıtlı). GPIO45/46'ya **hiçbir şey bağlanmayacak** (boş/NC) — reset anında yanlış seviyede sürülürlerse flash voltajı/boot mesaj davranışı bozulabilir, dokunmamak en güvenlisi.

**GPIO3 istisna — pil voltajı ölçümüne ayrıldı.** Datasheet'in birebir metni: *"GPIO3 is floating by default. When EFUSE_STRAP_JTAG_SEL is set, the strapping value of GPIO3 determines the source of the JTAG signal... When EFUSE_STRAP_JTAG_SEL is 0, the JTAG signal comes from the USB Serial/JTAG controller."* Bu eFuse fabrika/stok modüllerde yakılı değil (biz de yakmayacağız) — yani GPIO3'ün strap değeri varsayılan durumda **tamamen görmezden geliniyor**, JTAG kaynağı zaten sabit olarak USB Serial/JTAG kontrolcüsünden geliyor. Bu da GPIO3'ü, diğer üç strapping pininden farklı olarak, normal bir GPIO/ADC pini gibi güvenle kullanılabilir hale getiriyor.

- **Pil voltajı sense devresi:** BAT hattından (charger'ın BAT pini/pilin + ucu) 100kΩ + 100kΩ dirençli bir voltage divider (2:1 oran) → orta nokta GPIO3'e (ADC1_CH2). VBAT=4.2V'ta ADC ~2.1V okur (güvenli aralıkta). Sürekli çeker ama çok az (~21µA, 1000mAh pil için pratikte ~5+ yıl sürer — pilin kendi self-discharge'inden bile düşük). Bu değerler Adafruit Feather gibi yaygın LiPo+ESP32 kartlarında kullanılan standart yaklaşımla aynı.
- Bu, önceki "pil düşük gerilimde kendi kendine kapatma" fikrini donanımsal olarak mümkün kılıyor — firmware ADC'den okuyup eşik altına inince graceful shutdown/uyarı tetikleyebilir.

**USB (native, GPIO19=USB_D-, GPIO20=USB_D+) ve UART0 (GPIO43=U0TXD, GPIO44=U0RXD) donanımsal varsayılan pinler — bunlar bizim için hazır ve idealdi:**
- USB-C D+/D- → USBLC6-2SC6 → GPIO20/GPIO19 (programlama + native USB CDC seri, harici köprü chip gerekmiyor)
- **Kod yükleme doğrulaması:** Köprü chip (CP2102/CH340) yok, buna gerek de yok — ESP32-S3'ün dahili "USB Serial/JTAG" donanımı esptool ile otomatik bootloader geçişini native USB üzerinden kendi başına yapıyor (DTR/RTS'li transistör devresi gerekmiyor). Bunun çalışması için gereken varsayılan konfigürasyon (eFuse yakılmamış hali) zaten bizim durumumuz — bkz. GPIO3 notundaki aynı doğrulama. Tek şart: USB-C CC1/CC2 pull-down dirençleri (5.1kΩ) olmazsa olmaz, yoksa bazı host'lar (USB-C'den USB-C modern laptoplar gibi) VBUS bile vermeyebilir. Bu yaklaşım Adafruit QT Py ESP32-S3, Seeed XIAO ESP32-S3 gibi köprüsüz native-USB kartlarla aynı, kanıtlanmış bir yöntem.
- **UART genişletme çıkışı** (kullanıcının hatırladığı "dışarıya UART" isteği) → GPIO43(TX)/GPIO44(RX). Bunlar zaten donanımsal UART0 varsayılanı olduğu için firmware'de pin remap gerekmiyor, hatta ROM bootloader boot mesajlarını da buradan basıyor — debug için bonus.

**Tam atama tablosu:**

| GPIO | Fonksiyon | Not |
|---|---|---|
| 0 | BOOT butonu | Strapping — kasıtlı kullanım |
| 1 | LDR (ADC1) | ADC1 aralığı (GPIO1-10), WiFi aktifken güvenilir |
| 2 | POWER butonu | RTC-capable (deep sleep EXT0/EXT1 wake) |
| 3 | Pil voltajı (ADC1_CH2) | Strapping ama **güvenli** — bkz. not aşağıda |
| 4 | LCD Backlight — AO3400A gate (PWM/LEDC) | Direkt LED pinine değil, MOSFET gate'ine — backlight akımı bir GPIO'nun kaldıramayacağı kadar yüksek (~80-120mA) |
| 5 | WS2812B — kart üstü data | Ayrı zincir |
| 6 | WS2812B — harici JST data | Ayrı zincir |
| 7 | I2S DIN (MAX98357A) | |
| 8 | I2C0 SDA | ToF + AHT20/BMP280 + MPU6050 (iç/sabit sensör bus'ı) |
| 9 | I2C0 SCL | " |
| 10 | LCD CS | FSPICS0 (donanımsal SPI2 pini) |
| 11 | LCD MOSI (SDI) | FSPID — Touch T_DIN de aynı hatta paralel bağlanacak |
| 12 | LCD SCK | FSPICLK — Touch T_CLK de aynı hatta paralel bağlanacak |
| 13 | LCD MISO (SDO) | FSPIQ — **artık zorunlu** (touch eklendiğinde T_DO konum verisini bu hattan okuyor, öncesinde opsiyoneldi) |
| 14 | LCD DC | |
| 15 | I2S BCLK (MAX98357A) | |
| 16 | I2S LRC (MAX98357A) | |
| 17 | ToF SHUT | |
| 18 | ToF INT | |
| 19 | USB_D- | Native USB — bize ayrılmış, başka amaçla kullanılmayacak |
| 20 | USB_D+ | " |
| 21 | LCD RESET | |
| 35 | I2S SD (MAX98357A, tri-state) | Quad PSRAM modülünde serbest (OPI değil) |
| 36 | Titreşim motoru (AO3400A gate, PWM) | |
| 37 | TTP223 #1 OUT | |
| 38 | TTP223 #2 OUT | |
| 39 | Charger CE | Strapping değil, güvenli |
| 40 | Touch T_CS | Ekranla aynı SPI hattını paylaşıyor, kendi CS'i |
| 41 | Touch T_IRQ | Dokunma kesmesi (polling yerine) |
| 42 | I2C1 SDA | **Harici genişletme bus'ı** (2x JST konnektör, aynı bus'ta paralel) |
| 43 | UART TX (U0TXD) | Donanımsal varsayılan |
| 44 | UART RX (U0RXD) | Donanımsal varsayılan |
| 45 | — (BOŞ, NC) | Strapping (VDD_SPI voltage) |
| 46 | — (BOŞ, NC) | Strapping (boot mesaj kontrolü) |
| 47 | I2C1 SCL | **Harici genişletme bus'ı** (yukarıdaki ile eş) |
| 48 | **BOŞ — genişletme header** | Tek kalan serbest pin |
| EN | RESET butonu | GPIO değil, ayrı dedike pin |

**Kullanılmayan GPIO genişletme header'ı: sadece GPIO48 kaldı** (touch + ikinci I2C bus eklenince 5 boş pinin 4'ü doldu). Tek pin az geliyorsa ileride LCD MISO/Touch DO paylaşımı gibi bir alandan geri çalınabilir ama şu an gerek yok.

**Not — ISET telemetri artık mümkün değil (kapandı):** GPIO13 (LCD MISO) touch eklenince zorunlu hale geldi, ISET için "boşaltılabilir" yedek pin olma özelliğini kaybetti. ADC1/ADC2 aralıkları tamamen dolu — ISET telemetrisi istenirse ileride başka bir pinden (örn. TTP223 birinden) fedakarlık gerekir. Şimdilik plana dahil değil.

## Ekran — 2.8" ILI9341 dokunmatik

- SPI arayüz, 240x320. **Pinout modül üzerindeki silkscreen'den teyit edildi** (14 pin toplam): T_IRQ, T_DO, T_DIN, T_CS, T_CLK (touch grubu), SDO(MISO), LED, SCK, SDI(MOSI), DC, RESET, CS, GND, VCC.
- **Karar: Dokunmatik (touch) eklendi (XPT2046 tipi, pinout'a dahil edildi — gerekirse sonradan çıkarılabilir, sadece 2 GPIO ve birkaç iz maliyeti).**
  - T_CLK, T_DIN, T_DO ana ekranın SPI hattıyla (SCK/MOSI/MISO) **paylaşımlı/paralel** bağlanacak — modülün üzerinde bu ikisi ayrı pin olarak çıktığı için mainboard'da aynı nete lehimlenecekler.
  - T_CS → GPIO40 (touch'a özel chip select), T_IRQ → GPIO41 (dokunma kesmesi, polling yerine kullanılabilir).
  - Bu paylaşım nedeniyle **MISO (GPIO13) artık zorunlu** (öncesinde touch yokken opsiyoneldi) — touch konum verisi T_DO/MISO hattından okunuyor.
- **Kullanılacak pinler (toplam 11 adet — 9 ekran + 2 touch-özel):** VCC, GND, CS, RESET, DC, SDI(MOSI), SCK, SDO(MISO) [artık zorunlu], LED (backlight), T_CS, T_IRQ (T_CLK/T_DIN/T_DO ayrı pin harcamıyor, mevcut SPI hattına paralel).

**Güç kaynağı — VCC ve LED ayrı ele alındı (satıcı spec sayfası: "Güç Girişi: 3.3V veya 5V", "Arka Aydınlatma: 4 beyaz LED"):**

- **VCC (lojik/SPI besleme): 3.3V rayında kalıyor, OUT/pil rayına TAŞINMIYOR.** Modül geniş gerilim aralığını kabul etse de, SDO(MISO) ve T_DO hatları ESP32'ye **geri sinyal gönderiyor** (çift yönlü SPI). VCC pil gerilimini (4.2V'a kadar) görürse, bu geri dönen sinyallerin seviyesi VCC'yi takip edip ESP32-S3 GPIO'sunun güvenli giriş sınırını aşabilir — modülün dahili bir seviye kaydırıcısı olup olmadığı (ve varsa MCU tarafını gerçekten 3.3V'a sabitleyip sabitlemediği) teyit edilmeden bunu riske atmamak en doğrusu. Düşük risk/yüksek getiri olmadığı için **muhafazakar seçim: VCC sabit 3.3V.**
- **LED (backlight): ayrı bir pin, VCC'den bağımsız — OUT/pil rayına taşınabilir (güvenli, tek yönlü bir yük, ESP32'ye sinyal geri göndermiyor).**
- **Düzeltme — önceki plan hatalıydı:** "LED pini direkt bir GPIO'ya bağlanıp PWM yapılacak" planı **yanlıştı** — 4 LED'lik backlight ~80-120mA çekiyor, bir ESP32-S3 GPIO'su bunu güvenle sağlayamaz (pin başına pratik/mutlak sınır ~20-40mA). **Düzeltilmiş tasarım: titreşim motorundakiyle aynı desen** — bir **AO3400A** (stoktaki 10 adetten biri daha, motorla birlikte 2/10 kullanılmış olur) LED'in dönüş yolunda low-side switch olarak, gate'i GPIO4'ten (LEDC/PWM) sürülüyor, LED'in gerçek gücü OUT/pil rayından geliyor. Flyback diyoda gerek yok (LED endüktif bir yük değil, motorun aksine).
- Bu düzeltme, güç bütçesindeki ~80-120mA'lik backlight yükünü de 3.3V rayından tamamen kaldırıyor (bkz. Güç bölümü "3.3V Güç Bütçesi").
- **Konnektör kararı (kesinleşti): JST 2.0mm, diğer sensörlerle aynı yöntem.** Modülün kendi 2.54mm pitch header'ı (lehimli pinleri) kullanılmayacak — bunun yerine ekranın arkasındaki pad'lere doğrudan kablo lehimlenip mainboard'daki JST'lere bağlanacak. Yani ekran da diğer 5 modül gibi **kartın dışında**, PCB'de sadece JST footprint'leri var — **ekran sembolü de "Exclude from board" olarak işaretlenecek** (önceki not — "direkt 2.54mm header ile bağlanacağı için footprint gerekir" — bu kararla geçersiz oldu).
- **11 pin, elimizdeki JST stoğunda 11-pin olmadığı için 2 konnektöre bölündü (4-pin + 7-pin):**

| JST | Pinler | Mantık |
|---|---|---|
| **JST-A (4-pin) "LCD Güç"** | VCC, GND, LED, RESET | Güç + açılışta bir kez tetiklenen yavaş sinyal |
| **JST-B (7-pin) "LCD SPI+Touch"** | CS, DC, MOSI, SCK, MISO, T_CS, T_IRQ | Gerçek SPI veri/saat hattı + touch'ın kendi CS/IRQ'su, tek grupta temiz tutuldu |

  Stok: 4-pin bol (20 çift), 7-pin'den 1 tanesi bu iş için ayrılıyor (5 çiftten). Tek GND (JST-A'da) her iki konnektöre de yetiyor — kısa kablo mesafesinde ayrı toprak dönüşüne gerek görülmedi.
- **Diğer satıcı bilgileri (referans):** Rezistif dokunmatik (XPT2046 tipiyle uyumlu), aktif alan 43.2x57.6mm, 65K/262K renk derinliği, dahili microSD yuvası (kullanılmayacak, pin bütçesine dahil edilmedi), modül boyutu ~50x86mm.

## Sensör genişletmesi (JST 2.0mm, I2C — 2 AYRI BUS)

**Karar: İki bağımsız I2C hattı.** ESP32-S3'ün 2 donanımsal I2C kontrolcüsü (I2C0, I2C1) var, ikisi de aynı anda bağımsız çalışabiliyor. Ayırmanın amacı: harici JST'ye takılan bir şeyde sorun (kısa devre, kötü kablolama, hot-plug) olursa iç sensörlerin bus'ı etkilenmesin.

### I2C0 — iç/sabit sensörler (GPIO8=SDA, GPIO9=SCL)

- Sensörler direkt lehimlenmeyecek, JST 2.0mm header'lar üzerinden bağlanacak. **Pinout'lar modül görsellerinden teyit edildi — ToF ve diğerleri farklı pin sayısına sahip:**

| Sensör | Modül pinleri (kendi sırası) | JST | I2C adresi |
|---|---|---|---|
| **VL6180X ToF** (TOF050C) | VIN, GND, SDA, SCL, INT, SHUT | **6-pin JST** (VCC, GND, SDA, SCL, INT, SHUT) | 0x29 |
| **AHT20+BMP280** | SCL, GND, SDA, VDD | **4-pin JST** (VCC, GND, SDA, SCL) | AHT20: 0x38, BMP280: 0x76/0x77 |
| **MPU6050** (6-eksen ivme/jiroskop, GY-521 tipi) | VCC, GND, SCL, SDA (+ opsiyonel XDA/XCL/AD0/INT, bağlanmayacak) | **4-pin JST** (VCC, GND, SDA, SCL) — modül PCB üzerinde değil, JST ile bağlanıyor; şematikte sadece JST (J8) var, modül sembolü yerleştirilmedi | 0x68 (AD0 low, varsayılan) |

  - Adres çakışması yok: 0x29 (ToF), 0x38 (AHT20), 0x76/0x77 (BMP280), 0x68 (MPU6050) — hepsi farklı (EEPROM tasarımdan çıkarıldığı için onun adresini düşünmeye gerek kalmadı).
  - ToF'un **INT** (proximity/range interrupt çıkışı) ve **SHUT** (donanımsal shutdown/reset girişi) pinleri ayrıca 2 MCU GPIO'suna bağlanacak — INT ile polling yerine kesme tabanlı algılama yapılabilir, SHUT ile I2C askıda kalırsa yazılımsal reset atılabilir.

### I2C1 — harici genişletme bus'ı (GPIO42=SDA, GPIO47=SCL)

- **2x 4-pin JST konnektör (VCC, GND, SDA, SCL), aynı bus'ta paralel** — kartın çıkışında, gelecekte harici I2C cihazı bağlamak için. Kullanıcının "dışarıya mutlaka boş I2C çıkışı" isteği artık kendi bus'ı ve 2 fiziksel konnektörüyle karşılanıyor (eskiden tek bir 4-pin "boş header" olarak I2C0'a ekleniyordu, şimdi ayrı/izole bir bus).
- Bu bus'a başlangıçta hiçbir şey lehimli değil, tamamen kullanıcı tarafından ileride bağlanacak cihazlara açık.
- **Kullanım amacı netleşti (bkz. README "Konsept"):** Bu 2 konnektör, ayrıca tasarlanacak **mini I2C gamepad'lerin** bağlantı noktası — "desk buddy + mini oyun konsolu" vizyonunun parçası.
- **Önemli — bu PCB'yi değil, gamepad tasarımını ilgilendiren bir uyarı:** İki konnektör de aynı bus'ta paralel olduğu için, iki gamepad birbirinin aynıysa (aynı sabit I2C adresi) aynı anda takıldıklarında adres çakışması olur. Gamepad kartlarında bir adres-seçim pini/jumper'ı (ADDR pini gibi) olmalı ki "Player 1" ve "Player 2" farklı adreste konuşabilsin. VCC'nin bu konnektörlerde 3.3V (regüle) olması öneriliyor — gamepad'in kendi lojik/buton devresi için OUT/pil rayının ham gerilimi yerine.

Stokta hem 4-pin (20 çift) hem 6-pin (10 çift) JST bol — kısıtlı değiliz.

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
- **Güç kararı (güç bütçesi analizinden, bkz. Güç bölümü):** WS2812B'lerin VCC'si (hem kart üstü hem harici JST) **3.3V rayı değil, OUT/pil rayından (3.0-4.2V)** beslenecek — MAX98357A ve titreşim motoruyla aynı mantık. Veri hattı (GPIO çıkışı) hâlâ 3.3V lojik, değişmiyor; sadece güç neti değişti. Bu değişiklik 3.3V LDO'nun üzerinden ~360mA'lik sürekli yükü kaldırıyor.
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
- Flyback diyot: **SS14** (Schottky, 1A 40V, SMA/DO-214AC; şematikte D5, footprint `D_SMA`) — BOM'da yok, satın alınacak. BAT54 (SOT-23) planlanmıştı, ancak sembol (2 pinli) ile SOT-23 footprint pin uyuşmazlığı vardı ve elde de yoktu. Motor küçük/coreless olduğu için endüktans düşük, risk düşük ama yine de eklenmesi iyi pratik.

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
- [x] **ESP32-S3-WROOM-1-N8R2 tam GPIO ataması kesinleşti** — bkz. MCU bölümü "Pin ataması" tablosu (33 pin kullanıldı, sadece GPIO48 boşta kaldı)
- [x] Pil voltajı ölçümü: **GPIO3'e ayrıldı** (ADC1_CH2) — strapping olmasına rağmen güvenli, datasheet'ten teyitli (eFuse yakılı değilse strap değeri görmezden geliniyor). 100kΩ+100kΩ divider BAT'tan.
- [x] Harici UART çıkışı → GPIO43(TX)/GPIO44(RX), donanımsal UART0 varsayılanı — UART1 gerekmiyor, tek bus yeterli görüldü
- [x] CE → GPIO39
- [x] Touch (XPT2046) eklendi — T_CS=GPIO40, T_IRQ=GPIO41, T_CLK/T_DIN/T_DO ana SPI hattına paralel (MISO artık zorunlu)
- [x] İki ayrı I2C bus'ı: I2C0 (GPIO8/9, iç sensörler) + I2C1 (GPIO42/47, 2x harici JST, izole)
- [x] **3.3V güç bütçesi çalışıldı** — WS2812B'ler 3.3V yerine OUT/pil rayına taşındı (360mA tasarruf), ESP32-S3 VDD3P3 yakınına 22-47µF bulk kapasitör eklenmesi kararlaştırıldı. Detay: Güç bölümü "3.3V Güç Bütçesi"
- [x] Ekranın güç girişi teyit edildi (satıcı spec: "3.3V veya 5V") — **VCC (lojik) yine de 3.3V'ta bırakıldı** (SDO/T_DO geri sinyal riski), **LED (backlight) OUT/pil rayına taşındı**, AO3400A ile MOSFET switch üzerinden — güç bütçesi hedefine (~540mA tavan) ulaşıldı

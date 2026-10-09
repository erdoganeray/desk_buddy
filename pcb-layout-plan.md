# Desk Buddy PCB Layout Planı

## Context

Şematik tamamlandı (9 sayfa, 62 kart-üstü parça, footprint'ler atanıp pin/pad uyumu doğrulandı, ERC 0 hata). `kicad/Desk Buddy.kicad_pcb` bu plan yazılırken boştu (80 byte); güncel durum aşağıdaki "Durum" bölümünde. Kullanıcının ilk PCB'si; üretim Türkiye'deki bir PCB firması (muhtemelen arkada JLCPCB), 2 katman. Kutuyu kullanıcı PCB'ye göre çizecek, yani kart boyutu/şekli bize bağlı. 2 adet M3 delik. Ekran kart dışında (JST ile kablolu). JST'ler PH 2.0mm, dik (top-entry); konnektör kenarlarında fiş için az da olsa boşluk bırakılacak.

Amaç: kartı adım adım, her adımda kullanıcının kontrol edebileceği şekilde yerleştirmek ve yollarını çizmek; sonunda DRC temiz, şematikle uyumlu, üretime hazır dosyalar.

## Durum (2026-10-09, branch `pcb-layout`)

| Adım | Durum | Not |
|---|---|---|
| 0 Hazırlık / içe aktarma | ✅ | Parity temiz, araçlar çalışıyor |
| 1 Kart kurulumu | ✅ | Kurallar, net class'lar, M3 delikleri, anten keepout'u. ESP32 termal via'ları 0.2 → 0.3mm delik |
| 2 Kat planı | ✅ | Onaylandı, `layout/step2-floorplan.png`. Kart 90x65mm'ye büyütüldü (üst sınır 100x100) |
| 2+ Şematik düzeltme | ✅ | I2C1_SCL yanlışlıkla IO45'teydi, IO47'ye taşındı |
| 3a Güç bloğu | ✅ | `layout/step3a-*.png` |
| 3b MCU bloğu | ✅ | `layout/step3b-*.png`. Şematiğe **J18** (`PWR_BTN`, 3 pin: 3V3/GND/GPIO2) eklendi |
| **3c Çevre birimleri** | ⏸ **kısmen**, arka ışık sorunu bekliyor | 63 şematik parçadan 60'ı yerleşti. **Q1, R13, R14 yerleşmedi** (aşağıda) |
| 4–9 | ⬜ | Başlamadı |

**Kaldığımız yer:** LCD arka ışık devresindeki şematik hatası, kullanıcının LCD modülünü ölçmesine bağlı. Sonuç gelince şematik düzeltilir, ilgili parçalar J4'ün yanına yerleştirilir, sonra Adım 4'e geçilir.

### Başka bilgisayarda devam ederken

- **KiCad 10.0.x** gerekli (dosya biçimi `20260206`), `kicad-cli` ve gömülü Python yolu: `C:\Program Files\KiCad\10.0\bin\`.
- **Repoda olmayanlar:** yerleştirme script'leri (`step3a/3b/3c.py`), önizleme aracı (`preview.py`), JRE 25 ve Freerouting jar'ı oturumun scratchpad klasöründeydi, repoya girmedi. Yeni oturumda yeniden yazılır/indirilir (Freerouting 2.5.0: GitHub `freerouting/freerouting` release'i; JRE: Temurin 25 x64 Windows zip).
- `.kicad_pcb` editörde açıkken script çalıştırılmaz; önce KiCad'i kapat. Elle düzenlediysen script'e başlamadan commit et.
- Yerleştirme bir optimizasyonla (iz uzunluğu + bypass/ayar pasiflerinin pinine yapışık olması) yapıldı. Beğenilmeyen parça elle taşınabilir, sorun değil.

## Çalışma şekli (araçlar ve kurallar)

Kontrol edilen araçlar (hepsi mevcut):
- **KiCad 10.0.1 gömülü Python (`C:\Program Files\KiCad\10.0\bin\python.exe`) + `pcbnew` modülü**: footprint yerleştirme, iz/via, zone, outline, silkscreen'i script ile yapabilirim.
- **`kicad-cli`**: `pcb drc` (+ `--schematic-parity`), `pcb export pdf/svg/gerbers/drill/pos`, `pcb render` (3D PNG). Çıktıları (PDF/PNG) okuyup görsel kontrol yapabilirim.
- **Freerouting**: kurulu değil, Java da yok. Kullanıcı onayladı: taşınabilir JRE + Freerouting jar'ı scratchpad altına indirilecek (sistem geneline kurulum yok).

Kurallar:
- Ben `.kicad_pcb` dosyasını script ile düzenlerken **KiCad PCB editörü kapalı olmalı** (açıksa her adımdan sonra File > Revert).
- Her adımın sonunda: görsel (PDF/PNG) + DRC özeti + "senin kontrol listen". Sen onaylamadan sonraki adıma geçmem.
- Onayınla her adımı ayrı commit olarak kaydederim (geri dönülebilir). Push'u sen yaptığında/izin verdiğinde.
- Geçici script'ler scratchpad'de tutulur; repoya sadece son `.kicad_pcb`/`.kicad_pro` ve dokümanlar girer.

## Başlangıç tasarım kuralları (ilk PCB, muhafazakâr, çoğu fab'ın kabul ettiği)

| Kural | Değer |
|---|---|
| Katman / kalınlık | 2 katman, 1.6mm, tüm parçalar üst yüzde |
| Min iz / boşluk | 0.2mm / 0.2mm (BQ24075 0.5mm-pitch bölgesinde yerel olarak 0.15mm'ye inebilir, fab onayına bağlı) |
| Sinyal izi | 0.25mm |
| Güç izleri | 3V3: 0.5mm, VBAT_OUT / motor / LED / ses: 0.6-0.8mm, USB_VBUS / BAT / OUT (şarj yolu): 0.8-1.0mm |
| Via | 0.8mm pad / 0.4mm delik (min delik 0.3mm) |
| Bakır-kenar | ≥0.3mm; delik-delik ≥0.5mm |
| Silkscreen | yazı ≥1.0mm yükseklik, çizgi ≥0.15mm |
| GND | alt katman bütün GND dökümü + üstte boş yerlerde GND dökümü + dikiş via'ları |

Net class'lar: `Default`, `Power` (geniş), `Fine` (sadece BQ24075 fan-out). Değerler `kicad/Desk Buddy.kicad_pro` içinde saklanır.

**ESP32-S3 anten kuralı:** footprint courtyard'ı (48.1x41.3mm, T şeklinde) Espressif'in önerdiği anten boşluk alanını içeriyor. Modül kartın kenarına, anten ucu kenara bakacak şekilde konacak; anten bölgesinin altında ve yanında (yaklaşık 48mm genişlik x 6mm derinlik) bakır, iz ve parça olmayacak (her iki katmanda rule-area keepout). Pil, hoparlör, motor anten tarafından uzak. Courtyard çakışma DRC'si bunu ayrıca denetler.

## Adımlar

### Adım 0: Hazırlık ve içe aktarma
- **Karar:** Fab'a sorulmayacak, aşağıdaki genel (JLCPCB uyumlu, çoğu fab'ın kabul ettiği) kurallarla ilerlenir. Yüzey HASL (kurşunsuz) varsayılır, siparişte sen seçersin.
- **Sen (yapıldı):** `Tools > Update PCB from Schematic` ile içe aktarma, kaydet, KiCad'i kapat.
- **Doğrulandı (salt-okunur):** 62 footprint (şematikle birebir, adlar uyuşuyor), 64 net, 175 bağlanmamış hat (iz yok, beklenen), outline ve zone yok.
- **Kalan işler (ben):**
  - Taşınabilir **JRE 25** (Temurin 25.0.4.1, `.zip`, SHA256 doğrulamalı; Freerouting 2.5.0 Java 25 gerektiriyor, JRE 21 ile açılmıyor) + **Freerouting 2.5.0 `.jar`** scratchpad altına indirilir (Windows'ta native CLI yok, sadece `.msi` var, sistem geneline kurulum yapmıyoruz); `java -version` ve Freerouting'in açılması test edilir. **Yapıldı, çalışıyor.**
  - `pcbnew` ile açıp-kaydetme (round-trip) testi scratchpad'deki kopyada yapılır.
  - `kicad-cli pcb drc --schematic-parity` (çıktı scratchpad'e) ile parity kontrolü.
  - KiCad'in F8 sırasında oluşturduğu `kicad/report.txt` ve `.kicad_pro` değişikliği incelenir (rapor repoya girmesin).
- **Kapı:** Parity temiz, araçlar çalışıyor.

### Adım 1: Kart kurulumu
- Design rules/net class'lar, 2 katman, 1.6mm, ızgara, silkscreen varsayılanları.
- Geçici outline: 80x60mm, 3mm köşe yarıçapı (nihai boyut Adım 4'te).
- 2x M3 delik (3.2mm delik, 6mm bakır yasak alan), karşılıklı köşelere geçici.
- ESP32 anten keepout rule-area.
- **Kapı:** KiCad'de açıp outline/delikleri görürsün; DRC çalışıyor.

### Adım 2: Kat planı (floorplan) onayı, henüz parça yok
Ölçekli bir blok çizim (PNG) hazırlarım: bölgeler ve konnektör kenarları. Önerim:
- **Güç akışı soldan sağa:** USB-C bir kenarda → USBLC6 hemen arkasında → BQ24075 → LiPo JST → AP2112K LDO.
- **ESP32** üst kenarın ortasında, anten kenara bakıyor, bypass kondansatörleri (C6/C7) 3V3 pinlerinin dibinde.
- **Konnektör grupları kenarlarda:** LCD (J4/J5) ve ToF/SPI grubu ESP32'nin ilgili pin tarafına yakın; I2C0 (J6/J7/J8) ve I2C1 (J9/J10) bir kenarda pull-up'larla; ses (J12), motor (J13 + Q2 + D5), LED (J11), TTP (J15/J16), LDR (J14), UART (J17) kalan kenarlarda.
- **3 buton:** EN ve BOOT USB tarafına yakın, POWER kasadan erişilecek bir kenara.
- **WS2812B (D1/D4):** görünür kenarda, pil rayına yakın.
- Konnektörler arası ≥2.5mm (fiş gövdesi için), kenara yakın ama fişin oturmasına yetecek payla.
- **Kapı:** Sen bölgeleri/kenarları onaylar ya da değiştirirsin.

### Adım 3: Yerleşim (3 alt adım, her biri ayrı kapı)
- **3a Güç bloğu:** USB-C, U1, U2 (BQ24075), U3 (LDO), C1-C5, R1-R9, LED'ler, J2.
- **3b MCU bloğu:** U4 (ESP32), C6/C7, R10-R12, 3 buton, J3 (GPIO48), ilgili pull-up'lar.
- **3c Çevre birimleri:** tüm JST'ler, Q1/Q2, D5, R13-R23, WS2812B'ler, test noktaları.
- Her alt adımda PDF/PNG + courtyard DRC; sen KiCad'de bakarsın.

#### 3c durumu ve açık sorun: LCD arka ışık (backlight) devresi

**Yerleşen:** tüm JST'ler (alt kenar: J8, J6, J5, J4, J7; sağ kenar: J17, J14, J9, J10; sol üst: J11, J12; iç: J15, J16, J13), motor devresi (Q2, D5, R20, R21), D1/D4 (WS2812B), R19, R23. DRC: bakır boşluk/courtyard ihlali yok, şematik parity 0 sorun, 178 bağlanmamış hat (iz yok, beklenen). Görsel: `layout/step3c-kart.png`, `layout/step3c-kart-sade.png`.

**Yerleşmeyen:** **Q1 (AO3400A), R13 (100Ω), R14 (10k)**. Şematikteki devre hatalı olduğu için, düzeltilmeden yerleştirmek anlamsız.

**Sorun nedir?** `02_Display` sayfasında Q1 şöyle bağlı (netlist ile doğrulandı):

| Q1 pini | Net |
|---|---|
| Gate (1) | `Net-(Q1-G)` (GPIO4 → R13; R14 ile GND'ye çekiliyor) |
| Source (2) | GND |
| Drain (3) | **VBAT_OUT** |

LCD'nin LED pini (J4 pin 3 → modül U5 `LED`) **de sürekli VBAT_OUT'a bağlı**. Yani:
- GPIO4 yüksek olunca Q1 iletime geçer ve **pil rayını (VBAT_OUT) doğrudan GND'ye kısa devre eder**.
- Q1 hiçbir şeyi "anahtarlamıyor": LED pini zaten sürekli pil gerilimini görüyor, MOSFET'in yükü yok.

Kaynağı: `schematic-notes.md` ("Ekran" bölümü) "LED pini OUT/pil rayına taşındı, AO3400A ile low-side switch" diyor. Bu, LED'in artı ucu pil rayında, eksi ucu ise MOSFET'e giden ayrı bir pin olsaydı doğru olurdu. Elimizdeki modülün tek bir `LED` pini var; bu pinin ne yaptığını bilmeden devre çizilmiş. (Motor devresi, Q2/D5, doğru: drain motorun alt ucunda.)

**Çözüm LED pininin türüne bağlı, modülü ölçerek öğrenilecek:**

| Tür | Anlaşılma | Doğru bağlantı |
|---|---|---|
| **A. LED pini bir kontrol girişi** (modülde küçük bir transistör var, arka ışık akımını modülün VCC'sinden alıyor) | LED pinine 3.3V verince arka ışık yanar, pine **birkaç mA** girer | GPIO4 **doğrudan** LED pinine. Q1, R13, R14 **silinir** |
| **B. LED pini arka ışığın artısı** (LED'lerin eksi ucu modülde GND'ye bağlı) | LED pinine 3.3V verince arka ışık yanar, pine **~50-120mA** girer | Low-side anahtar olmaz (eksi uç modülde GND'de). **Yüksek taraf anahtarı** gerekir: P-kanal FET (örn. AO3401A, ayrıca alınır) + onu süren N-kanal Q1 (elimizdeki AO3400A). Ya da PWM'den vazgeçip LED'i sürekli VBAT_OUT'a bağlı bırakmak (Q1/R13/R14 silinir, arka ışık hep açık) |

**Kullanıcının yapacağı ölçüm (modül başında):**
1. Modülü VCC (3.3V) ve GND'ye bağla. LED pinine multimetreyi **mA konumunda seri** takıp 3.3V ver: arka ışık yanıyor mu, kaç mA çekiyor?
2. LED pinini GND'ye çekince de dene (bazı modüllerde aktif-düşük olur), arka ışık nasıl tepki veriyor?
3. LED pininin yanında küçük bir SOT-23 transistör var mı (S8050/SS8550/J3Y gibi bir işaret)? Fotoğrafını çek.
4. Sonuçları **A / B** olarak yaz (yaklaşık mA değeriyle).

**Sonuca göre yapılacaklar:**
- **A ise:** `02_Display` sayfasında Q1, R13, R14'ü sil, GPIO4 net'ini (`LCD_BL_GATE`) J4 pin 3'e bağla, J4 pin 3'ün VBAT_OUT bağını kaldır. PCB'de aynı değişikliği yap. **Dikkat:** bu durumda arka ışık akımı (~80-120mA) modülün VCC'sinden, yani **3V3 rayından** gelir. `schematic-notes.md` güç bütçesi tavanı ~540mA → ~640mA olur, AP2112K'nın 600mA sınırını kısa süreliğine aşar (WiFi patlamasında). 47µF bulk kondansatör (C6) bunu kısmen tamponlar ama hesabı yeniden yap, gerekirse backlight'ı ayrı bir yoldan besle.
- **B ise:** yüksek taraf anahtarı devresini çiz (P-FET + Q1 sürücü + pull-up), P-FET'i `purchase-list.md`'ye ekle, parçaları J4'ün yanına yerleştir; ya da "hep açık" seçeneği.
- Her iki durumda: `schematic-notes.md` "Ekran" ve "Güç" bölümlerini düzelt, ERC çalıştır, PCB'ye yansıt (F8 yerine script ile ya da senin F8'inle), `--schematic-parity` temiz olsun, sonra Q1/R13/R14 (veya yenileri) yerleştirilip 3c tamamlanır.

### Adım 4: Yerleşim incelemesi ve outline'ı kesinleştirme
- Kartı sıkıştırıp nihai boyutu ve köşe/delik konumlarını belirlerim.
- Kontroller: USB-C kenar çıkıntısı, JST fiş boşlukları, buton erişimi, anten alanı, ısıtıcı parçaların (BQ24075/LDO) ESP32'den uzaklığı.
- **Kapı:** Sen son yerleşimi onaylarsın. Bundan sonra parça taşıma maliyetli.

### Adım 5: Zone'lar ve kritik yollar (elle)
- Alt katman GND dökümü, üstte GND dökümü, dikiş via'ları.
- BQ24075: termal pad altına via'lar, IN/OUT/BAT kondansatörleri bacağa yapışık, ISET/ILIM dirençleri yakın.
- USB D+/D-: kısa, yan yana, ESD diyotu konnektöre yakın.
- ESP32 bypass, 3V3 dağıtımı, VBAT_OUT güç omurgası (motor/LED/ses).
- **Kapı:** PDF/PNG + DRC.

### Adım 6: Sinyal yolları (Freerouting + temizlik)
- Kritik yollar kilitli; DSN dışa aktar → Freerouting → SES içe aktar.
- Sonra elle temizlik: anten alanı temiz mi, ADC hatları (VBAT_SENSE, LDR_ADC) güç yollarından uzak mı, tüm net'ler bağlı mı.
- **Kapı:** unrouted = 0, görsel inceleme.

### Adım 7: Zone doldurma ve DRC
- Zone'ları doldur, DRC'yi hatasız hale getir; uyarıları tek tek gözden geçir.
- `kicad-cli pcb drc --schematic-parity --refill-zones --severity-all`.
- **Kapı:** 0 hata, açıklanmış uyarılar.

### Adım 8: Silkscreen ve etiketleme
- Her JST için isim (ör. `LCD-A`, `ToF`, `I2C1`) ve pin-1 işareti, kritik konnektörlerde pin fonksiyonları (VCC/GND/SDA/SCL…).
- LiPo ve USB-C polaritesi (+/−), buton etiketleri (RST/BOOT/PWR), M3 delikleri, versiyon/tarih.
- **Kapı:** PDF/PNG'de okunaklılık.

### Adım 9: Son kontrol ve üretim dosyaları
- Son DRC + parity, 3D render, tüm JST'ler için **pin-1/polarite tablosu** (kablo yaparken kullanman için, repoda doküman).
- Gerber, drill, pozisyon dosyaları (`gerber/` klasörü `.gitignore`'da, zip'ler sen firmana göndermek için).
- BOM'u `purchase-list.md`/`BOM.md` ile karşılaştır; `plan.md` Faz 5 maddelerini işaretle, README durumunu güncelle.
- **Kapı:** Sen firmaya göndermeden önce son bakış.

## Değiştirilecek/oluşturulacak dosyalar

- `kicad/Desk Buddy.kicad_pcb` (asıl iş) ve `kicad/Desk Buddy.kicad_pro` (design rules, net class'lar).
- Şematik dosyalarına dokunmam; sorun çıkarsa (ör. footprint değişikliği) ayrıca sana sorarım.
- `plan.md`, `README.md`, `schematic-notes.md` (layout notları), yeni bir pin/polarite dokümanı (Adım 9).

## Doğrulama (her adımda ve sonunda)

- `kicad-cli pcb drc --schematic-parity --severity-all` (courtyard çakışması anten alanını da denetler).
- `kicad-cli pcb export pdf` ile katman görselleri → okuyup incelerim; `kicad-cli pcb render` ile 3D PNG.
- Script'le: tüm footprint'ler outline içinde mi, JST'ler kenara yakın mı, unrouted net kalmadı mı.
- Sen KiCad'de aç, 3D görünüm ve DRC'yi çalıştır; farklı bir şey görürsen o adımı tekrar ederim.

## Bilinen riskler

- Fab kuralları henüz bilinmiyor: 0.2/0.2mm muhafazakâr varsayım, firmanın değerleri farklıysa Adım 1'i güncelleriz.
- `.kicad_pcb` formatı 10.0.1: round-trip testi Adım 0'da yapılır.
- Freerouting için uygun JRE sürümünü indirirken sürüm uyumunu doğrularım.
- Anten alanı kart boyutunu etkiler: üst kenarda ~48mm x 6mm şerit boş kalacak.

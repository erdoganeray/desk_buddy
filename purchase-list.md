# Satın Alım Listesi

Fiyat ve stok bilgisi 2026-10-09'da ilgili mağaza sayfalarından okundu (KDV dahil). Değişebilir, sipariş öncesi sayfadan tekrar kontrol et.

## Satın alınacaklar

| # | Parça | Nerede kullanılıyor | Model / stok kodu | Nereden | Adet | Birim fiyat | Toplam |
|---|---|---|---|---|---|---|---|
| 1 | Tactile switch, SPST-NO, üstten basmalı, SMD 6x6mm, H4.3mm | SW1 (EN/Reset), SW2 (BOOT), SW3 (POWER) | C&K **PTS645SM43SMTR92 LFS** | [e-komponent](https://www.e-komponent.com/arama?q=PTS645) | 5 (3 + 2 yedek) | ~24.88 TL | ~124.40 TL |
| 2 | 47µF, 10V, X5R, ±20%, 1206 seramik kondansatör | C6 (ESP32-S3 VDD3P3 bulk) | Samsung, stok kodu **T12752** | [direnc.net](https://www.direnc.net/22uf-25v-10-x5r-1206-smd-kondansator) | 2 (1 + 1 yedek) | 19.76 TL | 39.52 TL |
| 3 | SS14 Schottky diyot, 1A 40V, SMA (DO-214AC) | D5 (titreşim motoru flyback) | Panjit, stok kodu **18097** | [direnc.net](https://www.direnc.net/ss14-1a-40v-schottky-diyot-do-214ac) | 5 | 1.95 TL | 9.75 TL |

**Ara toplam: ~173.67 TL** (kargo hariç).

Notlar:
- **Tactile switch:** KiCad kütüphanesinde bu modele özel footprint hazır (`SW_SPST_PTS645Sx43SMTR92`). e-komponent'te listede görünüyor, stok bilgisi sayfadan okunamadı. Datasheet: [ckswitches.com/media/1471/pts645.pdf](https://www.ckswitches.com/media/1471/pts645.pdf).
- **47µF kondansatör:** Linkteki adres "22uf-25v..." diyor ama sayfa başlığı "47uF 10V 20% x5R 1206". Sepete eklerken ürün adını kontrol et. 1206 ve 10V seçilmesinin nedeni: 3.3V rayda DC bias kaybı 0805/6.3V'a göre daha az, ayrıca el lehimi kolay. Gerçek kapasite 3.3V'ta nominalin altına düşer, Espressif'in 22-47µF önerisi için yeterli.
- **SS14:** Önceki plandaki BAT54 (SOT-23) yerine geçti. Sembol/footprint pin uyuşmazlığı vardı ve BAT54 elde yoktu. SS14 motor akımı için bol marjlı, SMA kılıfı el lehimine uygun.

## Stok kontrolü: 0805 direnç ve kondansatörler

Elde 0805 paket olduğunu söyledin. Şematikteki değerlerin hepsi var mı diye kontrol et, eksik olanı yukarıdaki listeye ekle.

| Değer | Adet | Referanslar | Not |
|---|---|---|---|
| 100Ω | 2 | R13, R20 | |
| 330Ω | 1 | R19 | |
| 1.18kΩ | 1 | R6 | RILIM, %1 tolerans önerilir |
| 1.5kΩ | 2 | R4, R5 | CHG/PGOOD LED dirençleri |
| 1.78kΩ | 1 | R7 | RISET (500mA), **%1 tolerans şart** |
| 1.13kΩ | 0 (opsiyonel) | — | RISET alternatifi (800mA), %1. Kartta yok, sadece R7 yerine takılabilir |
| 4.7kΩ | 4 | R15, R16, R17, R18 | I2C pull-up'ları |
| 5.1kΩ | 2 | R1, R2 | USB-C CC pull-down'ları |
| 10kΩ | 7 | R3, R10, R11, R12, R14, R21, R23 | R3 charger TS, R23 LDR divider |
| 100kΩ | 2 | R8, R9 | Pil voltajı divider'ı |
| 100nF | 1 | C7 | |
| 1µF | 3 | C1, C4, C5 | |
| 4.7µF | 2 | C2, C3 | |

Ayrıca 0805 kırmızı ve yeşil LED zaten BOM'da elde görünüyor (D2, D3).

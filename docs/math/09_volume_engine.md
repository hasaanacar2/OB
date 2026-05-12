# Volume Engine — Matematiksel Model

## 1. Amaç

Gerçek fiyat hareketlerini fake hareketlerden ayırmak. Hacim analizi, fiyat hareketinin arkasında gerçek bir güç olup olmadığını doğrular.

---

## 2. Volume Spike Tespiti

### 2.1 Volume SMA Karşılaştırması

```
Volume_Ratio = Volume_t / SMA(Volume, 20)
```

| Volume Ratio | Yorum |
|---|---|
| < 0.5 | Çok düşük hacim |
| 0.5 - 1.0 | Normal altı |
| 1.0 - 1.5 | Normal |
| 1.5 - 2.5 | Yüksek hacim |
| > 2.5 | Volume spike |
| > 5.0 | Aşırı spike (haber/event) |

### 2.2 Volume Z-Score

```
Vol_Z = (Volume_t - SMA(Volume, 20)) / StdDev(Volume, 20)
```

- `Vol_Z > 2.0` → Anlamlı spike
- `Vol_Z > 3.0` → Aşırı spike

### 2.3 Relative Volume

```
RVOL = Volume_t / Avg_Volume_at_same_time_of_day
```

Zaman dilimi bazlı normalizasyon (Asia, London, NY farklı ortalama hacimlere sahiptir).

---

## 3. Volume Delta

### 3.1 Tick Delta (Gerçek Zamanlı)

```
Delta = Buy_Volume - Sell_Volume
```

burada:
- `Buy_Volume` = ask fiyatında veya üstünde gerçekleşen işlemler
- `Sell_Volume` = bid fiyatında veya altında gerçekleşen işlemler

### 3.2 Candle Delta (Yaklaşık)

Tick verisi yoksa yaklaşık hesaplama:

```
if Close > Open:   # Bullish candle
    Buy_Vol ≈ Volume × (Close - Low) / (High - Low)
    Sell_Vol ≈ Volume - Buy_Vol

if Close < Open:   # Bearish candle
    Sell_Vol ≈ Volume × (High - Close) / (High - Low)
    Buy_Vol ≈ Volume - Sell_Vol
```

### 3.3 Cumulative Delta

```
CVD_t = Σ Delta_i    (i = 1 ... t)
```

**CVD Divergence:**

```
Bullish: Price lower low ∧ CVD higher low → Alıcılar güçleniyor
Bearish: Price higher high ∧ CVD lower high → Satıcılar güçleniyor
```

---

## 4. Open Interest (OI)

### 4.1 OI Hesaplama

```
OI = Total_Open_Contracts (long + short aynıdır)
```

### 4.2 OI Değişimi

```
ΔOI = OI_t - OI_{t-1}
ΔOI% = ΔOI / OI_{t-1} × 100
```

### 4.3 Price-OI İlişki Matrisi

| Price | OI | Yorum |
|---|---|---|
| ↑ | ↑ | Yeni long açılıyor → Trend güçlü |
| ↑ | ↓ | Short kapatılıyor → Trend zayıflayabilir |
| ↓ | ↑ | Yeni short açılıyor → Düşüş güçlü |
| ↓ | ↓ | Long kapatılıyor → Düşüş zayıflayabilir |

### 4.4 OI Spike

```
OI_Ratio = ΔOI / SMA(|ΔOI|, 20)

if OI_Ratio > 2.0:
    OI_Spike = true    # Anormal pozisyon açılması
```

---

## 5. Bid/Ask Imbalance

### 5.1 Order Book Imbalance

```
Imbalance = (Bid_Volume - Ask_Volume) / (Bid_Volume + Ask_Volume)
```

- `Imbalance > 0.3` → güçlü alım baskısı
- `Imbalance < -0.3` → güçlü satış baskısı

### 5.2 Trade Imbalance (Aggressor Analysis)

```
Trade_Imbalance = (Aggressive_Buys - Aggressive_Sells) / Total_Trades
```

### 5.3 Footprint Imbalance (İleri Seviye)

Her fiyat seviyesinde:

```
Level_Imbalance_i = Bid_Volume_i / Ask_Volume_i
```

- `Level_Imbalance > 3.0` → strong bid (destek)
- `Level_Imbalance < 0.33` → strong ask (direnç)

---

## 6. Volume Profile

### 6.1 Point of Control (POC)

```
POC = fiyat seviyesi where Volume = max(Volume_per_level)
```

En fazla işlem yapılan fiyat → güçlü destek/direnç.

### 6.2 Value Area

```
VA_Total = %70 of total volume
VA_High = üst sınır of value area
VA_Low = alt sınır of value area
```

Hesaplama:
1. POC'dan başla
2. Yukarı ve aşağı yönde sırayla en yüksek hacimli seviyeleri ekle
3. Toplam %70'e ulaşana kadar devam et

### 6.3 Volume Nodes

```
HVN (High Volume Node): Volume at level > SMA(Volume_per_level) × 1.5
LVN (Low Volume Node): Volume at level < SMA(Volume_per_level) × 0.5
```

- HVN → fiyat burada konsolide olur (destek/direnç)
- LVN → fiyat buradan hızlı geçer (boşluk)

---

## 7. Volume Confirmation Rules

### 7.1 Breakout Validation

```
Valid_Breakout ⟺ 
    price_breaks_level ∧ 
    Volume_Ratio > 1.5 ∧ 
    Delta_direction_matches_breakout
```

**Fake breakout tespiti:**

```
Fake_Breakout ⟺ 
    price_breaks_level ∧ 
    Volume_Ratio < 1.0 ∧ 
    quick_rejection (within 3 candles)
```

### 7.2 Trend Strength Validation

```
Trend_Valid ⟺ 
    price_trending ∧ 
    CVD_trending_in_same_direction ∧ 
    OI_increasing
```

### 7.3 Momentum Confirmation

```
Momentum_Confirmed ⟺ 
    impulse_candle ∧ 
    Volume_Ratio > 2.0 ∧ 
    |Delta| > SMA(|Delta|, 20) × 1.5
```

---

## 8. Volume Composite Score

```
Vol_Score = w₁ × VR_Signal + w₂ × Delta_Signal + w₃ × OI_Signal + w₄ × Imbalance_Signal
```

burada:
- `VR_Signal = clip((Volume_Ratio - 1) / 2, -1, 1)` → normalleştirilmiş
- `Delta_Signal = clip(Delta / SMA(|Delta|, 20), -1, 1)` → yönlü
- `OI_Signal = clip(ΔOI% / 5, -1, 1)`
- `Imbalance_Signal = clip(Imbalance × 3, -1, 1)`

Ağırlıklar: `w₁ = 0.30, w₂ = 0.30, w₃ = 0.20, w₄ = 0.20`

---

## 9. Algoritmik Akış

```
1. OHLCV verisi + OI + funding verisi al
2. Volume SMA, Z-Score, RVOL hesapla
3. Delta hesapla (tick veya candle bazlı)
4. CVD güncelle
5. OI değişimi hesapla
6. Bid/Ask imbalance hesapla
7. Volume Composite Score oluştur
8. Diğer engine'lere sinyal olarak bildir:
   - Breakout validation
   - Trend strength
   - Momentum confirmation
   - Divergence uyarısı
```

---

## 10. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `VOL_SMA_PERIOD` | 20 | Volume SMA penceresi |
| `VOL_SPIKE_THRESHOLD` | 2.0 | Volume spike eşiği (ratio) |
| `DELTA_SMA_PERIOD` | 20 | Delta SMA penceresi |
| `OI_SPIKE_THRESHOLD` | 2.0 | OI spike eşiği (ratio) |
| `IMBALANCE_THRESHOLD` | 0.3 | Bid/Ask imbalance eşiği |
| `VP_LOOKBACK` | 100 | Volume Profile hesaplama penceresi |
| `VALUE_AREA_PCT` | 70% | Value Area yüzdesi |

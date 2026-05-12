# Regime Detection Engine — Matematiksel Model

## 1. Amaç

Market tipini (rejimini) belirlemek ve aktif stratejileri buna göre seçmek. Yanlış rejimde doğru strateji bile kayıp üretir. Bu engine tüm diğer engine'lere "hangi ortamdasınız" bilgisini verir.

---

## 2. Market Rejimleri

| Rejim | Tanım | Uygun Stratejiler |
|---|---|---|
| **TRENDING** | Güçlü tek yönlü hareket | Trend Following, OB Continuation |
| **RANGING** | Belirli band içinde salınım | Mean Reversion, Grid |
| **HIGH_VOL** | Yüksek volatilite, hızlı hareketler | Cautious trading, reduced size |
| **LOW_VOL** | Düşük volatilite, sıkışma | Mean Reversion, breakout bekleme |
| **NEWS_DRIVEN** | Haber/event kaynaklı hareketler | Trading durdur veya azalt |

---

## 3. Rejim Tespit Metrikleri

### 3.1 ADX (Trend/Range Ayrımı)

```
if ADX > 25:
    regime_trend_signal = "TRENDING"
elif ADX < 20:
    regime_trend_signal = "RANGING"
else:
    regime_trend_signal = "TRANSITIONAL"
```

### 3.2 ATR Ratio (Volatilite Seviyesi)

```
ATR_Ratio = ATR(14) / SMA(ATR(14), 50)
```

| ATR_Ratio | Volatilite |
|---|---|
| < 0.7 | LOW_VOL |
| 0.7 - 1.3 | NORMAL |
| 1.3 - 2.0 | HIGH_VOL |
| > 2.0 | EXTREME |

### 3.3 Bollinger Bandwidth

```
BW = (Upper_Band - Lower_Band) / Middle_Band
BW_Percentile = percentile_rank(BW, Historical_BW_90days)
```

- `BW_Percentile < 10` → Squeeze (LOW_VOL → breakout yakın)
- `BW_Percentile > 90` → Aşırı genişleme (HIGH_VOL)

### 3.4 Hurst Exponent (İleri Seviye)

```
H = Hurst_Exponent(price_series, lags)
```

| Hurst | Yorum |
|---|---|
| H < 0.4 | Mean-reverting (RANGING) |
| 0.4 < H < 0.6 | Random walk (kararsız) |
| H > 0.6 | Trending (TRENDING) |

Hesaplama (Rescaled Range yöntemi):

```
For each lag n:
    1. Sub-series oluştur
    2. Her sub-series için:
       a. Mean çıkar → Y_t = X_t - μ
       b. Kümülatif sapma → Z_t = Σ Y_i
       c. Range → R = max(Z) - min(Z)
       d. Standart sapma → S
       e. R/S hesapla
    3. log(R/S) vs log(n) regresyon
    4. Eğim = H
```

### 3.5 Choppiness Index

```
CI = 100 × log10(Σ ATR_i / (Highest_High - Lowest_Low)) / log10(n)
    i = son n periyot
```

- `CI > 61.8` → Choppy / RANGING
- `CI < 38.2` → Trending

### 3.6 Efficiency Ratio (Kaufman)

```
ER = |Close_t - Close_{t-n}| / Σ |Close_i - Close_{i-1}|    (i = t-n+1 ... t)
```

- `ER > 0.6` → Efficient movement → TRENDING
- `ER < 0.3` → Noisy → RANGING

---

## 4. Composite Regime Score

### 4.1 Trend Score

```
Trend_Score = w₁ × ADX_norm + w₂ × ER + w₃ × Hurst_norm + w₄ × (1 - CI_norm)
```

burada normalize:

```
ADX_norm = min(ADX / 50, 1)
ER = ER (zaten 0-1)
Hurst_norm = min(max(H - 0.3, 0) / 0.4, 1)
CI_norm = min(max(CI - 30, 0) / 40, 1)
```

Ağırlıklar: `w₁ = 0.30, w₂ = 0.25, w₃ = 0.25, w₄ = 0.20`

### 4.2 Volatilite Score

```
Vol_Score = w₁ × ATR_Ratio_norm + w₂ × BW_Percentile_norm + w₃ × Vol_of_Vol_norm
```

burada:

```
ATR_Ratio_norm = min(ATR_Ratio / 2, 1)
BW_Percentile_norm = BW_Percentile / 100
Vol_of_Vol = StdDev(ATR, 14) / SMA(ATR, 14)    # Volatilitenin volatilitesi
Vol_of_Vol_norm = min(Vol_of_Vol / 0.5, 1)
```

### 4.3 Rejim Belirleme

```
if Trend_Score ≥ 0.6 ∧ Vol_Score ≥ 0.6:
    regime = "TRENDING_HIGH_VOL"
elif Trend_Score ≥ 0.6 ∧ Vol_Score < 0.6:
    regime = "TRENDING"
elif Trend_Score < 0.4 ∧ Vol_Score < 0.4:
    regime = "RANGING_LOW_VOL" (squeeze)
elif Trend_Score < 0.4 ∧ Vol_Score ≥ 0.4:
    regime = "RANGING"
elif Vol_Score ≥ 0.8:
    regime = "HIGH_VOL"
else:
    regime = "TRANSITIONAL"
```

---

## 5. News Detection

### 5.1 Anomali Tespiti

```
News_Event ⟺ 
    Volume_Spike > 5.0 ∧ 
    ATR_Spike > 3.0 ∧ 
    (within economic calendar event window)
```

### 5.2 Volatilite Anomalisi

```
Vol_Anomaly = (ATR_current - SMA(ATR, 50)) / StdDev(ATR, 50)

if Vol_Anomaly > 4.0 ∧ no_technical_reason:
    regime = "NEWS_DRIVEN"
```

### 5.3 Economic Calendar Entegrasyonu

```
Upcoming_Events = fetch_calendar()

if time_until_event < 30 min ∧ event_importance = "HIGH":
    RESTRICT_TRADING()
    
if time_since_event < 60 min:
    MONITOR_ONLY()
```

---

## 6. Regime Transition

### 6.1 Transition Smoothing

Ani rejim değişikliklerini önlemek:

```
Smoothed_Regime = mode(regime_last_n_candles)    n = 5
```

### 6.2 Transition Confirmation

```
Regime_Changed ⟺ new_regime sustained for ≥ MIN_CONFIRMATION_CANDLES
```

Önerilen: `MIN_CONFIRMATION_CANDLES = 3-5`

### 6.3 Hysteresis (Gecikme Tamponu) ve Churn Önleme

Sistemin en büyük düşmanlarından biri "Churn" yani sürekli fikir değiştirip strateji atlamasıdır. Sadece 1-2 saatlik farklı bir mum yapısı rejim değiştirmemelidir.

```
# TRENDING → RANGING geçişi (daha yüksek eşik ve süre)
if current_regime == "TRENDING":
    switch_to_RANGING ⟺ Trend_Score < 0.35 ∧ Sustained_for_Bars ≥ 6

# RANGING → TRENDING geçişi (daha yüksek eşik ve süre)
if current_regime == "RANGING":
    switch_to_TRENDING ⟺ Trend_Score > 0.65 ∧ Sustained_for_Bars ≥ 6
```

Bu hysteresis (gecikme) ve zaman filtresi, botun geçici fiyat dalgalanmalarında stratejiden stratejiye zıplayıp (flip-flopping) sürekli pozisyon açıp kapatmasını engeller. Yeni bir stratejiye ancak piyasanın yönü ve yapısı KALICI olarak değiştiğinde geçilir.

### 6.4 Hysteresis Override (Acil Rejim Kırıcı)

Hysteresis kuralı yatay piyasada hayat kurtarır ancak piyasa bir habere veya balina alımına aşırı şiddetli tepki verirse "6 mum beklemek" büyük fırsatları veya çöküşleri kaçırmak demektir (Gecikme Girdabı açığı).
Bu yüzden sisteme bir "Devre Kesici" (Circuit Breaker) eklenmiştir:

```
if (ADX_current - ADX_previous) > 15 ∨ (Close - Open) > (ATR × 3):
    IGNORE_HYSTERESIS()
    FORCE_REGIME_CHANGE()
```
Bu sayede anlık şoklarda bot beklemez, hemen o anki gerçek rejime uyum sağlar.

---

## 7. Strateji-Rejim Eşleştirmesi

### 7.1 Aktivasyon Matrisi

| Engine | TRENDING | RANGING | HIGH_VOL | LOW_VOL | NEWS |
|---|---|---|---|---|---|
| Trend Following | ✅ ON | ❌ OFF | ⚠️ Small | ❌ OFF | ❌ OFF |
| Mean Reversion | ❌ OFF | ✅ ON | ❌ OFF | ✅ ON | ❌ OFF |
| OB Retest | ✅ ON | ✅ ON | ⚠️ Small | ✅ ON | ❌ OFF |
| Liquidity Sweep | ✅ ON | ✅ ON | ✅ ON | ⚠️ Careful | ❌ OFF |
| Grid Strategy | ❌ OFF | ✅ ON | ❌ OFF | ✅ ON | ❌ OFF |
| Stat Arb | ⚠️ Careful | ✅ ON | ❌ OFF | ✅ ON | ❌ OFF |

### 7.2 Position Size Ayarı

```
Regime_Size_Multiplier = {
    "TRENDING": 1.0,
    "RANGING": 0.8,
    "HIGH_VOL": 0.5,
    "LOW_VOL": 0.7,
    "NEWS_DRIVEN": 0.0,     # Trading durdur
    "TRANSITIONAL": 0.5
}

Adjusted_Position_Size = Base_Size × Regime_Size_Multiplier[current_regime]
```

---

## 8. Algoritmik Akış

```
1. Her candle'da:
   a. ADX, ATR_Ratio, BW, ER, Hurst hesapla
   b. Trend Score ve Vol Score hesapla
   c. Rejim belirle
   d. Transition smoothing uygula
2. Rejim değişirse:
   a. Strateji aktivasyon matrisini güncelle
   b. Position size multiplier güncelle
   c. Diğer engine'lere rejim bildir
3. News check:
   a. Volume/ATR anomalisi kontrol et
   b. Economic calendar kontrol et
   c. NEWS_DRIVEN ise tüm trading durdur
```

---

## 9. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `ADX_TREND_THRESHOLD` | 25 | ADX trend eşiği |
| `ADX_RANGE_THRESHOLD` | 20 | ADX range eşiği |
| `ATR_RATIO_WINDOW` | 50 | ATR ratio SMA penceresi |
| `BW_PERIOD` | 20 | Bollinger BW periyodu |
| `HURST_LAGS` | [10,20,40,80] | Hurst exponent lag'ları |
| `CI_PERIOD` | 14 | Choppiness Index periyodu |
| `ER_PERIOD` | 10 | Efficiency Ratio periyodu |
| `REGIME_SMOOTH_N` | 5 | Rejim smoothing penceresi |
| `REGIME_CONFIRM_CANDLES` | 3 | Rejim değişim konfirmasyonu |
| `NEWS_VOL_THRESHOLD` | 4.0 | News volatilite anomali eşiği |

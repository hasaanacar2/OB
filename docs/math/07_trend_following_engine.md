# Trend Following Engine — Matematiksel Model

## 1. Amaç

Güçlü ve sürdürülebilir trend hareketlerini tespit edip onları takip etmek. Trend piyasalarında en yüksek beklenen değeri sağlayan stratejidir.

---

## 2. Trend Tespit Yöntemleri

### 2.1 EMA (Exponential Moving Average) Bazlı

**EMA hesaplama:**

```
EMA_t = Price_t × α + EMA_{t-1} × (1 - α)
```

burada:

```
α = 2 / (n + 1)
```

**Trend belirleme:**

```
Bullish Trend ⟺ EMA(50) > EMA(200)
Bearish Trend ⟺ EMA(50) < EMA(200)
```

**EMA ayrışma gücü:**

```
EMA_Spread = (EMA_fast - EMA_slow) / ATR
```

- `|EMA_Spread| > 1.0` → güçlü trend
- `|EMA_Spread| < 0.3` → zayıf trend / range

### 2.2 ADX (Average Directional Index)

**+DI ve -DI hesaplama:**

```
+DM = H_t - H_{t-1}   (if > 0 and > |L_{t-1} - L_t|, else 0)
-DM = L_{t-1} - L_t    (if > 0 and > H_t - H_{t-1}, else 0)

+DI = 100 × SMA(+DM, 14) / ATR(14)
-DI = 100 × SMA(-DM, 14) / ATR(14)
```

**ADX hesaplama:**

```
DX = 100 × |+DI - (-DI)| / (+DI + (-DI))
ADX = SMA(DX, 14)
```

**Trend gücü:**

| ADX Değeri | Yorum |
|---|---|
| < 20 | Trend yok (range) |
| 20 - 25 | Zayıf trend |
| 25 - 50 | Güçlü trend |
| 50 - 75 | Çok güçlü trend |
| > 75 | Aşırı güçlü (nadirdir) |

**Trend yönü:**

```
Bullish ⟺ +DI > -DI ∧ ADX > 25
Bearish ⟺ -DI > +DI ∧ ADX > 25
```

### 2.3 Market Structure Bazlı

```
Bullish Trend ⟺ HH ∧ HL dizisi mevcut (Higher Highs + Higher Lows)
Bearish Trend ⟺ LH ∧ LL dizisi mevcut (Lower Highs + Lower Lows)
```

Minimum ardışık swing sayısı:

```
Trend_Confirmed ⟺ consecutive_HH_HL ≥ 3 (veya consecutive_LH_LL ≥ 3)
```

---

## 3. Trend Strength (Güç) Metrikleri

### 3.1 Composite Trend Score

```
Trend_Score = w₁ × EMA_Signal + w₂ × ADX_Signal + w₃ × Structure_Signal + w₄ × Momentum_Signal
```

burada:
- `EMA_Signal = sign(EMA_fast - EMA_slow) × min(|EMA_Spread|, 2) / 2`  → [-1, 1]
- `ADX_Signal = min(ADX, 50) / 50 × sign(+DI - (-DI))`  → [-1, 1]
- `Structure_Signal = +1 (HH/HL), -1 (LH/LL), 0 (kararsız)`
- `Momentum_Signal = ROC(close, 20) / (ATR × 20) clipped to [-1, 1]`

Ağırlıklar: `w₁ = 0.25, w₂ = 0.25, w₃ = 0.30, w₄ = 0.20`

### 3.2 ROC (Rate of Change)

```
ROC(n) = (Close_t - Close_{t-n}) / Close_{t-n} × 100
```

### 3.3 Slope (Eğim)

```
Slope = linear_regression_slope(close, n_periods)
```

Normalize edilmiş:

```
Normalized_Slope = Slope × n_periods / ATR
```

---

## 4. Pullback Tespiti

### 4.1 Pullback Derinliği

```
Pullback_Depth = |Swing_Extreme - Current_Price| / |Swing_Extreme - Previous_Swing|
```

**Fibonacci seviyeleri:**

```
Fib_38.2 = Swing_High - (Swing_High - Swing_Low) × 0.382
Fib_50.0 = Swing_High - (Swing_High - Swing_Low) × 0.500
Fib_61.8 = Swing_High - (Swing_High - Swing_Low) × 0.618
```

İdeal pullback: %38.2 - %61.8 arası

### 4.2 Pullback Kalitesi

```
PB_Quality = {
    depth_ok: 0.382 ≤ Pullback_Depth ≤ 0.618,
    volume_decreasing: Volume_pullback < Volume_impulse × 0.7,
    structure_intact: no_CHOCH_during_pullback,
    ema_support: price_bounced_near_EMA(20_or_50)
}
```

### 4.3 Pullback Entry Kuralı

```
Entry_Bullish ⟺ 
    Trend = Bullish ∧ 
    Pullback_Depth ∈ [0.382, 0.618] ∧ 
    BOS_continuation_after_pullback ∧
    Volume_confirmation
```

---

## 5. Continuation Signal

### 5.1 Flag / Pennant Pattern

```
Flag ⟺ 
    impulse_range ≥ ATR × 2 ∧ 
    consolidation_range < impulse_range × 0.5 ∧
    consolidation_duration ≥ 5 candles ∧
    slope_of_consolidation is_opposite_to_trend
```

### 5.2 Momentum Divergence Check

Trend devam ediyor ama momentum azalıyor mu?

```
if price_making_new_high ∧ RSI_not_making_new_high:
    bearish_divergence = true    # Trend zayıflıyor
```

---

## 6. Exit Kuralları (Trend Following Specific)

### 6.1 Trend Sonu Sinyalleri

```
EXIT if:
    EMA_fast crosses below EMA_slow    # Golden/Death cross
    ADX drops below 20                  # Trend gücü bitti
    CHOCH detected                      # Yapı değişimi
    RSI extreme + divergence            # Momentum tükenme
```

### 6.2 Trend Trailing Stop

Daha geniş trailing (trend'i öldürmemek için):

```
Trail_SL = Close_t - ATR × 3.0    # Normal trailing'den daha geniş
```

---

## 7. Risk: Range Market Sorunu

Trend Following, range market'te sistematik kayıp üretir:

```
Range_Detection:
    ADX < 20 ∧ |EMA_Spread| < 0.3 ∧ ATR_contraction
    
if Range_Detected:
    DISABLE_TREND_FOLLOWING()
```

**Whipsaw maliyeti:**

```
Whipsaw_Cost = false_entry_count × avg_loss_per_whipsaw
```

Range'de bu maliyet birikir → Regime Detection Engine ile entegrasyon kritik.

---

## 8. Algoritmik Akış

```
1. EMA(50), EMA(200), ADX hesapla
2. Market structure analiz et (HH/HL veya LH/LL)
3. Trend Score hesapla
4. Trend güçlüyse:
   a. Pullback bekle
   b. Pullback kalitesini değerlendir
   c. Continuation BOS sonrası entry sinyali üret
5. Trend zayıfsa veya range'deyse:
   a. Sinyal üretme
   b. Regime Detection Engine'e bildir
6. Açık pozisyonlarda:
   a. Geniş trailing stop kullan
   b. Divergence kontrolü yap
   c. Trend sonu sinyallerini izle
```

---

## 9. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `EMA_FAST` | 50 | Hızlı EMA periyodu |
| `EMA_SLOW` | 200 | Yavaş EMA periyodu |
| `ADX_PERIOD` | 14 | ADX hesaplama periyodu |
| `ADX_TREND_THRESHOLD` | 25 | Trend kabul eşiği |
| `MIN_PULLBACK_DEPTH` | 0.382 | Minimum pullback derinliği |
| `MAX_PULLBACK_DEPTH` | 0.618 | Maksimum pullback derinliği |
| `TREND_TRAIL_ATR` | 3.0 | Trend trailing ATR çarpanı |
| `MIN_CONSECUTIVE_SWINGS` | 3 | Trend konfirmasyonu için min swing |
| `ROC_PERIOD` | 20 | Rate of Change periyodu |

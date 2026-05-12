# Mean Reversion Engine — Matematiksel Model

## 1. Amaç

Fiyatın ortalamadan aşırı sapma gösterdiği durumlarda ortalamaya dönüş hareketini trade etmek. Range ve düşük volatilite marketlerde güçlü; trend marketlerde tehlikeli.

---

## 2. Temel Kavram: Mean (Ortalama)

### 2.1 VWAP (Volume Weighted Average Price)

```
VWAP = Σ (Typical_Price × Volume) / Σ Volume
```

burada:

```
Typical_Price = (H + L + C) / 3
```

**İntraday VWAP**: Her gün sıfırlanır.

**Anchored VWAP**: Belirli bir noktadan (swing, event) başlar.

### 2.2 SMA / EMA

```
SMA(n) = Σ Close_i / n     (i = t-n+1 ... t)

EMA(n) = Close_t × α + EMA_{t-1} × (1-α)    α = 2/(n+1)
```

### 2.3 Ortalamadan Sapma

```
Deviation = (Close - Mean) / σ
```

veya ATR normalize:

```
Deviation_ATR = (Close - Mean) / ATR
```

---

## 3. Z-Score Modeli

### 3.1 Z-Score Hesaplama

```
Z = (X - μ) / σ
```

burada:
- `X` = mevcut fiyat (veya spread)
- `μ` = SMA(n) veya VWAP
- `σ` = standart sapma (rolling, aynı window)

### 3.2 Z-Score Sinyal Kuralları

```
Long  ⟺ Z ≤ -Z_threshold    (aşırı düşük, ortalamaya dönüş beklenir)
Short ⟺ Z ≥ +Z_threshold    (aşırı yüksek, ortalamaya dönüş beklenir)
Exit  ⟺ |Z| ≤ Z_exit        (ortalamaya döndü)
```

**Önerilen eşikler:**

| Eşik | Değer | Yorum |
|---|---|---|
| `Z_entry` | ±2.0 | Entry sinyali |
| `Z_strong` | ±2.5 | Güçlü sinyal |
| `Z_extreme` | ±3.0 | Aşırı sapma |
| `Z_exit` | ±0.5 | Çıkış (ortalamaya dönüş) |

### 3.3 Rolling Z-Score

```
μ_t = SMA(Close, n)_t
σ_t = StdDev(Close, n)_t
Z_t = (Close_t - μ_t) / σ_t
```

Rolling pencere: `n = 20-50`

---

## 4. Bollinger Bands

### 4.1 Hesaplama

```
Middle_Band = SMA(Close, 20)
Upper_Band = Middle_Band + k × σ
Lower_Band = Middle_Band - k × σ
```

- `k` = 2.0 (standart), 2.5 veya 3.0 (daha seçici)
- `σ` = rolling standart sapma (20 periyot)

### 4.2 %B (Bandwidth Position)

```
%B = (Close - Lower_Band) / (Upper_Band - Lower_Band)
```

- `%B > 1.0` → fiyat üst bandın üstünde (overbought)
- `%B < 0.0` → fiyat alt bandın altında (oversold)
- `%B ≈ 0.5` → fiyat orta bantta

### 4.3 Bandwidth (Volatilite Ölçüsü)

```
BW = (Upper_Band - Lower_Band) / Middle_Band
```

- Düşük BW → squeeze (volatilite sıkışması) → breakout beklenebilir
- Yüksek BW → yüksek volatilite

### 4.4 Bollinger Sinyal Kuralları

```
Long  ⟺ Close < Lower_Band ∧ %B < -0.05
Short ⟺ Close > Upper_Band ∧ %B > 1.05
Exit  ⟺ Close crosses Middle_Band
```

---

## 5. RSI (Relative Strength Index)

### 5.1 Hesaplama

```
RS = SMA(Gains, 14) / SMA(Losses, 14)
RSI = 100 - (100 / (1 + RS))
```

veya Wilder smoothing:

```
Avg_Gain_t = (Avg_Gain_{t-1} × 13 + Gain_t) / 14
Avg_Loss_t = (Avg_Loss_{t-1} × 13 + Loss_t) / 14
```

### 5.2 RSI Sinyal Kuralları

```
Oversold  ⟺ RSI < 30    → Long sinyali
Overbought ⟺ RSI > 70    → Short sinyali
```

**Daha seçici:**

```
Strong Oversold  ⟺ RSI < 20
Strong Overbought ⟺ RSI > 80
```

### 5.3 RSI Divergence

**Bullish Divergence:**

```
Price makes lower low ∧ RSI makes higher low → Potential reversal up
```

**Bearish Divergence:**

```
Price makes higher high ∧ RSI makes lower high → Potential reversal down
```

---

## 6. VWAP Bands

### 6.1 VWAP Standart Sapma Bantları

```
VWAP_Upper_1 = VWAP + 1σ
VWAP_Upper_2 = VWAP + 2σ
VWAP_Lower_1 = VWAP - 1σ
VWAP_Lower_2 = VWAP - 2σ
```

burada σ:

```
σ = √(Σ Volume × (Typical_Price - VWAP)² / Σ Volume)
```

### 6.2 VWAP Mean Reversion Kuralları

```
Long  ⟺ Close < VWAP - 2σ
Short ⟺ Close > VWAP + 2σ
TP    = VWAP (ortalamaya dönüş)
```

---

## 7. Composite Mean Reversion Score

```
MR_Score = w₁ × Z_Signal + w₂ × BB_Signal + w₃ × RSI_Signal + w₄ × VWAP_Signal
```

burada her sinyal [-1, 1] aralığında:

```
Z_Signal = clip(Z / Z_threshold, -1, 1) × (-1)
BB_Signal = clip((%B - 0.5) × 2, -1, 1) × (-1)
RSI_Signal = clip((RSI - 50) / 30, -1, 1) × (-1)
VWAP_Signal = clip((Close - VWAP) / (2σ), -1, 1) × (-1)
```

(-1 ile çarpılır çünkü mean reversion ters yön trade eder)

Ağırlıklar: `w₁ = 0.30, w₂ = 0.25, w₃ = 0.20, w₄ = 0.25`

**Sinyal eşikleri:**

```
|MR_Score| ≥ 0.7  → güçlü sinyal
|MR_Score| ≥ 0.5  → orta sinyal
|MR_Score| < 0.5  → sinyal yok
```

---

## 8. Entry ve Exit

### 8.1 Entry

```
Long Entry ⟺ MR_Score ≤ -0.7 ∧ Regime = RANGING ∧ Volume_not_spiking
Short Entry ⟺ MR_Score ≥ +0.7 ∧ Regime = RANGING ∧ Volume_not_spiking
```

### 8.2 Stop-Loss

```
SL = Entry ± ATR × 2.0    (ortalamadan daha da uzaklaşması durumuna karşı)
```

Veya Bollinger dışı:

```
SL_Long = Lower_Band - ATR × 0.5
SL_Short = Upper_Band + ATR × 0.5
```

### 8.3 Take-Profit

```
TP = Mean (VWAP veya SMA)    # Ortalamaya dönüş hedefi
```

Alternatif:

```
TP₁ = Entry + 1σ (ortalamaya doğru)    # %50 pozisyon
TP₂ = Mean                              # %50 pozisyon
```

---

## 9. Trend Markette Tehlike

Mean reversion trend'de sistematik kayıp üretir:

```
DISABLE_MR if:
    ADX > 30
    |EMA_Spread| > 1.0
    Consecutive BOS count > 3
```

**Risk kontrolü:**

```
MR_Max_Loss_Per_Trade = 0.3%    # Normal'den daha düşük risk
MR_Max_Consecutive_Losses = 3    # Hızlı durdurma
```

---

## 10. Algoritmik Akış

```
1. Regime Detection Engine'den market tipi al
2. Range veya Low-Vol market ise:
   a. Z-Score, Bollinger, RSI, VWAP hesapla
   b. Composite MR Score hesapla
   c. Eşik kontrolü yap
   d. Volume spike olmadığını doğrula
   e. Entry sinyali üret
3. Trend market ise:
   a. MR Engine devre dışı
4. Açık pozisyonda:
   a. Ortalamaya dönüşü izle
   b. Z-Score 0'a yaklaşırsa → exit
   c. Z-Score daha da sapma gösterirse → SL kontrol
```

---

## 11. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `Z_WINDOW` | 20 | Z-Score rolling penceresi |
| `Z_ENTRY_THRESHOLD` | 2.0 | Z-Score entry eşiği |
| `Z_EXIT_THRESHOLD` | 0.5 | Z-Score exit eşiği |
| `BB_PERIOD` | 20 | Bollinger periyodu |
| `BB_STD` | 2.0 | Bollinger standart sapma çarpanı |
| `RSI_PERIOD` | 14 | RSI periyodu |
| `RSI_OVERSOLD` | 30 | RSI oversold eşiği |
| `RSI_OVERBOUGHT` | 70 | RSI overbought eşiği |
| `MR_SCORE_THRESHOLD` | 0.7 | Composite score entry eşiği |
| `MR_MAX_RISK` | 0.3% | Trade başına max risk (düşük) |

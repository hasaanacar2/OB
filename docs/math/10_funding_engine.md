# Funding Engine — Matematiksel Model

## 1. Amaç

Perpetual futures piyasalarındaki kalabalık pozisyonları analiz etmek. Aşırı funding, aşırı OI ve likidasyon verileri ile squeeze ortamlarını tespit etmek.

---

## 2. Funding Rate

### 2.1 Tanım

Funding rate, perpetual futures fiyatını spot fiyata yakın tutmak için her 8 saatte bir pozisyon sahipleri arasında yapılan ödeme oranıdır.

```
Funding_Payment = Position_Size × Funding_Rate
```

- `Funding_Rate > 0` → Longlar shortlara ödüyor (çoğunluk long)
- `Funding_Rate < 0` → Shortlar longlara ödüyor (çoğunluk short)

### 2.2 Funding Rate Hesaplaması (Binance)

```
Funding_Rate = clamp(Premium_Index + clamp(Interest_Rate - Premium_Index, -0.05%, 0.05%), -0.75%, 0.75%)
```

burada:

```
Premium_Index = (Futures_TWAP - Spot_TWAP) / Spot_TWAP
```

### 2.3 Annualized Funding

```
Annualized_Funding = Funding_Rate × 3 × 365 × 100    (yıllık %)
```

(3 = günde 3 funding period)

---

## 3. Funding Extremes

### 3.1 Z-Score Bazlı Extreme Tespiti

```
FR_Z = (Funding_Rate - SMA(Funding_Rate, 30)) / StdDev(Funding_Rate, 30)
```

| FR_Z | Yorum |
|---|---|
| > +2.0 | Aşırı pozitif funding → long crowded |
| > +3.0 | Ekstrem → short squeeze potansiyeli bitti, long squeeze riski |
| < -2.0 | Aşırı negatif funding → short crowded |
| < -3.0 | Ekstrem → long squeeze potansiyeli bitti, short squeeze riski |

### 3.2 Percentile Bazlı

```
FR_Percentile = percentile_rank(Current_FR, Historical_FR_90days)
```

- `FR_Percentile > 90` → Aşırı pozitif
- `FR_Percentile < 10` → Aşırı negatif

### 3.3 Kümülatif Funding

```
Cumulative_Funding(n) = Σ Funding_Rate_i    (son n period)
```

Kümülatif funding ne kadar yüksekse, pozisyonları taşıma maliyeti o kadar artmıştır → pozisyon kapatma baskısı.

---

## 4. Open Interest Analizi

### 4.1 OI Değişimi

```
ΔOI = OI_t - OI_{t-1}
ΔOI_pct = ΔOI / OI_{t-1} × 100
```

### 4.2 OI / Market Cap Oranı

```
OI_Ratio = Total_OI / Market_Cap
```

- `OI_Ratio > 3%` → Yüksek leverage ortamı
- `OI_Ratio > 5%` → Aşırı leverage → cascade riski yüksek

### 4.3 OI ile Funding Kombinasyonu

| OI Değişimi | Funding | Yorum |
|---|---|---|
| ↑ Artıyor | Pozitif ↑ | Agresif long açılıyor → squeeze riski |
| ↑ Artıyor | Negatif ↓ | Agresif short açılıyor → bounce riski |
| ↓ Azalıyor | Pozitif → Nötr | Longlar kapatıyor → düşüş |
| ↓ Azalıyor | Negatif → Nötr | Shortlar kapatıyor → yükseliş |

---

## 5. Long/Short Ratio

### 5.1 Hesaplama

```
LSR = Long_Accounts / Short_Accounts
```

veya hacim bazlı:

```
LSR_Volume = Long_Volume / Short_Volume
```

### 5.2 LSR Extreme Tespiti

```
LSR_Z = (LSR - SMA(LSR, 30)) / StdDev(LSR, 30)
```

- `LSR_Z > 2.0` → Çoğunluk long → contrarian short fırsatı
- `LSR_Z < -2.0` → Çoğunluk short → contrarian long fırsatı

### 5.3 LSR Trend

```
LSR_Trend = SMA(LSR, 5) - SMA(LSR, 20)
```

LSR hızla yükseliyorsa → retail aşırı long oluyor.

---

## 6. Liquidation Analizi

### 6.1 Likidasyon Fiyatı Tahmini

Long pozisyon likidasyon fiyatı:

```
Liq_Price_Long = Entry × (1 - 1/Leverage + Maintenance_Margin)
```

Short pozisyon likidasyon fiyatı:

```
Liq_Price_Short = Entry × (1 + 1/Leverage - Maintenance_Margin)
```

### 6.2 Likidasyon Kümesi Tespiti

```
Liq_Density(price_level) = Σ estimated_position_size_i
    ∀ positions where Liq_Price_i ∈ [price_level - δ, price_level + δ]
```

- `δ` = fiyat bandı (ATR × 0.1)

### 6.3 Cascade Riski

```
Cascade_Risk = Liq_Density / Average_Hourly_Volume
```

- `Cascade_Risk > 0.5` → Yüksek cascade potansiyeli
- `Cascade_Risk > 1.0` → Çok yüksek → hızlı hareket beklenir

---

## 7. Squeeze Detection

### 7.1 Long Squeeze Koşulları

```
Long_Squeeze_Risk ⟺
    Funding_Rate > +0.05% ∧
    OI at high levels ∧
    LSR > 2.0 ∧
    Price approaching liquidation clusters below
```

### 7.2 Short Squeeze Koşulları

```
Short_Squeeze_Risk ⟺
    Funding_Rate < -0.03% ∧
    OI at high levels ∧
    LSR < 0.5 ∧
    Price approaching liquidation clusters above
```

### 7.3 Squeeze Composite Score

```
Squeeze_Score = w₁ × FR_Extreme + w₂ × OI_Level + w₃ × LSR_Extreme + w₄ × Liq_Proximity
```

burada:
- `FR_Extreme = |FR_Z| / 3 clipped to [0, 1]`
- `OI_Level = min(OI_Ratio / 5%, 1)`
- `LSR_Extreme = |LSR_Z| / 3 clipped to [0, 1]`
- `Liq_Proximity = 1 - (distance_to_liq_cluster / ATR × 5) clipped to [0, 1]`

Ağırlıklar: `w₁ = 0.30, w₂ = 0.25, w₃ = 0.20, w₄ = 0.25`

**Yön:**

```
if Funding > 0 ∧ LSR > 1:
    Squeeze_Direction = "long_squeeze" (aşağı hareket)
else:
    Squeeze_Direction = "short_squeeze" (yukarı hareket)
```

---

## 8. Funding Arbitrage (Ek Strateji)

### 8.1 Cash and Carry

```
if Annualized_Funding > 30%:
    Spot_Long + Futures_Short
    Expected_Return ≈ Annualized_Funding - Trading_Fees
```

### 8.2 Reverse Cash and Carry

```
if Annualized_Funding < -20%:
    Spot_Short (veya borrow) + Futures_Long
    Expected_Return ≈ |Annualized_Funding| - Trading_Fees - Borrow_Cost
```

---

## 9. Algoritmik Akış

```
1. Binance'den funding rate, OI, LSR, likidasyon verisi al
2. Funding Z-Score ve percentile hesapla
3. OI değişimi ve OI ratio hesapla
4. LSR trend ve extreme analizi yap
5. Likidasyon kümelerini tespit et
6. Squeeze Score hesapla
7. Sonuçları diğer engine'lere bildir:
   a. Risk Engine → leverage uyarısı
   b. Liquidity Engine → likidasyon bölgeleri
   c. OB Engine → contrarian sinyal
```

---

## 10. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `FR_Z_WINDOW` | 30 | Funding rate Z-Score penceresi |
| `FR_EXTREME_Z` | 2.0 | Funding extreme eşiği |
| `OI_HIGH_RATIO` | 3% | Yüksek OI oranı eşiği |
| `LSR_Z_WINDOW` | 30 | LSR Z-Score penceresi |
| `LSR_EXTREME_Z` | 2.0 | LSR extreme eşiği |
| `LIQ_PROXIMITY_ATR` | 5.0 | Likidasyon yakınlık eşiği (ATR) |
| `CASCADE_RISK_THRESHOLD` | 0.5 | Cascade riski eşiği |
| `FUNDING_ARB_THRESHOLD` | 30% | Funding arbitrage aktivasyon (annualized) |

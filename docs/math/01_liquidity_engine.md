# Liquidity Engine — Matematiksel Model

## 1. Amaç

Marketin stop-loss ve limit order biriktirdiği bölgeleri (liquidity pools) tespit etmek. Fiyat bu bölgelere doğru çekilir, likiditeyi toplar ve ardından gerçek yönünü belirler.

---

## 2. Temel Kavramlar

### 2.1 Equal Highs / Equal Lows

İki veya daha fazla swing noktasının birbirine çok yakın fiyatlarda oluşması durumudur.

**Equal Highs tanımı:**

```
|H_i - H_j| < ε
```

burada:
- `H_i`, `H_j` = ardışık veya yakın swing high fiyatları
- `ε` = tolerans eşiği (genelde ATR'nin %5-10'u kadar)

**Equal Lows tanımı:**

```
|L_i - L_j| < ε
```

burada:
- `L_i`, `L_j` = ardışık veya yakın swing low fiyatları

### 2.2 Epsilon (ε) Hesaplama

Sabit bir değer yerine volatiliteye göre dinamik hesaplanmalıdır:

```
ε = ATR(14) × k
```

burada:
- `ATR(14)` = 14 periyotluk Average True Range
- `k` = katsayı (önerilen: 0.05 ≤ k ≤ 0.10)

### 2.3 ATR (Average True Range)

```
TR_t = max(H_t - L_t, |H_t - C_{t-1}|, |L_t - C_{t-1}|)

ATR(n) = (1/n) × Σ TR_t    (t = 1...n)
```

veya EMA versiyonu:

```
ATR_t = ATR_{t-1} × (n-1)/n + TR_t × (1/n)
```

---

## 3. Liquidity Zone Tespiti

### 3.1 Swing High/Low Tespiti (Fractal)

Bir mum `swing high` olabilmesi için:

```
H_t > H_{t-k}  ve  H_t > H_{t+k}   (k = 1, 2, ..., lookback)
```

Bir mum `swing low` olabilmesi için:

```
L_t < L_{t-k}  ve  L_t < L_{t+k}   (k = 1, 2, ..., lookback)
```

Önerilen `lookback = 3` (yani 3 sol, 3 sağ toplam 7 mum).

### 3.2 Liquidity Level Skorlama

Her liquidity seviyesi için bir güç skoru:

```
Liq_Score = w₁ × touch_count + w₂ × age_factor + w₃ × volume_ratio
```

burada:
- `touch_count` = seviyeye kaç kez dokunulduğu (ne kadar çoksa, o kadar fazla stop birikir)
- `age_factor = 1 / (1 + candle_age / decay_period)` — zaman geçtikçe azalır
- `volume_ratio = volume_at_level / avg_volume` — hacim yoğunluğu
- `w₁, w₂, w₃` = ağırlıklar (toplamı 1, önerilen: 0.4, 0.3, 0.3)

### 3.3 Session Highs/Lows

Session bazlı liquidity:

```
Session_High = max(H_t)   t ∈ session_range
Session_Low  = min(L_t)   t ∈ session_range
```

Session tanımları (UTC):
- Asia: 00:00 - 08:00
- London: 08:00 - 13:00
- New York: 13:00 - 21:00

Previous Day High/Low:

```
PDH = max(H_t)   t ∈ previous_day
PDL = min(L_t)   t ∈ previous_day
```

---

## 4. Liquidity Sweep Tespiti

### 4.1 Sweep Koşulları

**Bullish Sweep (aşağı sweep → yukarı hareket beklenir):**

```
L_t < Liquidity_Level
C_t > Liquidity_Level
```

Ve wick boyutu yeterli olmalı:

```
sweep_wick = Liquidity_Level - L_t
sweep_wick ≥ ATR × 0.5
```

**Bearish Sweep (yukarı sweep → aşağı hareket beklenir):**

```
H_t > Liquidity_Level
C_t < Liquidity_Level
```

Ve:

```
sweep_wick = H_t - Liquidity_Level
sweep_wick ≥ ATR × 0.5
```

### 4.2 Sweep Kalitesi

```
Sweep_Quality = (sweep_wick / ATR) × (volume_t / avg_volume)
```

- `Sweep_Quality ≥ 1.0` → güçlü sweep
- `0.5 ≤ Sweep_Quality < 1.0` → orta kalite
- `Sweep_Quality < 0.5` → zayıf, dikkatli olunmalı

---

## 5. Kullanım Bağlamları

| Kullanım Alanı | Açıklama |
|---|---|
| **Liquidity Sweep** | Stop-loss toplandıktan sonra ters yönde entry |
| **Reversal Setup** | Sweep + BOS ile trend dönüşü tespiti |
| **Breakout Validation** | Gerçek breakout mu yoksa sweep mi ayrımı |
| **OB Entegrasyonu** | Sweep sonrası OB testi ile güçlü confluence |

---

## 6. Algoritmik Akış

```
1. Swing high/low tespit et (fractal)
2. Equal highs/lows bul (|H_i - H_j| < ε)
3. Session high/low ve PDH/PDL hesapla
4. Tüm liquidity seviyeleri listele ve skorla
5. Canlı fiyat bu seviyeleri sweep ederse:
   a. Sweep yönünü belirle
   b. Wick boyutunu kontrol et
   c. Close pozisyonunu kontrol et
   d. Sweep kalitesini hesapla
6. Signal üret veya diğer engine'lere bildir
```

---

## 7. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `ATR_PERIOD` | 14 | ATR hesaplama penceresi |
| `EPSILON_K` | 0.07 | Equal high/low toleransı (ATR çarpanı) |
| `FRACTAL_LOOKBACK` | 3 | Swing tespit lookback sayısı |
| `MIN_SWEEP_WICK_ATR` | 0.5 | Minimum sweep wick (ATR çarpanı) |
| `LEVEL_DECAY_PERIOD` | 100 | Liquidity seviye yaşlanma periodu (candle) |
| `MIN_TOUCH_COUNT` | 2 | Minimum dokunma sayısı |

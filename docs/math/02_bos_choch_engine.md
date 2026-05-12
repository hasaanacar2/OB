# BOS / CHOCH Engine — Matematiksel Model

## 1. Amaç

Market structure (piyasa yapısı) değişimlerini tespit etmek. Bu engine iki temel sinyali üretir:
- **BOS (Break of Structure)**: Trend devamı sinyali
- **CHOCH (Change of Character)**: Trend değişim sinyali

---

## 2. Ön Gereksinimler: Swing Tespiti

### 2.1 Swing High (SH)

```
SH_t ⟺ H_t > H_{t-k}  ∧  H_t > H_{t+k}   ∀k ∈ {1, 2, ..., n}
```

### 2.2 Swing Low (SL)

```
SL_t ⟺ L_t < L_{t-k}  ∧  L_t < L_{t+k}   ∀k ∈ {1, 2, ..., n}
```

- `n` = lookback (önerilen: 3-5 candle)
- Daha büyük `n` → daha az ama daha güvenilir swing noktaları

### 2.3 Swing Dizisi

Tüm swing noktaları kronolojik olarak dizilir:

```
S = {(SH₁, t₁), (SL₂, t₂), (SH₃, t₃), (SL₄, t₄), ...}
```

---

## 3. Market Structure Tanımları

### 3.1 Bullish Market Structure

```
HH: SH_current > SH_previous   (Higher High)
HL: SL_current > SL_previous   (Higher Low)
```

**Bullish yapı koşulu:**

```
Bullish ⟺ (SH_n > SH_{n-1}) ∧ (SL_n > SL_{n-1})
```

### 3.2 Bearish Market Structure

```
LH: SH_current < SH_previous   (Lower High)
LL: SL_current < SL_previous   (Lower Low)
```

**Bearish yapı koşulu:**

```
Bearish ⟺ (SH_n < SH_{n-1}) ∧ (SL_n < SL_{n-1})
```

---

## 4. BOS (Break of Structure)

### 4.1 Tanım

BOS, mevcut trendin devam ettiğini gösteren yapısal kırılımdır.

### 4.2 Bullish BOS

```
BOS_bullish ⟺ C_t > SH_previous
```

burada:
- `C_t` = mevcut mum kapanışı
- `SH_previous` = en son swing high

**Ek güvenilirlik koşulu:**

```
C_t > SH_previous  (sadece close ile kırılma, wick değil)
```

Candle body kırma oranı:

```
break_strength = (C_t - SH_previous) / ATR
```

- `break_strength ≥ 0.3` → güvenilir BOS
- `break_strength < 0.1` → zayıf BOS, dikkatli olunmalı

### 4.3 Bearish BOS

```
BOS_bearish ⟺ C_t < SL_previous
```

Break gücü:

```
break_strength = (SL_previous - C_t) / ATR
```

### 4.4 BOS Konfirmasyon Seviyeleri

| Seviye | Koşul | Güvenilirlik |
|---|---|---|
| **Level 1** | Wick kırılma (`H_t > SH` ama `C_t < SH`) | Düşük (sweep olabilir) |
| **Level 2** | Close kırılma (`C_t > SH`) | Orta |
| **Level 3** | Close kırılma + volüm artışı | Yüksek |
| **Level 4** | Close kırılma + impulsive candle | Çok yüksek |

---

## 5. CHOCH (Change of Character)

### 5.1 Tanım

CHOCH, mevcut trendin sona erdiğini ve yeni bir trendin başladığını gösteren ilk yapısal değişim sinyalidir.

### 5.2 Bullish CHOCH (Bearish → Bullish)

Önceki yapı: Lower Highs + Lower Lows

```
CHOCH_bullish ⟺ (Structure = Bearish) ∧ (C_t > SH_previous)
```

Yani:
1. Düşüş yapısı mevcuttu: `SH_n < SH_{n-1}` ve `SL_n < SL_{n-1}`
2. Aniden: `C_t > SH_{n}` → Bearish yapı bozuldu

### 5.3 Bearish CHOCH (Bullish → Bearish)

Önceki yapı: Higher Highs + Higher Lows

```
CHOCH_bearish ⟺ (Structure = Bullish) ∧ (C_t < SL_previous)
```

### 5.4 BOS vs CHOCH Ayrımı

```
if current_trend == "bullish":
    if C_t > SH_previous:
        signal = BOS_bullish      # Trend devam
    if C_t < SL_previous:
        signal = CHOCH_bearish    # Trend değişim

if current_trend == "bearish":
    if C_t < SL_previous:
        signal = BOS_bearish      # Trend devam
    if C_t > SH_previous:
        signal = CHOCH_bullish    # Trend değişim
```

---

## 6. Multi-Timeframe Structure

### 6.1 HTF-LTF Uyumu

```
HTF_bias = structure_direction(higher_timeframe)
LTF_signal = BOS_or_CHOCH(lower_timeframe)
```

**Konfirmasyon matrisi:**

| HTF Bias | LTF Signal | Sonuç |
|---|---|---|
| Bullish | BOS_bullish | ✅ Güçlü long |
| Bullish | CHOCH_bearish | ⚠️ Pullback olabilir |
| Bearish | BOS_bearish | ✅ Güçlü short |
| Bearish | CHOCH_bullish | ⚠️ Bounce olabilir |
| Bullish | BOS_bearish | ❌ Counter-trend, riskli |
| Bearish | BOS_bullish | ❌ Counter-trend, riskli |

### 6.2 Structure Mapping

Her timeframe için bağımsız yapı takibi:

```
Structure_Map = {
    "4h": {trend: "bullish", last_SH: x, last_SL: y},
    "1h": {trend: "bearish", last_SH: x, last_SL: y},
    "15m": {trend: "bullish", last_SH: x, last_SL: y},
    "5m": {trend: "bearish", last_SH: x, last_SL: y}
}
```

---

## 7. Structure Strength Skoru

```
SS = w₁ × break_strength + w₂ × volume_factor + w₃ × impulse_factor + w₄ × htf_alignment
```

burada:
- `break_strength = |C_t - swing_level| / ATR`
- `volume_factor = volume_t / SMA(volume, 20)`
- `impulse_factor = candle_range / ATR` (≥ 1.5 ise güçlü)
- `htf_alignment = 1 (uyumlu), 0 (nötr), -0.5 (ters)`
- Ağırlıklar: `w₁ = 0.3, w₂ = 0.2, w₃ = 0.3, w₄ = 0.2`

---

## 8. Algoritmik Akış

```
1. Swing High / Swing Low tespit et
2. Mevcut market structure'ı belirle (HH/HL veya LH/LL)
3. Her yeni candle'da:
   a. Close, son swing high'ı kırıyor mu? → BOS veya CHOCH
   b. Close, son swing low'u kırıyor mu? → BOS veya CHOCH
4. BOS/CHOCH tipini belirle (trend yönüne göre)
5. Break strength hesapla
6. HTF uyumunu kontrol et
7. Structure Strength skoru hesapla
8. Signal üret ve diğer engine'lere bildir
```

---

## 9. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `SWING_LOOKBACK` | 3 | Swing tespit lookback |
| `MIN_BREAK_STRENGTH` | 0.1 | Minimum kırılma gücü (ATR oranı) |
| `BOS_CONFIRM_TYPE` | "close" | Kırılma tipi: "close" veya "wick" |
| `STRUCTURE_MEMORY` | 50 | Kaç swing noktası hatırlanacak |
| `HTF_TIMEFRAMES` | ["4h", "1h"] | HTF referans timeframe'ler |

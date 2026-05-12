# Execution Engine — Matematiksel Model

## 1. Amaç

Trade sinyallerini deterministik emirlere çevirmek. Sinyal doğru olsa bile kötü execution ciddi kayıplara yol açar. Bu engine giriş, çıkış, partial close ve trailing stop mantığını formalize eder.

---

## 2. Entry Hesaplamaları

### 2.1 Limit Order Entry

OB retest senaryosunda:

```
Entry_Price = OB_Mid = (OB_High + OB_Low) / 2
```

Alternatif entry seviyeleri:

```
Entry_Conservative = OB_Low + (OB_High - OB_Low) × 0.3    # OB'nin %30'u (bullish)
Entry_Optimal      = OB_Low + (OB_High - OB_Low) × 0.5    # OB'nin %50'si
Entry_Aggressive   = OB_Low + (OB_High - OB_Low) × 0.7    # OB'nin %70'i
```

### 2.2 Market Order vs Limit Order

| Durum | Emir Tipi | Gerekçe |
|---|---|---|
| OB retest | Limit | Belirli seviyede giriş |
| Sweep + hızlı BOS | Market | Kaçırma riski yüksek |
| CHOCH konfirmasyonu | Limit (retrace bekle) | Daha iyi RR |

### 2.3 Slippage Modeli

```
Effective_Entry = Intended_Entry ± Slippage
```

```
Expected_Slippage = f(volume, spread, order_size)
```

Basit yaklaşım:

```
Slippage_Estimate = Spread × (1 + Order_Size / Avg_Volume × k)
```

burada `k` = piyasa etki katsayısı (genelde 0.5-2.0)

---

## 3. Stop-Loss Hesaplamaları

### 3.1 OB Bazlı Stop-Loss

**Bullish trade:**

```
SL = OB_Low - ATR × buffer
```

**Bearish trade:**

```
SL = OB_High + ATR × buffer
```

- `buffer` = 0.1 - 0.3 (önerilen: 0.2)

### 3.2 Swing Bazlı Stop-Loss

```
SL_bullish = Recent_Swing_Low - ATR × 0.2
SL_bearish = Recent_Swing_High + ATR × 0.2
```

### 3.3 Stop-Loss Mesafe Kontrolü

```
SL_Distance = |Entry - SL|

if SL_Distance > ATR × 3:
    REJECT (SL çok uzak, risk/reward bozulur)
    
if SL_Distance < ATR × 0.3:
    REJECT (SL çok yakın, noise ile vurulur)
```

---

## 4. Take-Profit Hesaplamaları

### 4.1 RR Bazlı TP

```
Risk = |Entry - SL|
TP = Entry + Risk × RR_Target × direction
```

burada `direction = +1 (long), -1 (short)`

### 4.2 Liquidity Bazlı TP

```
TP = Nearest_Liquidity_Level
```

Liquidity seviyeleri: equal highs/lows, session highs/lows, PDH/PDL

### 4.3 Fibonacci Extension TP

```
TP_1 = Swing_Low + (Swing_High - Swing_Low) × 1.272
TP_2 = Swing_Low + (Swing_High - Swing_Low) × 1.618
TP_3 = Swing_Low + (Swing_High - Swing_Low) × 2.618
```

### 4.4 TP Seçim Matrisi

```
Final_TP = min(RR_TP, Liquidity_TP)
```

Çünkü fiyat likidite seviyesine ulaştığında reaksiyon gösterebilir.

---

## 5. Partial Close (Kısmi Kapatma)

### 5.1 Sabit Oranlı Partial Close

```
TP₁'de kapat: %40 pozisyon  (1:1 RR)
TP₂'de kapat: %30 pozisyon  (1:2 RR)
TP₃'de kapat: %30 pozisyon  (1:3 RR)
```

### 5.2 Dinamik Partial Close

```
Close_Ratio = min(0.5, profit_pips / (TP_distance × 2))
```

Profit arttıkça daha fazla kapatılır.

### 5.3 Partial Close Sonrası SL Güncelleme

```
if Partial_1_hit:
    SL = Entry_Price    # Breakeven'a çek
    
if Partial_2_hit:
    SL = TP₁_Price      # İlk TP seviyesine çek
```

---

## 6. Trailing Stop

### 6.1 ATR Trailing Stop

```
Trail_Stop_Long = Highest_Close_Since_Entry - ATR × trail_multiplier
Trail_Stop_Short = Lowest_Close_Since_Entry + ATR × trail_multiplier
```

- `trail_multiplier` = 1.5 - 3.0

### 6.2 Swing-Based Trailing Stop

```
Trail_Stop_Long = Last_Confirmed_Swing_Low
Trail_Stop_Short = Last_Confirmed_Swing_High
```

### 6.3 Chandelier Exit

```
Chandelier_Long = Highest_High(n) - ATR × multiplier
Chandelier_Short = Lowest_Low(n) + ATR × multiplier
```

### 6.4 Trailing Aktivasyon

Trailing stop ancak belirli bir profit seviyesinden sonra aktif olur:

```
if unrealized_profit ≥ Risk × activation_rr:
    activate_trailing_stop()
```

Önerilen `activation_rr = 1.5`

---

## 7. Order Cancellation

### 7.1 Zaman Bazlı İptal

```
if time_since_order > MAX_PENDING_TIME:
    CANCEL_ORDER()
```

| Timeframe | Max Pending Time |
|---|---|
| 5m | 30 dakika |
| 15m | 2 saat |
| 1h | 8 saat |
| 4h | 24 saat |

### 7.2 Koşul Bazlı İptal

```
if BOS_reversed:
    CANCEL_ORDER()    # Market structure değişti

if price_moved > ATR × 2 from entry zone:
    CANCEL_ORDER()    # Fiyat OB'den çok uzaklaştı

if new_signal_conflicts:
    CANCEL_ORDER()    # Çelişen sinyal geldi
```

---

## 8. RR (Risk-Reward) Hesaplama

```
RR = |TP - Entry| / |Entry - SL|
```

**Minimum RR filtresi:**

```
if RR < MIN_RR (2.0):
    REJECT_TRADE()
```

**Gerçek RR (trade kapandıktan sonra):**

```
Realized_RR = |Exit - Entry| / |Entry - SL|
```

---

## 9. Emir Tipleri ve Seçim Mantığı

```
Order_Type = f(signal_urgency, market_conditions, strategy)
```

| Senaryo | Order Type | Stop Type | TP Type |
|---|---|---|---|
| OB Retest | LIMIT | STOP_MARKET | LIMIT |
| Sweep + BOS | MARKET | STOP_MARKET | LIMIT |
| Breakout | STOP_LIMIT | STOP_MARKET | LIMIT |

---

## 10. Algoritmik Akış

```
1. Signal al (yön, entry zone, SL zone, TP target)
2. Entry price hesapla (OB mid veya optimal entry)
3. SL hesapla (OB dışı + ATR buffer)
4. TP hesapla (RR bazlı veya liquidity bazlı)
5. RR kontrol et (≥ 2.0 olmalı)
6. SL mesafe kontrol et (noise aralığında olmamalı)
7. Slippage tahmini ekle
8. Order type belirle
9. Risk Engine'den onay al (position size)
10. Emri gönder
11. Trailing stop ve partial close kurallarını aktifleştir
12. Trade Lifecycle Engine'e devret
```

---

## 11. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `OB_ENTRY_LEVEL` | 0.5 | OB zone'daki entry seviyesi (0-1) |
| `SL_ATR_BUFFER` | 0.2 | SL için ATR buffer çarpanı |
| `MIN_RR` | 2.0 | Minimum risk-reward oranı |
| `PARTIAL_1_RATIO` | 0.4 | İlk partial close oranı |
| `PARTIAL_1_RR` | 1.0 | İlk partial close RR seviyesi |
| `TRAIL_ATR_MULT` | 2.0 | Trailing stop ATR çarpanı |
| `TRAIL_ACTIVATION_RR` | 1.5 | Trailing stop aktivasyon RR |
| `MAX_SL_DISTANCE_ATR` | 3.0 | Max SL mesafesi (ATR çarpanı) |
| `MIN_SL_DISTANCE_ATR` | 0.3 | Min SL mesafesi (ATR çarpanı) |

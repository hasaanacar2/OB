# Trade Lifecycle Engine — Matematiksel Model

## 1. Amaç

Trade açıldıktan sonraki tüm yönetim sürecini formalize etmek. Entry kalitesi kadar exit ve pozisyon yönetimi de karlılığı belirler.

---

## 2. Lifecycle Durumları (State Machine)

```
PENDING → ACTIVE → PARTIAL → TRAILING → CLOSED
              ↓                    ↓
           STOPPED             STOPPED
              ↓                    ↓
           CLOSED              CLOSED
```

### 2.1 State Tanımları

| State | Açıklama |
|---|---|
| `PENDING` | Limit order bekliyor, henüz fill olmadı |
| `ACTIVE` | Pozisyon açık, SL ve TP yerinde |
| `PARTIAL` | İlk TP alındı, kalan pozisyon devam |
| `TRAILING` | Trailing stop aktif |
| `STOPPED` | SL vuruldu |
| `CLOSED` | Trade tamamen kapandı |

### 2.2 State Transition Kuralları

```
PENDING → ACTIVE:    limit_order_filled = true
PENDING → CLOSED:    cancel_conditions_met = true
ACTIVE → PARTIAL:    price_hit_TP1 = true
ACTIVE → STOPPED:    price_hit_SL = true
ACTIVE → CLOSED:     manual_close OR time_limit OR structure_break
PARTIAL → TRAILING:  price_hit_TP2 OR trailing_activation = true
PARTIAL → STOPPED:   price_hit_new_SL (breakeven)
TRAILING → CLOSED:   trailing_stop_hit = true
```

---

## 3. Entry Aşaması (PENDING → ACTIVE)

### 3.1 Fill Kontrolü

```
Bullish limit order fill:
    fill ⟺ L_t ≤ Entry_Price

Bearish limit order fill:
    fill ⟺ H_t ≥ Entry_Price
```

### 3.2 Pending Timeout

```
if (current_time - order_time) > MAX_PENDING_TIME:
    cancel_order()
    state = CLOSED
```

### 3.3 Entry Kalite Kaydı

```
Entry_Quality = {
    signal_score: int,
    entry_price: float,
    ideal_entry: float,
    slippage: entry_price - ideal_entry,
    time_to_fill: fill_time - order_time,
    market_regime: string
}
```

---

## 4. Aktif Pozisyon Yönetimi (ACTIVE)

### 4.1 Unrealized PnL

```
Long:  PnL_unrealized = (Current_Price - Entry_Price) × Position_Size
Short: PnL_unrealized = (Entry_Price - Current_Price) × Position_Size
```

### 4.2 R-Multiple Tracking

```
R = |Entry - SL|    (1 birim risk)

Current_R_Multiple = PnL_unrealized / R
```

Negatif R = kayıpta, pozitif R = kârda

### 4.3 MAE / MFE Tracking

Her candle'da güncellenir:

```
MAE = max(MAE_prev, -Current_R_Multiple)    # En kötü nokta
MFE = max(MFE_prev, Current_R_Multiple)     # En iyi nokta
```

---

## 5. Partial Take-Profit (ACTIVE → PARTIAL)

### 5.1 Partial Close Seviyeleri

```
TP₁: RR = 1.0 → Close %40 pozisyon
TP₂: RR = 2.0 → Close %30 pozisyon
TP₃: RR = 3.0 → Close %30 pozisyon (veya trailing)
```

### 5.2 Partial Close Sonrası Güncelleme

```
Remaining_Size = Original_Size × (1 - Partial_Ratio)

Realized_PnL += Partial_Size × (TP_Price - Entry_Price)

if TP₁_hit:
    SL = Entry_Price    # Breakeven
    
if TP₂_hit:
    SL = TP₁_Price      # Lock profit
```

### 5.3 Weighted Average Exit

```
Avg_Exit = Σ (Exit_Price_i × Size_i) / Σ Size_i
```

---

## 6. Breakeven Yönetimi

### 6.1 Breakeven Koşulu

```
if Current_R ≥ BE_Activation_R:
    New_SL = Entry_Price + (Entry_Fee × 2 / Position_Size)
```

Komisyon dahil breakeven:

```
True_Breakeven = Entry_Price + Total_Fees / Position_Size
```

### 6.2 Breakeven + Buffer

```
New_SL = Entry_Price + ATR × 0.05    # Küçük buffer
```

Tam entry'ye koyma: noise ile vurulma riski yüksek.

---

## 7. Trailing Stop (PARTIAL → TRAILING)

### 7.1 ATR Trailing

```
Long:
    Trail_SL_t = max(Trail_SL_{t-1}, Close_t - ATR × trail_mult)
    
Short:
    Trail_SL_t = min(Trail_SL_{t-1}, Close_t + ATR × trail_mult)
```

Önemli: Trailing stop sadece pozisyon lehine hareket eder, asla geriler.

### 7.2 Parabolic SAR Trailing

```
SAR_t = SAR_{t-1} + AF × (EP - SAR_{t-1})
```

burada:
- `AF` = acceleration factor (başlangıç 0.02, her yeni EP'de +0.02, max 0.2)
- `EP` = extreme point (trend yönündeki en uç nokta)

### 7.3 Swing-Based Trailing

```
Long:  Trail_SL = Most_Recent_Swing_Low - ATR × 0.1
Short: Trail_SL = Most_Recent_Swing_High + ATR × 0.1
```

### 7.4 Hibrit Trailing

```
Trail_SL = max(ATR_Trail, Swing_Trail)    # Long için
Trail_SL = min(ATR_Trail, Swing_Trail)    # Short için
```

En koruyucu olanı seç.

---

## 8. Exit Koşulları

### 8.1 Normal Exit

```
EXIT if:
    price_hit_SL                           # Stop-loss
    price_hit_TP (final)                   # Son take-profit
    trailing_stop_triggered                # Trailing vuruldu
```

### 8.2 Erken Exit Koşulları

```
EARLY_EXIT if:
    opposite_CHOCH_detected               # Ters CHOCH (yapı değişimi)
    time_elapsed > MAX_HOLD_TIME           # Süre doldu
    risk_engine_halt                       # Risk limiti aşıldı
    news_volatility_detected              # Haber volatilitesi
```

### 8.3 Strategy Stickiness (Stratejiye Sadakat)

Eğer bir işlem aktifse (ACTIVE state), rejim motoru (Regime Engine) o an kısa süreliğine başka bir rejime geçse bile işlem KESİNLİKLE anında kapatılmaz veya strateji değiştirilmez. 

Bir pozisyon, sadece ve sadece şu durumlarda terk edilir:
1. Analizin temeli çürümüşse (Opposite CHOCH).
2. Orijinal stratejinin zarar kes (SL) seviyesi vurulmuşsa.
3. Çok uzun süre geçmiş ve fiyat gitmemişse (Time Decay).

*1 saatlik bir volatilite veya geçici hacim düşüklüğü, açılmış bir pozisyonu kapattırıp botu başka bir stratejiye atlatmamalıdır (No Strategy Churn).*

### 8.4 Zaman Bazlı Bozunma

Trade yaşlandıkça, beklenen edge azalır:

```
Time_Decay = 1 - (elapsed_time / max_hold_time)²

if Time_Decay < 0.3:
    TIGHTEN_SL()    # SL'yi sıkılaştır
    
if Time_Decay < 0.1:
    CLOSE_AT_MARKET()    # Kapat
```

---

## 9. Trade Sonucu Kaydı

```
Trade_Result = {
    id: unique_id,
    symbol: string,
    direction: "long" | "short",
    entry_price: float,
    exit_price: float,
    position_size: float,
    
    sl_original: float,
    sl_final: float,
    tp_targets: [float],
    
    pnl_realized: float,
    pnl_percentage: float,
    r_multiple: float,
    
    mae: float,            # Max adverse excursion
    mfe: float,            # Max favorable excursion
    
    entry_time: datetime,
    exit_time: datetime,
    hold_duration: timedelta,
    
    signal_score: int,
    exit_reason: string,    # "tp" | "sl" | "trailing" | "time" | "manual" | "structure"
    market_regime: string,
    
    partial_fills: [{price, size, time}],
    fees_paid: float
}
```

---

## 10. Lifecycle Metrikleri

### 10.1 Hold Time Analysis

```
Avg_Hold_Win = mean(hold_time | winning trades)
Avg_Hold_Loss = mean(hold_time | losing trades)
```

Ideal: `Avg_Hold_Win > Avg_Hold_Loss` (winners daha uzun tutulmalı)

### 10.2 Exit Efficiency

```
Exit_Efficiency = Realized_PnL / MFE
```

- 1.0 = mükemmel (en iyi noktada çıkıldı)
- 0.5 = orta (potansiyelin yarısı yakalandı)
- < 0.3 = kötü exit management

### 10.3 Entry Efficiency

```
Entry_Efficiency = 1 - (MAE / Risk)
```

- 1.0 = mükemmel (hiç adverse excursion yok)
- > 0.7 = iyi entry

---

## 11. Algoritmik Akış

```
1. Signal gelir → PENDING state
2. Fill olursa → ACTIVE state, SL/TP set et
3. Her candle'da:
   a. SL/TP kontrolü
   b. MAE/MFE güncelle
   c. Time decay hesapla
   d. Structure break kontrolü
4. TP₁ hit → PARTIAL state, SL = breakeven
5. TP₂ hit → TRAILING state
6. Trail stop aktif → her candle trail güncelle
7. Exit tetiklenir → CLOSED state
8. Trade result kaydet
```

---

## 12. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `BE_ACTIVATION_R` | 1.0 | Breakeven aktivasyon (R-multiple) |
| `BE_BUFFER_ATR` | 0.05 | Breakeven buffer (ATR çarpanı) |
| `TRAIL_ATR_MULT` | 2.0 | Trailing stop ATR çarpanı |
| `TRAIL_ACTIVATION_R` | 1.5 | Trailing aktivasyon R-multiple |
| `TIME_DECAY_THRESHOLD` | 0.3 | SL sıkılaştırma eşiği |
| `MAX_HOLD_TIME_5M` | 2 saat | 5m setup max süre |
| `MAX_HOLD_TIME_15M` | 6 saat | 15m setup max süre |
| `MAX_HOLD_TIME_1H` | 24 saat | 1h setup max süre |
| `MAX_HOLD_TIME_4H` | 7 gün | 4h setup max süre |

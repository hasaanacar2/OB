# Risk Engine — Matematiksel Model

## 1. Amaç

Sistemin uzun vadede hayatta kalmasını garanti altına almak. Hiçbir single trade veya kötü gün serisi hesabı sıfırlamamalıdır. Risk Engine, diğer tüm engine'lerin üzerinde veto yetkisine sahip tek engine'dir.

---

## 2. Position Sizing

### 2.1 Fixed Fractional Model

Her trade'de risk edilen tutar:

```
Risk_Amount = Account_Balance × Risk_Percent
```

```
Position_Size = Risk_Amount / (Entry_Price - Stop_Loss_Price)
```

Kontrat bazlı:

```
Contracts = Risk_Amount / (|Entry - SL| × Contract_Size)
```

**Örnek:**
- Balance = $10,000
- Risk = 0.5%
- Risk_Amount = $50
- Entry = $100, SL = $98 → |Entry - SL| = $2
- Position_Size = $50 / $2 = 25 kontrat

### 2.2 Volatilite Bazlı Position Sizing

ATR kullanarak dinamik sizing:

```
Position_Size = Risk_Amount / (ATR × ATR_Multiplier)
```

Yüksek volatilitede pozisyon küçülür, düşük volatilitede büyür.

### 2.3 Kelly Criterion (İleri Seviye)

```
K = W - (1 - W) / R
```

burada:
- `K` = portföyün yüzde kaçının riske edileceği
- `W` = win rate (kazanma oranı)
- `R` = average win / average loss

**Half-Kelly (Pratikte önerilen):**

```
K_practical = K / 2
```

Çünkü tam Kelly çok agresiftir ve parameter uncertainty yüksektir.

**Örnek:**
- Win Rate = 55%, Avg Win = $150, Avg Loss = $100
- R = 150/100 = 1.5
- K = 0.55 - (0.45 / 1.5) = 0.55 - 0.30 = 0.25
- Half-Kelly = 12.5%

---

## 3. Risk Limitleri

### 3.1 Trade Bazlı Limitler

```
MAX_RISK_PER_TRADE = 0.5%    # Tek trade risk limiti
MAX_POSITION_SIZE = 5%        # Tek pozisyon büyüklüğü (bakiye %)
MAX_LEVERAGE = 10x            # Maksimum kaldıraç
```

### 3.2 Günlük Limitler

```
MAX_DAILY_LOSS = 3%           # Günlük kayıp limiti
MAX_DAILY_TRADES = 10         # Günlük trade sayısı
```

**Günlük kayıp kontrolü:**

```
Daily_PnL = Σ PnL_i    (i = tüm günlük kapanan trade'ler)

if |Daily_PnL| ≥ Account_Balance × MAX_DAILY_LOSS:
    HALT_TRADING()
```

### 3.3 Haftalık / Aylık Limitler

```
MAX_WEEKLY_LOSS = 7%
MAX_MONTHLY_LOSS = 15%
```

### 3.4 Ardışık Kayıp Limiti

```
MAX_CONSECUTIVE_LOSSES = 5

if consecutive_losses ≥ MAX_CONSECUTIVE_LOSSES:
    REDUCE_SIZE(50%)    # Pozisyon boyutunu yarıya düşür
    COOLDOWN(2_hours)   # 2 saat bekleme süresi
```

### 3.5 Minimum Expected Edge (Slippage ve Komisyon Koruması)
Sistem sadece kağıt üzerindeki kâra değil, "Net Kâra" (Slippage + Komisyon Düşülmüş) odaklanır. Bir işlemin hedef kârı (TP), o işlemin maliyetinin en az 3 katı olmak zorundadır. Aksi halde işlem VETO edilir.
```
Total_Cost = Maker_Fee + Taker_Fee + Estimated_Slippage
if Expected_Profit_Percent < Total_Cost × 3:
    REJECT_ORDER ("Kâr marjı, komisyon ve kaymayı kurtarmıyor. Çöp işlem.")
```

---

## 4. Drawdown Kontrolü

### 4.1 Drawdown Hesaplama

```
Peak_Balance = max(Balance_history)
Drawdown = (Peak_Balance - Current_Balance) / Peak_Balance × 100
```

### 4.2 Drawdown Seviyeleri

| Drawdown | Aksiyon |
|---|---|
| < 5% | Normal operasyon |
| 5% - 10% | Pozisyon boyutu %50 azalt |
| 10% - 15% | Sadece en güçlü sinyaller (score ≥ 85) |
| 15% - 20% | Trading durdur, analiz yap |
| > 20% | Sistem kapat, manual review |

### 4.3 Dinamik Risk Azaltma

```
if Drawdown > DD_Threshold_1:
    Risk_Per_Trade = Base_Risk × (1 - (Drawdown - DD_Threshold_1) / (DD_Max - DD_Threshold_1))
```

Doğrusal azaltma: drawdown arttıkça risk per trade azalır.

---

## 5. Leverage Kontrolü

### 5.1 Efektif Leverage

```
Effective_Leverage = Total_Position_Value / Account_Balance
```

### 5.2 Dinamik Leverage Limiti

```
Max_Leverage = Base_Leverage × (1 - Drawdown_Percentage / 100)
```

**Örnek:**
- Base Leverage = 10x
- Drawdown = 8%
- Max_Leverage = 10 × (1 - 0.08) = 9.2x

### 5.3 Volatilite Bazlı Leverage

```
Vol_Adjusted_Leverage = Base_Leverage × (Avg_ATR / Current_ATR)
```

Yüksek volatilitede leverage otomatik düşer.

---

## 6. Korelasyon Riski

### 6.1 Açık Pozisyon Korelasyonu

```
Total_Correlated_Risk = Σ |Risk_i|   i ∈ correlated_positions
```

burada korelasyon:

```
ρ(A, B) = Cov(R_A, R_B) / (σ_A × σ_B)
```

### 6.2 Korelasyon Limiti

```
if |ρ(A, B)| > 0.7:
    # A ve B yüksek korelasyonlu
    Total_Risk(A + B) ≤ MAX_RISK_PER_TRADE × 1.5    # Bağımsız değil
```

### 6.3 Duplicate Signal & Over-Trigger Lockout (Mükerrer İşlem Koruması)

Aynı fiyat bölgesinden (OB) peş peşe gelen sinyallere tekrar tekrar işlem açılması (Spam/Pyramiding) kesinlikle engellenmelidir.

```
if current_signal.coin == active_position.coin ∧ 
   |current_signal.entry - active_position.entry| < (ATR × 0.5) ∧
   current_signal.direction == active_position.direction:
    
    REJECT_ORDER ("Duplicate Entry Zone - Zaten İçeridesin")
```

Ayrıca, bir işlem o bölgede Stop-Loss (SL) olduysa ve fiyat hala aynı OB içinde dolanmaya devam ediyorsa, o bölge (Zone) "Tükenmiş" (Mitigated/Fail) kabul edilir. Aynı bölgeden en az `COOLDOWN_PERIOD` (örn. 2 saat) geçmeden yeni bir işlem açılmaz.

---

## 7. Risk-Adjusted Return Metrikleri

### 7.1 Sharpe Ratio

```
Sharpe = (R̄ - R_f) / σ_R
```

- `R̄` = ortalama getiri
- `R_f` = risksiz getiri (kripto için genelde 0)
- `σ_R` = getiri standart sapması

### 7.2 Sortino Ratio

```
Sortino = (R̄ - R_f) / σ_downside
```

Sadece negatif getirilerin volatilitesini kullanır — Risk Engine için daha uygun.

### 7.3 Calmar Ratio

```
Calmar = Annualized_Return / Max_Drawdown
```

### 7.4 Profit Factor

```
PF = Σ Winning_Trades / |Σ Losing_Trades|
```

- PF > 1.5 → kabul edilebilir
- PF > 2.0 → iyi
- PF > 3.0 → mükemmel

---

## 8. Hard vs Soft Limitler

| Limit Tipi | Davranış | Örnekler |
|---|---|---|
| **Hard Limit** | Anında trading durdur, override yok | Max daily loss, max drawdown |
| **Soft Limit** | Uyarı ver, pozisyon küçült | Ardışık kayıp, yüksek volatilite |

```
Hard_Limits:
    daily_loss ≥ 3%     → HALT
    drawdown ≥ 20%      → HALT
    leverage > 10x      → REJECT_ORDER

Soft_Limits:
    consecutive_loss ≥ 3  → WARNING + REDUCE_SIZE
    daily_trades ≥ 8      → WARNING
    volatility > 2×avg    → REDUCE_LEVERAGE
```

### 8.1 Human Override (God Mode)
Risk Engine sistemdeki en yetkili motor olmasına rağmen, tek bir istisnası vardır: **Yönetici Onayı**.
Eğer Claude veya Risk Motoru bir işlemi (Örn: Korelasyon limiti veya Claude tereddütü yüzünden) reddederse ve bu işlem Telegram üzerinden Yöneticiye (Size) sorulursa; sizin vereceğiniz `ONAYLA` komutu Risk Engine'in tüm Soft ve Hard limitlerini (Drawdown ve Bakiye yetersizliği hariç) ezip geçer (Override). Yönetici kararı mutlaktır.

---

## 9. Algoritmik Akış

```
1. Trade sinyali geldiğinde:
   a. Günlük PnL kontrol et
   b. Drawdown kontrol et
   c. Açık pozisyon sayısı kontrol et
   d. Korelasyon riski kontrol et
   e. Leverage kontrol et
   
2. Hard limit ihlali varsa → REJECT

3. Position size hesapla:
   a. Fixed fractional veya ATR-based
   b. Drawdown azaltma uygula
   c. Leverage limiti uygula

4. Soft limit uyarıları oluştur

5. Onaylanmış pozisyon boyutu döndür
```

---

## 10. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `MAX_RISK_PER_TRADE` | 0.5% | Trade başına max risk |
| `MAX_DAILY_LOSS` | 3.0% | Günlük max kayıp |
| `MAX_WEEKLY_LOSS` | 7.0% | Haftalık max kayıp |
| `MAX_MONTHLY_LOSS` | 15.0% | Aylık max kayıp |
| `MAX_DRAWDOWN` | 20.0% | Max drawdown |
| `MAX_LEVERAGE` | 10x | Max kaldıraç |
| `MAX_CONSECUTIVE_LOSSES` | 5 | Ardışık kayıp limiti |
| `MAX_OPEN_POSITIONS` | 3 | Eşzamanlı açık pozisyon |
| `ATR_MULTIPLIER` | 1.5 | ATR bazlı SL çarpanı |
| `COOLDOWN_PERIOD` | 2 saat | Kayıp sonrası bekleme |

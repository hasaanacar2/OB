# Replay / Backtest Engine — Matematiksel Model

## 1. Amaç

Stratejilerin geçmiş veriler üzerinde simüle edilmesiyle gerçek edge'in ölçülmesi. Canlı markette öğrenmek pahalıdır; bu engine, her strateji değişikliğinin istatistiksel olarak test edilmesini sağlar.

---

## 2. Backtest Çerçevesi

### 2.1 Veri Hazırlama

```
Dataset = {(O_t, H_t, L_t, C_t, V_t, T_t) | t = 1, 2, ..., N}
```

**Minimum veri gereksinimleri:**

| Timeframe | Minimum Geçmiş | Önerilen Geçmiş |
|---|---|---|
| 5m | 30 gün | 90 gün |
| 15m | 60 gün | 180 gün |
| 1h | 180 gün | 365 gün |
| 4h | 365 gün | 730 gün |

### 2.2 Event-Driven Backtest

Her candle'da sıralı kontrol:

```
for t in range(lookback, N):
    # 1. Pending orders kontrol (SL, TP, limit fills)
    check_pending_orders(candle_t)
    
    # 2. Trailing stop güncelle
    update_trailing_stops(candle_t)
    
    # 3. Yeni sinyal üret
    signal = generate_signal(data[:t+1])
    
    # 4. Risk kontrol ve emir gönder
    if signal and risk_engine.approve(signal):
        execute_order(signal)
```

### 2.3 Look-Ahead Bias Önleme

```
Kural: t anında yalnızca data[0:t] kullanılabilir
data[t+1:] kesinlikle erişilemez
```

Her hesaplama rolling window ile yapılır:

```
ATR_t = ATR(data[t-14:t])     # ✅ Doğru
ATR_t = ATR(data[t-14:t+1])   # ❌ Yanlış (gelecek veri)
```

---

## 3. Performans Metrikleri

### 3.1 Temel Metrikler

**Win Rate:**

```
WR = Winning_Trades / Total_Trades × 100
```

**Profit Factor:**

```
PF = Σ Gross_Profit / |Σ Gross_Loss|
```

**Average RR (Gerçekleşen):**

```
Avg_RR = mean(|Exit_i - Entry_i| / |Entry_i - SL_i|)   ∀ winning trades
```

**Expectancy (Beklenen Değer):**

```
E = (WR × Avg_Win) - ((1 - WR) × Avg_Loss)
```

veya risk birimi olarak:

```
E_R = (WR × Avg_RR_Win) - ((1 - WR) × 1)
```

burada 1R = bir birim risk

### 3.2 Drawdown Metrikleri

**Maximum Drawdown:**

```
MDD = max((Peak_i - Trough_i) / Peak_i)   ∀ i ∈ peak-trough pairs
```

**Maximum Drawdown Süresi:**

```
MDD_Duration = max(t_recovery - t_peak)   ∀ drawdown periods
```

**Average Drawdown:**

```
Avg_DD = mean(all drawdown magnitudes)
```

### 3.3 Risk-Adjusted Metrikler

**Sharpe Ratio (Annualized):**

```
Sharpe = (R̄_daily × 252) / (σ_daily × √252)
```

Kripto 24/7 olduğu için:

```
Sharpe_crypto = (R̄_daily × 365) / (σ_daily × √365)
```

**Sortino Ratio:**

```
Sortino = R̄ / σ_downside
```

burada:

```
σ_downside = √(Σ min(R_i, 0)² / n)
```

**Calmar Ratio:**

```
Calmar = Annualized_Return / MDD
```

### 3.4 Trade Kalitesi Metrikleri

**MAE (Maximum Adverse Excursion):**

```
MAE_i = max drawdown during trade_i before exit
```

Düşük MAE = iyi entry timing

**MFE (Maximum Favorable Excursion):**

```
MFE_i = max profit during trade_i before exit
```

Yüksek MFE ile düşük realized profit = kötü exit timing

**Edge Ratio:**

```
Edge_Ratio = median(MFE) / median(MAE)
```

- Edge Ratio > 1.5 → iyi edge
- Edge Ratio < 1.0 → edge yok

---

## 4. İstatistiksel Anlamlılık

### 4.1 Minimum Trade Sayısı

```
N_min = (z² × p × (1 - p)) / e²
```

burada:
- `z` = güven aralığı z-değeri (1.96 for 95%)
- `p` = beklenen win rate
- `e` = hata marjı

**Örnek:** WR = 55%, e = 5%, z = 1.96

```
N_min = (1.96² × 0.55 × 0.45) / 0.05² = 380 trade
```

### 4.2 T-Test (Strateji Karlılığı)

Null hipotez: Strateji ortalaması ≤ 0

```
t = R̄ / (σ / √n)
```

- `t > 1.96` (p < 0.05) → İstatistiksel olarak karlı
- `t > 2.576` (p < 0.01) → Güçlü kanıt

### 4.3 Monte Carlo Simülasyonu

Backtest sonuçlarının robustluğunu test etmek:

```
for simulation in range(10000):
    shuffled_returns = random.shuffle(trade_returns)
    equity_curve = cumsum(shuffled_returns)
    record(max_drawdown, final_return, sharpe)
    
confidence_interval_95 = percentile(results, [2.5, 97.5])
```

---

## 5. Walk-Forward Optimization

### 5.1 Yapı

```
Total_Data = [────────────────────────────────────]

Window 1:   [===TRAIN===][=TEST=]
Window 2:        [===TRAIN===][=TEST=]
Window 3:             [===TRAIN===][=TEST=]
Window 4:                  [===TRAIN===][=TEST=]
```

### 5.2 Parametreler

```
Train_Period = 60 gün
Test_Period = 20 gün
Step_Size = 20 gün
```

### 5.3 Walk-Forward Efficiency

```
WFE = Out_of_Sample_Return / In_Sample_Return
```

- WFE > 0.5 → kabul edilebilir
- WFE > 0.7 → iyi
- WFE < 0.3 → overfit

---

## 6. Regime-Specific Backtest

### 6.1 Market Rejimi Bazlı Analiz

```
for regime in [TRENDING, RANGING, HIGH_VOL, LOW_VOL]:
    filtered_data = data[data.regime == regime]
    results[regime] = backtest(strategy, filtered_data)
```

### 6.2 Karşılaştırma Tablosu

| Metrik | Trending | Ranging | High Vol | Low Vol |
|---|---|---|---|---|
| Win Rate | ? | ? | ? | ? |
| Profit Factor | ? | ? | ? | ? |
| Sharpe | ? | ? | ? | ? |
| Max DD | ? | ? | ? | ? |

---

## 7. Backtest Tuzakları ve Çözümler

| Tuzak | Çözüm |
|---|---|
| **Look-ahead bias** | Sadece `data[:t]` kullan |
| **Survivorship bias** | Delisted coinleri dahil et |
| **Overfitting** | Walk-forward validation |
| **Slippage yok sayma** | Slippage modeli ekle |
| **Commission yok sayma** | Her trade'e fee ekle |
| **Fill assumption** | Limit order fill olasılığı modelle |

### 7.1 Fee Modeli

```
Total_Cost = Entry_Fee + Exit_Fee + Funding_Cost

Entry_Fee = Position_Size × Fee_Rate    # Maker: 0.02%, Taker: 0.04%
Exit_Fee = Position_Size × Fee_Rate
Funding_Cost = Position_Size × Funding_Rate × (Hold_Time / 8h)
```

### 7.2 Slippage Modeli

```
Simulated_Slippage = ATR × slippage_factor
```

burada `slippage_factor = 0.01 - 0.05` (piyasa koşullarına göre)

---

## 8. Algoritmik Akış

```
1. Geçmiş veri yükle (OHLCV + ek veri)
2. Train / Test split veya walk-forward window tanımla
3. Event-driven backtest çalıştır:
   a. Her candle'da sinyal üret
   b. Risk Engine simüle et
   c. Execution Engine simüle et (slippage + fee dahil)
   d. Trade sonuçlarını kaydet
4. Performans metrikleri hesapla
5. İstatistiksel anlamlılık test et
6. Regime bazlı analiz yap
7. Monte Carlo simülasyonu çalıştır
8. Rapor oluştur
```

---

## 9. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `MIN_TRADES` | 100 | Minimum trade sayısı (anlamlılık için) |
| `TRAIN_PERIOD` | 60 gün | Walk-forward eğitim penceresi |
| `TEST_PERIOD` | 20 gün | Walk-forward test penceresi |
| `MAKER_FEE` | 0.02% | Maker fee |
| `TAKER_FEE` | 0.04% | Taker fee |
| `SLIPPAGE_FACTOR` | 0.02 | Slippage çarpanı |
| `MONTE_CARLO_RUNS` | 10000 | Monte Carlo simülasyon sayısı |
| `CONFIDENCE_LEVEL` | 95% | İstatistiksel güven seviyesi |

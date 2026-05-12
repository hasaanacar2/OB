# Session Engine — Matematiksel Model

## 1. Amaç

Market davranışını session (seans) bazlı analiz etmek. Kripto 7/24 açık olsa da, geleneksel finans merkezlerinin aktif saatlerinde farklı volatilite ve likidite profilleri oluşur.

---

## 2. Session Tanımları

### 2.1 Zaman Aralıkları (UTC)

| Session | Başlangıç (UTC) | Bitiş (UTC) | Karakteristik |
|---|---|---|---|
| **Asia** | 00:00 | 08:00 | Düşük volatilite, range oluşumu |
| **London** | 08:00 | 13:00 | İlk büyük hareket, sweep potansiyeli |
| **New York** | 13:00 | 21:00 | En yüksek hacim, trend hareketler |
| **Off-Hours** | 21:00 | 00:00 | Düşük likidite |

### 2.2 Session Overlap

```
London-NY Overlap: 13:00 - 17:00 UTC
```

Bu 4 saatlik pencere en yüksek likidite ve volatiliteye sahiptir.

---

## 3. Session Range

### 3.1 Asia Range Hesaplama

```
Asia_High = max(H_t)    t ∈ [00:00, 08:00] UTC
Asia_Low  = min(L_t)    t ∈ [00:00, 08:00] UTC
Asia_Range = Asia_High - Asia_Low
```

### 3.2 Range Normalize

```
Normalized_Range = Asia_Range / ATR
```

- `Normalized_Range < 0.5` → Çok dar range → büyük hareket potansiyeli
- `Normalized_Range > 1.5` → Geniş range → zaten hareket olmuş

### 3.3 Herhangi Bir Session'ın Range'i

```
Session_High(S) = max(H_t)    t ∈ S
Session_Low(S)  = min(L_t)    t ∈ S
Session_Range(S) = Session_High(S) - Session_Low(S)
```

---

## 4. Session Sweep

### 4.1 Asia Range Sweep

**Bullish sweep (aşağı sweep → yukarı hareket):**

```
L_t < Asia_Low ∧ C_t > Asia_Low
```

burada `t ∈ London session veya NY session`

**Bearish sweep (yukarı sweep → aşağı hareket):**

```
H_t > Asia_High ∧ C_t < Asia_High
```

### 4.2 Sweep Kalitesi

```
Sweep_Quality = (|sweep_wick| / ATR) × (Volume_t / Avg_Volume_session)
```

- `Sweep_Quality ≥ 1.0` → güçlü sweep
- `Sweep_Quality < 0.5` → zayıf

### 4.3 Previous Session Sweep

London range, NY açılışında sweep edilir:

```
London_High = max(H_t)    t ∈ London
London_Low  = min(L_t)    t ∈ London

NY_Sweep_High ⟺ H_t > London_High ∧ C_t < London_High    (t ∈ NY)
NY_Sweep_Low  ⟺ L_t < London_Low ∧ C_t > London_Low       (t ∈ NY)
```

---

## 5. Session Bazlı Volatilite Profili

### 5.1 Session ATR

```
Session_ATR(S) = mean(TR_t)    t ∈ S, rolling over past_n_days
```

### 5.2 Volatilite Oranı

```
Vol_Ratio(S) = Session_ATR(S) / Daily_ATR
```

Tipik değerler:

| Session | Vol_Ratio |
|---|---|
| Asia | 0.25 - 0.35 |
| London | 0.30 - 0.40 |
| NY | 0.35 - 0.50 |
| Overlap | 0.40 - 0.55 |

### 5.3 Session Volume Profili

```
Session_Vol_Ratio(S) = Avg_Volume(S) / Daily_Avg_Volume
```

---

## 6. Session Bazlı Strateji Seçimi

### 6.1 Karar Matrisi

| Session | Birincil Strateji | İkincil Strateji |
|---|---|---|
| **Asia** | Range tespiti, seviye belirleme | Mean Reversion |
| **London Open** | Asia sweep + OB retest | Breakout / BOS |
| **London-NY Overlap** | Trend continuation | Volume breakout |
| **NY** | Trend devam / Reversal | Liquidity sweep |
| **Off-Hours** | İşlem yapma (düşük likidite) | Sadece izle |

### 6.2 Session Filter

```
Trade_Allowed ⟺ current_session ∈ ALLOWED_SESSIONS
```

Önerilen:

```
ALLOWED_SESSIONS = {London, London_NY_Overlap, NY}
```

Asia ve Off-Hours'da yeni trade açılmamalı (düşük likidite).

---

## 7. Session Open / Close Seviyeleri

### 7.1 Session Open

```
Session_Open(S) = Open_price_of_first_candle_in_S
```

### 7.2 Günlük Open

```
Daily_Open = Open_price_at_00:00_UTC
```

### 7.3 Session Open Retest

```
if price_returns_to_session_open ∧ BOS_confirmed:
    potential_entry_signal()
```

---

## 8. Killzone Tanımları

ICT (Inner Circle Trader) terminolojisinde:

### 8.1 London Killzone

```
LKZ = 07:00 - 10:00 UTC
```

Bu pencerede Asia likidityesi sweep edilir.

### 8.2 New York Killzone

```
NYKZ = 12:00 - 15:00 UTC
```

Bu pencerede London likidityesi sweep edilir.

### 8.3 Killzone Ağırlıklandırma

```
if t ∈ Killzone:
    signal_weight × 1.2    # %20 bonus
else:
    signal_weight × 1.0
```

---

## 9. Time-Based Edge Statistics

### 9.1 Saatlik Win Rate

Her saat için geçmiş performans:

```
WR_hour(h) = wins_at_hour(h) / trades_at_hour(h)
```

### 9.2 Gün Bazlı Performans

```
WR_day(d) = wins_on_day(d) / trades_on_day(d)
```

Pazartesi / Cuma genelde farklı davranır.

### 9.3 Time Heatmap

```
Performance_Matrix[day][hour] = avg_pnl_at(day, hour)
```

Bu matris hangi zaman dilimlerinin en karlı olduğunu gösterir.

---

## 10. Asian Session Sweep Stratejisi (Tam Formül)

### 10.1 Setup Koşulları

```
1. Asia_Range_Normalized < 1.0    # Dar range
2. t ∈ London_Killzone            # London açılışı
3. L_t < Asia_Low ∧ C_t > Asia_Low    # Aşağı sweep
   VEYA
   H_t > Asia_High ∧ C_t < Asia_High   # Yukarı sweep
4. BOS_confirmed after sweep       # Yapısal kırılma
5. OB_identified after BOS         # Order block tespiti
```

### 10.2 Entry

```
Entry = OB_Mid = (OB_High + OB_Low) / 2
```

### 10.3 SL ve TP

```
SL = Sweep_Extreme - ATR × 0.2    # Sweep'in ötesi
TP = Opposite_Session_Extreme       # Asia'nın diğer ucu veya ötesi
```

### 10.4 RR Kontrolü

```
RR = |TP - Entry| / |Entry - SL|
if RR < 2.0: REJECT
```

---

## 11. Algoritmik Akış

```
1. Mevcut session'ı belirle (UTC bazlı)
2. Session range hesapla (Asia, London, etc.)
3. Session volatilite ve volume profili hesapla
4. Killzone kontrolü yap
5. Sweep kontrolü yap:
   a. Asia range sweep?
   b. Previous session sweep?
6. Sweep + BOS varsa → OB retest bekle
7. Session filter uygula (off-hours'da trade yok)
8. Time-based edge statistics ile sinyal ağırlıklandır
```

---

## 12. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `ASIA_START_UTC` | 00:00 | Asia session başlangıcı |
| `ASIA_END_UTC` | 08:00 | Asia session bitişi |
| `LONDON_START_UTC` | 08:00 | London session başlangıcı |
| `NY_START_UTC` | 13:00 | NY session başlangıcı |
| `LKZ_START_UTC` | 07:00 | London Killzone başlangıcı |
| `LKZ_END_UTC` | 10:00 | London Killzone bitişi |
| `NYKZ_START_UTC` | 12:00 | NY Killzone başlangıcı |
| `NYKZ_END_UTC` | 15:00 | NY Killzone bitişi |
| `MAX_ASIA_RANGE_NORM` | 1.0 | Asia range max (ATR normalized) |
| `MIN_SWEEP_WICK_ATR` | 0.5 | Minimum sweep wick |
| `KZ_SIGNAL_BONUS` | 1.2 | Killzone sinyal bonusu |

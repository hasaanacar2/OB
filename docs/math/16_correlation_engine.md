# Correlation Engine — Matematiksel Model

## 1. Amaç

Piyasalar arası ilişkileri (inter-market analysis) ve varlıklar arası korelasyonu (cross-asset) inceleyerek genel pazar riskini ve yönünü tahmin etmek. Sadece teknik analiz yetmez, makro/global para akışları da trendi etkiler.

---

## 2. Pearson Korelasyon Katsayısı (ρ)

İki varlığın (örn. BTC ve NASDAQ) belirli bir periyot üzerindeki getiri (return) serileri arasındaki doğrusal ilişkiyi ölçer.

### 2.1 Getiri (Return) Hesaplaması

```
R_t = (Close_t - Close_{t-1}) / Close_{t-1}
```

### 2.2 Korelasyon Katsayısı

```
ρ(X, Y) = Cov(X, Y) / (σ_X × σ_Y)
```

burada:
- `Cov(X, Y) = Σ(X_i - μ_X)(Y_i - μ_Y) / (n - 1)`
- `μ` = Ortalamalar
- `σ` = Standart sapmalar
- `n` = Rolling window (genelde 30, 90 gün)

**Yorumlama:**
- `ρ ≈ 1`: Mükemmel pozitif korelasyon (Birlikte hareket)
- `ρ ≈ -1`: Mükemmel negatif korelasyon (Ters hareket)
- `ρ ≈ 0`: İlişki yok

---

## 3. Temel Makro / Kripto Korelasyon Çiftleri

Aşağıdaki verilerin BTC ile korelasyonu sürekli izlenmelidir:

1. **BTC vs SPX/NDX (Hisse Senetleri):** Risk-on ortamını gösterir. Pozitif korelasyon yüksektir.
2. **BTC vs DXY (Dolar Endeksi):** Geleneksel olarak ters koreledir. Dolar güçlenirse BTC düşer.
3. **BTC vs GOLD:** Safe haven (güvenli liman) korelasyonu. Bazen pozitif, bazen negatiftir.
4. **ETH/BTC vs Altcoinler:** ETH'nin BTC'ye karşı güçlenmesi, altcoin sezonunu (risk-on içindeki risk-on) tetikler.
5. **BTC Dominance (BTC.D):** Kripto pazarındaki paranın BTC'ye mi altcoinlere mi aktığını gösterir.

---

## 4. Korelasyon Kopması (Decoupling) Analizi

Piyasalar arası ilişkilerin koptuğu anlar (decoupling), genellikle büyük trend değişimlerinin veya spesifik haberlerin (ETF onayı vb.) habercisidir.

### 4.1 Divergence (Uyumsuzluk) Tespiti

```
if ρ(BTC, SPX) > 0.7 for last 30 days:
    if SPX makes Higher High ∧ BTC makes Lower High:
        Macro_Bearish_Divergence = True
        
    if SPX makes Lower Low ∧ BTC makes Higher Low:
        Macro_Bullish_Divergence = True
```

### 4.2 Beta (β) Hassasiyet Analizi

BTC'nin Nasdaq'a göre volatilitesini ölçer.

```
β(BTC, NDX) = Cov(R_BTC, R_NDX) / Var(R_NDX)
```

Eğer `β > 1` ise, BTC Nasdaq'ın kaldıraçlı hali gibi davranıyor demektir.

---

## 5. Risk On / Risk Off (RORO) Rejimi

Global piyasaların risk alma iştahını ölçen kompozit bir endekstir.

### 5.1 RORO Skoru

```
Risk_On_Assets = [SPX, NDX, BTC]
Risk_Off_Assets = [DXY, GOLD, TLT (Tahvil)]

RORO_Score = w₁ × Momentum(SPX) - w₂ × Momentum(DXY) + w₃ × Momentum(BTC) - w₄ × Momentum(GOLD)
```
Momentum ROC (Rate of Change) ile hesaplanır.

- `RORO_Score > 0` → Risk-On Rejimi (Kripto için Bullish)
- `RORO_Score < 0` → Risk-Off Rejimi (Kripto için Bearish)

---

## 6. Altcoin - BTC İlişkisi (Korelasyon Filtresi)

Altcoin trade ederken BTC'nin yönü ve dominansı hayati önem taşır.

### 6.1 Altcoin Risk Filtresi

Altcoin'ler BTC'ye göbekten bağlıdır. Eğer BTC direnç altındaysa altcoin longlamak tehlikelidir.

```
if Altcoin_Signal == LONG ∧ BTC_Trend == BEARISH:
    REJECT_SIGNAL()  # Veya position size düşür
    
if Altcoin_Signal == LONG ∧ BTC_Trend == BULLISH ∧ BTC.D_Trend == DOWN:
    MULTIPLY_POSITION_SIZE(1.5)  # İdeal altcoin koşulu
```

---

## 7. Algoritmik Akış

```
1. BTC, SPX, DXY, ETH/BTC, BTC.D verilerini çek (Günlük veya 4H).
2. Rolling korelasyonları (Pearson ρ) hesapla.
3. RORO_Score hesaplayarak global risk rejimini belirle.
4. Divergence kontrolü yap (SPX yeni zirve yapıp BTC yapamıyorsa uyar).
5. Kripto-spesifik engine'lere (Trend, Reversion) Macro-Bias sinyalini gönder.
6. Altcoin trade'leri için BTC trend filtresini uygula.
```

---

## 8. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `CORR_WINDOW` | 30 | Pearson korelasyon penceresi (gün) |
| `HIGH_CORR_THRESH`| 0.7 | Yüksek korelasyon eşiği |
| `MACRO_MOMENTUM_N`| 14 | RORO score için momentum periyodu |
| `BETA_WINDOW` | 60 | Beta hesaplama penceresi |

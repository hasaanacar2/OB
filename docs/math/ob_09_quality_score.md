# Order Block (OB) Kalite Skoru — Matematiksel Model

## 1. Amaç

Farklı stratejilerden (Retest, Sweep, Breaker vb.) gelen her OB sinyalini objektif, deterministik ve matematiksel bir filtreye sokarak en yüksek kazanma olasılığına sahip olanları (A+ Setups) seçmek. Düşük skorlu sinyalleri işleme sokmayarak risk yönetimini güçlendirmek.

---

## 2. Puanlama Kriterleri (Scoring Matrix)

Bir OB sinyalinin kalitesi 0 ile 100 arasında bir `Total_Score` ile belirlenir. Toplam 6 ana metrik üzerinden değerlendirme yapılır.

### 2.1 BOS / CHOCH Gücü (Maksimum 20 Puan)

Order Block'u yaratan yapısal kırılımın (Break of Structure) kalitesi. Sadece iğne ile mi kırıldı, yoksa gövdeyle ve güçlü mü kırıldı?

```
Break_Distance = |Close_after_break - Broken_Level|
BOS_Strength = Break_Distance / ATR(14)

if BOS_Strength > 1.0:
    Score_BOS = 20
elif BOS_Strength > 0.5:
    Score_BOS = 10
else:
    Score_BOS = 0  # Sadece zayıf bir iğne kırılımı
```

### 2.2 Impulse Kalitesi (Maksimum 20 Puan)

BOS'u yaratan dalganın hızı ve büyüklüğü. Fiyat OB'den ne kadar hızlı uzaklaştı?

```
Impulse_Ratio = Impulse_Move_Size / ATR(14)

if Impulse_Ratio >= 2.0:
    Score_Impulse = 20
elif Impulse_Ratio >= 1.5:
    Score_Impulse = 15
elif Impulse_Ratio >= 1.0:
    Score_Impulse = 5
else:
    Score_Impulse = 0
```

### 2.3 Liquidity Sweep (Maksimum 20 Puan)

OB oluşmadan hemen önce fiyat bir likidite havuzunu (Equal Lows/Highs, Session H/L) temizledi mi?

```
if Liquidity_Sweep_Detected == True:
    if Sweep_Wick > ATR(14) * 0.5:
        Score_Sweep = 20   # Çok güçlü likidite alımı
    else:
        Score_Sweep = 10   # Zayıf likidite alımı
else:
    Score_Sweep = 0
```

### 2.4 FVG (Fair Value Gap) Overlap (Maksimum 15 Puan)

OB'nin hemen önünde (veya bitişiğinde) temiz bir boşluk var mı? Fiyatın oraya dönmek için mıknatıs etkisi (Confluence) var mı?

```
if FVG_Detected_Near_OB == True:
    if FVG_Size > ATR(14) * 0.5:
        Score_FVG = 15
    else:
        Score_FVG = 5
else:
    Score_FVG = 0
```

### 2.5 Higher Timeframe (HTF) Uyumu (Maksimum 15 Puan)

Açılan işlem, bir üst zaman dilimindeki (HTF) genel trendle (Bias) aynı yönde mi?

```
LTF_Signal_Direction = +1 (Long) veya -1 (Short)

HTF_Trend_Score = w1 * EMA(50_HTF) + w2 * ADX(HTF) ... ([-1, 1] arası)

Alignment = LTF_Signal_Direction * Sign(HTF_Trend_Score)

if Alignment == 1:
    Score_HTF = 15  # Trend yönünde
elif Alignment == -1:
    Score_HTF = 0   # Trende karşı (Counter-trend)
else:
    Score_HTF = 5   # HTF yatay (Ranging)
```

### 2.6 Hacim (Volume) Onayı (Maksimum 10 Puan)

OB'den çıkış anında (Impulse sırasında) hacim ortalamanın üzerinde miydi?

```
Volume_Ratio = Impulse_Volume / SMA(Volume, 20)

if Volume_Ratio > 2.0:
    Score_Volume = 10
elif Volume_Ratio > 1.2:
    Score_Volume = 5
else:
    Score_Volume = 0
```

---

## 3. Toplam Skor ve Karar (Decision Matrix)

```
Total_Score = Score_BOS + Score_Impulse + Score_Sweep + Score_FVG + Score_HTF + Score_Volume
```

Skor üzerinden risk yönetimi kararı:

| Total Score | Kalite (Setup Grade) | Karar / Aksiyon | Pozisyon Büyüklüğü |
|---|---|---|---|
| **80 - 100** | **A+ Setup** | GİRİŞ (Güçlü Onay) | Tam Risk (1.0x) |
| **60 - 79** | **B Setup** | GİRİŞ (Standart Onay) | Yarım Risk (0.5x) |
| **40 - 59** | **C Setup** | İZLE (Onay Bekle) | Çeyrek Risk veya Girme |
| **0 - 39** | **D Setup** | ÇÖP SİNYAL | KESİNLİKLE GİRME |

---

## 4. Ek Risk Çarpanları (Penalties & Multipliers)

Eğer temel skor yüksek olsa bile, dışsal bazı faktörler (Regime veya News) skoru düşürebilir (Penalize edilebilir).

```
Adjusted_Score = Total_Score

if Regime_Detection == "NEWS_DRIVEN":
    Adjusted_Score *= 0.0  # Haber anında tüm sinyalleri iptal et

if Regime_Detection == "HIGH_VOL":
    Adjusted_Score *= 0.8  # Yüksek volatilitede güveni azalt

if Order_Book_Imbalance_Against_Trade > 0.4:
    Adjusted_Score -= 15   # Emir defteri işleme çok tersse puan kır
```

---

## 5. Algoritmik Akış

```
1. Herhangi bir OB stratejisi (Retest, Sweep vb.) bir sinyal ürettiğinde parametreleri (Impulse, BOS, OB) Skor Engine'e gönderir.
2. Skor Engine; FVG, Volume, HTF Trend ve Sweep durumlarını geriye dönük hesaplar.
3. 6 ana metrik üzerinden puanları toplar (Total_Score).
4. Dış etkenlere (Rejim, Haber, Order Book) göre Penalty (ceza) çarpanlarını uygular (Adjusted_Score).
5. Adjusted_Score < 60 ise, sinyal Risk Engine'e gönderilmeden doğrudan iptal (Reject) edilir.
6. Adjusted_Score >= 60 ise, Skor ve Kategori (A+ veya B) Risk Engine'e iletilir ve position size buna göre belirlenir.
```

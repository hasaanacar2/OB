# FVG + OB (Fair Value Gap ve Order Block) Stratejisi — Matematiksel Model

## 1. Amaç

Tek başına bir Order Block (OB) her zaman yeterince güçlü olmayabilir. Ancak bir OB'nin hemen önünde bir FVG (Fair Value Gap / Imbalance) varsa, fiyatın o FVG'yi doldurmak için geri dönmesi ve ardından OB'den sekmesi çok yüksek olasılıklıdır. Strateji, FVG ve OB kesişimini (Confluence) trade eder.

---

## 2. Fair Value Gap (FVG) Tespiti

FVG, üç mumluk (3-candle) bir formasyonda ortadaki mumun gövdesinde oluşan ve diğer iki mumun iğneleri (wicks) tarafından doldurulmamış olan "boşluk"tur.

### 2.1 Bullish FVG (Yükseliş Boşluğu)

```
Condition ⟺ Low(Candle_3) > High(Candle_1)
```
**Boşluk Sınırları:**
```
FVG_Top = Low(Candle_3)
FVG_Bottom = High(Candle_1)
```

### 2.2 Bearish FVG (Düşüş Boşluğu)

```
Condition ⟺ High(Candle_3) < Low(Candle_1)
```
**Boşluk Sınırları:**
```
FVG_Top = Low(Candle_1)
FVG_Bottom = High(Candle_3)
```

### 2.3 FVG Kalitesi (Genişliği)

Çok dar boşluklar (noise) filtreye takılmalıdır.

```
FVG_Size = FVG_Top - FVG_Bottom
Valid_FVG ⟺ FVG_Size > ATR(14) × 0.2
```

---

## 3. Confluence (Kesişim) Tespiti

Stratejinin çekirdek mantığı: FVG ve Sponsor OB'nin aynı fiyat bölgesinde üst üste binmesi (overlap) veya yan yana (bitişik) olmasıdır.

### 3.1 OB ve FVG İlişkisi

Diyelim ki bir **Bullish OB** tespit edildi ve hemen ardından yükseliş yönlü bir **Bullish FVG** oluştu.

```
Confluence_Condition ⟺ 
    FVG_Bottom ≤ OB_High + (ATR × 0.1)  # FVG'nin alt sınırı, OB'nin üst sınırına çok yakın veya iç içe
```

### 3.2 Imbalance Doldurulması (Fill)

Fiyatın bu yüksek kaliteli bölgeye geri çekilmesi beklenir.

```
Partial_Fill ⟺ Low_t < FVG_Top ∧ Low_t > FVG_Bottom
Full_Fill    ⟺ Low_t ≤ FVG_Bottom
```

En ideal senaryoda fiyat FVG'yi doldurur (Full Fill), OB'ye iğne atar ve fırlar.

---

## 4. Entry, Stop-Loss ve Take-Profit

### 4.1 Entry (Giriş)

İki farklı giriş stratejisi uygulanabilir:

**Agresif Giriş:** FVG'nin başladığı yer (FVG_Top)
```
Entry_Aggressive = FVG_Top
```

**Optimal Giriş:** FVG'nin yarısı (%50) veya doğrudan OB'nin başlangıcı.
```
Entry_Optimal = (FVG_Top + FVG_Bottom) / 2
# Veya
Entry_Optimal = OB_High
```

*Not: Sistem deterministik yapısında "Entry_Optimal = FVG %50" varsayılan olarak kullanılmalıdır.*

### 4.2 Stop-Loss

Stop noktası her zaman FVG'yi yaratan OB'nin altıdır. Çünkü fiyat OB'nin de altına sarkarsa tüm analiz çöp olur.

```
Bullish_SL = OB_Low - ATR(14) × 0.1
Bearish_SL = OB_High + ATR(14) × 0.1
```

### 4.3 Take-Profit

FVG dolduktan sonraki ilk hedef, o boşluğu yaratan impulse hareketinin zirvesidir (Liquidity target).

```
Bullish_TP = Impulse_High
Bearish_TP = Impulse_Low
```

---

## 5. Algoritmik Akış

```
1. 3 mumluk pencerelerde (rolling window) FVG (Boşluk) tara.
2. FVG bulunursa, FVG_Size > ATR * 0.2 kontrolü yap.
3. FVG'yi başlatan hareketin kökündeki mumu (OB) bul.
4. FVG bölgesi ile OB bölgesi birbirine bitişik mi (veya iç içe mi) kontrol et (Confluence).
5. Eğer bitişikse, bu bölgeyi "High Probability Zone" olarak işaretle.
6. Fiyat geri çekilip FVG alanına girdiğinde (Fill):
   a. FVG'nin %50 seviyesinde Limit Emir aç.
   b. SL'yi OB'nin dışına koy.
7. Fiyat OB'nin altına kırarsa bölgeyi invalid et.
```

---

## 6. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `MIN_FVG_SIZE` | 0.2 ATR | Dikkate alınacak minimum FVG genişliği |
| `MAX_FVG_AGE` | 40 mum | FVG'nin açık kalabileceği maksimum süre |
| `ENTRY_LEVEL` | FVG_Mid | FVG içinde işleme girilecek seviye |
| `SL_BUFFER` | 0.1 ATR | OB dışına konulacak stop toleransı |

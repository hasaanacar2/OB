# Asian Session Sweep Stratejisi — Matematiksel Model

## 1. Amaç

Kripto (ve Forex) piyasalarında Asya seansı genellikle düşük hacimli ve dar bir aralıkta (range) geçer. Londra veya New York açılışlarında algoritmalar ve büyük oyuncular, bu Asya range'inin altında veya üstünde biriken likiditeyi (stopları) avlamak için sahte bir hareket (sweep) yapar. Bu strateji, bu saat bazlı likidite avını ve ardından gelen gerçek hareketi trade eder.

---

## 2. Zaman Pencereleri (Time Windows)

Bu strateji zaman (Session) bağımlıdır. İşlemler yalnızca belirli saatlerde aranır (UTC bazlı).

### 2.1 Asya Seansı (Range Belirleme)

```
Asia_Start = 00:00 UTC
Asia_End   = 08:00 UTC
```

Bu saatler arasında piyasanın yaptığı en yüksek ve en düşük değerler kaydedilir.

```
Asia_High = Max(High_t) ∀ t ∈ [Asia_Start, Asia_End]
Asia_Low  = Min(Low_t)  ∀ t ∈ [Asia_Start, Asia_End]
```

### 2.2 Killzone (Avlanma Saatleri)

Sweep hareketinin gerçekleşmesi beklenen ve yüksek hacmin piyasaya girdiği anlar:

**London Killzone (LKZ):** 07:00 - 10:00 UTC
**New York Killzone (NYKZ):** 12:00 - 15:00 UTC

```
Valid_Time_Window ⟺ Current_Time ∈ (LKZ ∪ NYKZ)
```

---

## 3. Range Genişliği (Filtre)

Eğer Asya seansı çok geniş (trend halinde) geçtiyse, üstünde ve altında yeterince likidite birikmemiş demektir. Strateji sadece "dar" range'lerde çalışır.

```
Asia_Range_Size = Asia_High - Asia_Low
Normalized_Range = Asia_Range_Size / ATR(14)

Valid_Range ⟺ Normalized_Range ≤ 1.2
```

*(Eğer Asya range'i ATR'nin 1.2 katından büyükse o gün bu strateji uygulanmaz).*

---

## 4. Sweep ve Geri Dönüş (Reversal) Tespiti

Fiyatın Killzone saatleri içinde Asya sınırlarından birini ihlal edip hızla geri dönmesi.

### 4.1 Bullish Senaryo (Asya Low Sweep)

Büyük oyuncular fiyatı Asya Low'un altına itip stopları patlatır, sonra alım yapar.

```
Sweep_Condition ⟺ 
    Low_t < Asia_Low ∧     # İğne Asya Low'u geçti
    Close_t > Asia_Low ∧   # Mum Asya sınırının içinde kapandı
    Time_t ∈ Killzone
```

**Sweep Kalitesi:**
```
Wick_Size = Asia_Low - Low_t
Valid_Wick ⟺ Wick_Size > ATR(14) × 0.2
```

### 4.2 Bearish Senaryo (Asya High Sweep)

```
Sweep_Condition ⟺ 
    High_t > Asia_High ∧ 
    Close_t < Asia_High ∧ 
    Time_t ∈ Killzone
```

---

## 5. Onay: CHOCH ve OB

Sadece sweep olması yetmez, piyasanın ters yöne döneceğini teyit eden bir Market Structure kırılımı (CHOCH) olmalıdır.

**Bullish Onay:**
```
CHOCH ⟺ Close > Swing_High_Before_Sweep
Sponsor_OB = CHOCH'u başlatan son düşüş mumu (Bearish Candle)
```

---

## 6. Entry, Stop-Loss ve Take-Profit

### 6.1 Entry (Giriş)

Sweep ve CHOCH oluştuktan sonra fiyat Sponsor OB'ye döndüğünde limit emir.

```
Entry_Long = Sponsor_OB_High
```

### 6.2 Stop-Loss

Stop noktası her zaman manipülasyon (sweep) iğnesinin ucudur.

```
SL_Long = Sweep_Low - ATR(14) × 0.1
```

### 6.3 Take-Profit

Bu stratejinin en büyük avantajı hedefin çok net olmasıdır. Eğer Asya Low sweep edildiyse (stoplar alındıysa), fiyatın yeni hedefi Asya High'dır (üstteki stoplar).

```
TP_1 = Asia_High (Bullish işlem için)
TP_2 = Asia_High + (Asia_High - Asia_Low) × 0.5  # Range'in %50'si kadar extension
```

---

## 7. Algoritmik Akış

```
1. Saat 08:00 UTC olduğunda `Asia_High` ve `Asia_Low` değerlerini hesapla ve kaydet.
2. `Asia_Range_Size` değerinin ATR'ye göre yeterince dar olup olmadığını kontrol et. Genişse o günü pas geç.
3. Saat 07:00-10:00 (London) veya 12:00-15:00 (NY) aralığında fiyatın bu sınırları kırmasını izle.
4. Fiyat bir sınırı (örn. Low) aşağı kırıp yukarı kapanırsa (Sweep), LTF'de yukarı yönlü CHOCH ara.
5. CHOCH oluşursa, hareketi başlatan OB'yi işaretle.
6. Fiyat OB'ye geri çekildiğinde Long Limit Emir gir (Hedef: Asia_High, SL: İğne ucu).
7. Eğer Asya saatleri bitmeden (00:00 - 08:00 arası) fiyat kendi sınırlarını kırarsa strateji geçersizdir.
```

---

## 8. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `MAX_ASIA_RANGE` | 1.2 ATR | Kabul edilebilir maksimum Asya aralığı |
| `MIN_SWEEP_WICK` | 0.2 ATR | Geçerli sayılacak minimum sweep iğne boyu |
| `KILLZONES` | LKZ, NYKZ | İşlem aranacak saat aralıkları |
| `TP_TARGET` | Opposite_Asia | Kar al seviyesi (Karşı Asya sınırı) |

# Breaker Block Stratejisi — Matematiksel Model

## 1. Amaç

Başarısız olan (kırılan) bir Order Block'un, fiyat için destekten dirence (veya dirençten desteğe) dönüşmesini trade etmek. Fiyatı tutamayan bir likidite bölgesi, karşı yöndeki traderlar için güçlü bir giriş noktasına dönüşür (S/R flip mantığı).

---

## 2. Failed OB (Başarısız Order Block) Tespiti

Bir OB'nin kırılması (fail olması), fiyatın OB bölgesinin tamamen zıt tarafında kapanış yapmasıyla onaylanır.

### 2.1 Bullish OB'nin Fail Olması (Bearish Breaker'a Dönüşümü)

Eskiden fiyatı yukarı itmesi beklenen bir Bullish OB vardı:
```
OB_High = High(OB_Bullish_Candle)
OB_Low = Low(OB_Bullish_Candle)
```

Fiyat bu OB'yi aşağı yönlü güçlü bir şekilde kırar:
```
Failure_Condition ⟺ 
    Close_t < OB_Low ∧ 
    Down_Impulse_Size > ATR(14)
```
Bu durumda eski `Bullish OB` artık bir **Bearish Breaker Block** olmuştur.

### 2.2 Bearish OB'nin Fail Olması (Bullish Breaker'a Dönüşümü)

```
Failure_Condition ⟺ 
    Close_t > OB_High ∧ 
    Up_Impulse_Size > ATR(14)
```
Eski `Bearish OB` artık bir **Bullish Breaker Block** olmuştur.

---

## 3. Retest (Geri Dönüş) Şartları

Kırılan OB'nin bir Breaker olarak çalışması için fiyatın bu bölgeye geri dönmesi (retest) gerekir. Ancak bu dönüşün çok uzun sürmemesi veya aralarda başka büyük yapıların oluşmaması önemlidir.

### 3.1 Geri Dönüş Zamanı (Time Limit)

```
Time_Since_Break = Current_Time - Break_Time
Valid_Retest ⟺ Time_Since_Break ≤ MAX_BREAKER_AGE (örn. 50 mum)
```

### 3.2 Fiyatın Breaker Alanına Girmesi

Bearish Breaker için (Short arıyoruz):
```
Retest_Price ∈ [OB_Low, OB_High]
```
Eğer fiyat `OB_High`'ın da üstüne çıkarsa (yukarı kırarsa), Breaker geçersiz (invalid) olur.

---

## 4. Onay (Confirmation) ve Momentum

Breaker'lar genellikle trend değişimlerinde veya sert düzeltmelerde (correction) çalışır. Bu nedenle momentumun yönü destekleyici olmalıdır.

**Short (Bearish Breaker) için Onay:**
```
RSI(14) < 50  (Trend artık aşağı yönlü)
MACD_Histogram < 0
```

Veya LTF'de (Düşük zaman dilimi) Breaker bölgesi içinde bir dönüş emaresi (LTF CHOCH):
```
LTF_Close < LTF_Previous_Swing_Low
```

---

## 5. Entry, Stop-Loss ve Take-Profit

### 5.1 Entry (Giriş)

Fiyat Breaker bölgesine (eski OB sınırları) girdiğinde limit emir.

```
Entry_Short = OB_Low  (Bearish Breaker için eski OB'nin alt sınırı artık dirençtir)
Entry_Long  = OB_High (Bullish Breaker için eski OB'nin üst sınırı artık destektir)
```

### 5.2 Stop-Loss

Stop, Breaker bloğunun tamamen geçersiz olduğu (fiyatın eski yerine döndüğü) noktadır.

```
SL_Short = OB_High + ATR(14) × 0.2
SL_Long  = OB_Low - ATR(14) × 0.2
```

### 5.3 Take-Profit

İlk hedef yeni oluşan trendin yaratmış olduğu son dip/tepe noktasıdır.

```
TP_1_Short = Recent_Swing_Low_Formed_After_Break
TP_1_Long  = Recent_Swing_High_Formed_After_Break
```

---

## 6. Algoritmik Akış

```
1. Grafikteki tüm geçerli OB'leri (mitigated olmayan) izle.
2. Bir OB'nin ters yönde hacimli/sert bir mumla (Close) kırılıp kırılmadığını kontrol et.
3. Kırılma (Failure) gerçekleştiyse, bu OB'yi "Breaker Block" listesine al.
4. Fiyat bu Breaker bölgesine geri döndüğünde (Retest):
   a. Momentum/Trend filtresini kontrol et.
   b. Bölge içinde Limit İşlem aç (Kırılma yönüne doğru).
5. Fiyat Breaker bloğunun dışına çıkarsa (Invalidation) işlemi stopla ve Breaker'ı listeden sil.
```

---

## 7. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `BREAK_IMPULSE` | 1.0 ATR | OB'yi kıran mumun minimum büyüklüğü |
| `MAX_BREAKER_AGE` | 50 mum | Kırılımdan sonra retest için beklenecek max süre |
| `SL_BUFFER` | 0.2 ATR | Breaker dışına konan stop toleransı |

# OB Retest Stratejisi — Matematiksel Model

## 1. Amaç

Güçlü bir kırılım (Break of Structure - BOS) sonrasında, fiyatın kırılımı başlatan orijinal Order Block (OB) bölgesine dönmesiyle birlikte trend yönünde işleme girmek.

---

## 2. Order Block (OB) Tespiti

### 2.1 Güçlü (Impulsive) Hareket Tanımı

Order block olabilmesi için fiyatın bir bölgeden çok hızlı ve hacimli ayrılması gerekir.

```
Impulse_Size = |Close_{impulse_end} - Open_{impulse_start}|
ATR_Condition ⟺ Impulse_Size ≥ ATR(14) × 1.5
```

### 2.2 OB Mumunun Seçimi

**Bullish OB:**
Impulsive yükseliş hareketinden hemen önceki son düşüş (bearish) mumudur.
```
OB_Bullish_Candle ⟺ (Close < Open) ∧ (Candle_Index = Impulse_Start_Index - 1)
```

**Bearish OB:**
Impulsive düşüş hareketinden hemen önceki son yükseliş (bullish) mumudur.
```
OB_Bearish_Candle ⟺ (Close > Open) ∧ (Candle_Index = Impulse_Start_Index - 1)
```

### 2.3 OB Sınırlarının Çizilmesi

En deterministik (ve kural bazlı) yöntem tüm mumu almaktır:

```
OB_High = High(OB_Candle)
OB_Low  = Low(OB_Candle)
OB_Mid  = (OB_High + OB_Low) / 2
```

---

## 3. Setup Şartları (Retest)

Stratejinin aktif olması için sıralı olaylar bütünü (Sequence of Events) gerçekleşmelidir:

1. **BOS Oluşumu:** `Close_t > Previous_Swing_High` (Bullish)
2. **Impulse Şartı:** Kırılım `ATR × 1.5` büyüklüğünde bir hareketle gelmeli.
3. **OB Yaşı (Zaman Kısıtlaması):** OB oluştuktan sonra çok uzun süre geçerse gücünü yitirir.
   ```
   Age_of_OB = Current_Time - OB_Creation_Time
   Valid_OB ⟺ Age_of_OB ≤ MAX_OB_AGE (örn. 80 mum)
   ```
4. **Mitigation (Temas) Kontrolü:** Fiyat henüz bu OB bölgesine dönmemiş olmalıdır (Unmitigated).
   ```
   Unmitigated ⟺ Min(Low_Series_Since_OB) > OB_High  (Bullish için)
   ```

---

## 4. Entry, Stop-Loss ve Take-Profit (Emir Yönetimi)

### 4.1 Entry (Giriş)

Fiyat geri çekilerek (pullback) OB bölgesine dokunduğunda (veya içine girdiğinde).

```
Entry_Price = OB_Mid  (Veya OB_High - spread_buffer)
```

**Tetikleyici (Trigger):** Fiyat OB'ye girdiğinde doğrudan limit emir alınabilir veya LTF'de (düşük zaman dilimi) bir dönüş onayı (CHOCH) beklenebilir. (MVP için doğrudan limit emir daha uygundur).

### 4.2 Stop-Loss (Zarar Kes)

Stop seviyesi OB'nin tamamen ihlal edildiği noktadır, volatiliteye (ATR) göre küçük bir tampon (buffer) eklenir.

```
Bullish_SL = OB_Low - ATR(14) × 0.2
Bearish_SL = OB_High + ATR(14) × 0.2
```

### 4.3 Take-Profit (Kar Al)

İlk hedef her zaman yapının karşı tarafındaki likiditedir (Swing High/Low).

```
Bullish_TP_1 = Recent_Swing_High
RR = |TP_1 - Entry| / |Entry - SL|

if RR < 2.0:
    REJECT_TRADE()  # Risk-Reward kurtarmıyorsa işleme girilmez.
```

---

## 5. Algoritmik Akış

```
1. Swing seviyeleri belirle ve BOS kontrolü yap.
2. BOS varsa, kırılımın ATR x 1.5'ten büyük olup olmadığını (Impulse) kontrol et.
3. Impulse onaylandıysa, kırılımdan önceki son zıt mumu (OB) bul ve sınırlarını çiz.
4. OB'nin daha önce test edilip edilmediğini (mitigated) kontrol et.
5. Fiyat OB'ye doğru geri çekilmeye (pullback) başladığında limit emri (Entry=Mid, SL=OB_dışı, TP=Swing_Tepesi) deftere gönder.
6. İşlem gerçekleştiğinde Trade Lifecycle Engine'e devret.
```

---

## 6. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `IMPULSE_ATR_MULT` | 1.5 | Hareketin OB yaratması için gereken hız |
| `MAX_OB_AGE` | 80 mum | OB'nin geçerliliğini koruduğu maksimum süre |
| `OB_ENTRY_PCT` | 0.5 | OB'nin yüzde kaçından işleme girileceği (0.5 = Mid) |
| `SL_BUFFER_ATR` | 0.2 | OB dışına konan stop tamponu |
| `MIN_RR` | 2.0 | Minimum Kabul edilebilir Risk/Reward |

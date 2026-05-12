# Trend Continuation OB Stratejisi — Matematiksel Model

## 1. Amaç

Güçlü bir trend piyasasında (Directional Bias netken), trend yönünde oluşan ara düzeltmelerdeki (pullback) devam formasyonlarını (Continuation Order Blocks) trade etmek. Sweep stratejilerine göre daha düşük win-rate ama çok yüksek Risk/Reward (RR) sunar.

---

## 2. Trend Tespiti ve Yön (Bias) Filtresi

Bu strateji kesinlikle yatay (range) piyasada çalıştırılmamalıdır. Birincil kural güçlü bir trendin varlığıdır.

### 2.1 Çoklu EMA (Moving Average) Filtresi

```
EMA_Fast = EMA(20)
EMA_Mid = EMA(50)
EMA_Slow = EMA(200)

Bullish_Trend ⟺ (Close > EMA_Fast) ∧ (EMA_Fast > EMA_Mid) ∧ (EMA_Mid > EMA_Slow)
Bearish_Trend ⟺ (Close < EMA_Fast) ∧ (EMA_Fast < EMA_Mid) ∧ (EMA_Mid < EMA_Slow)
```

### 2.2 ADX (Trend Gücü) Filtresi

Trendin gerçekten ivmeli olduğunu doğrulamak için:
```
Strong_Trend ⟺ ADX(14) > 25
```

---

## 3. Pullback (Geri Çekilme) ve OB Tespiti

Güçlü bir trendde oluşan her OB işleme dahil edilmez. İdeal bir OB'nin trend düzeltmesinin (pullback) tam bittiği yerde oluşması gerekir.

### 3.1 Pullback Derinliği (Fibonacci)

Trend içindeki son impulse dalgası ölçülür:
```
Impulse_Size = Swing_High - Swing_Low
```

Geri çekilme (Pullback) bu dalganın en az %38.2'sine, en fazla %78.6'sına kadar olmalıdır.
```
PB_Level = (Current_Price - Swing_Low) / Impulse_Size

Valid_Pullback ⟺ 0.382 ≤ PB_Level ≤ 0.786
```
*(Eğer geri çekilme %38.2'den azsa, fiyat yeterince ucuzlamamıştır (Premium bölgededir). Eğer %78.6'yı geçerse trend bozuluyor (reversal) olabilir.)*

### 3.2 Continuation OB Oluşumu

Fiyat ideal pullback bölgesindeyken, trend yönünde yeni bir kırılım (BOS) yapar ve ardında bir OB bırakır.

**Bullish Continuation:**
```
1. Fiyat %50 Fib seviyesine (Discount Zone) iner.
2. Yukarı yönlü bir engulfing (yutan) mum veya sert bir yeşil mum oluşur.
3. Bu mum, önceki küçük tepeyi kırar (Micro BOS).
4. Bu yeşil mumu başlatan kırmızı mum Bullish OB olarak işaretlenir.
```

---

## 4. Entry, Stop-Loss ve Take-Profit

### 4.1 Entry (Giriş)

Fiyat, pullback sonrası yeni bir zirve yapmadan önce bu mini OB'ye geri çekildiğinde girilir.

```
Entry_Long = OB_High (Trend momentumu yüksek olduğu için fiyatın OB'nin derinine inmesi beklenmez, temas yeterlidir).
```

### 4.2 Stop-Loss

Trend işlemleri olduğu için, SL son oluşan Swing Low'un (pullback dip noktasının) altına konur. Sadece OB altına konursa noise (gürültü) ile patlama riski çok yüksektir.

```
SL_Long = Swing_Low - ATR(14) × 0.5
```

### 4.3 Take-Profit

Trendin devam edeceği varsayıldığı için, hedef önceki zirvenin geçilmesidir.

```
TP_1 = Previous_Swing_High  (Bu noktada pozisyonun %50'si kapatılabilir).
TP_2 = Swing_Low + (Previous_Swing_High - Swing_Low) × 1.618  (Fib Extension)
```

---

## 5. İptal Şartı (Invalidation)

Eğer trend bozulursa bekleyen limit emirler iptal edilmelidir.

```
Invalidation_Condition ⟺ 
    Close < EMA(50)  (Bullish için)
    VEYA 
    Close < Swing_Low  (Major CHOCH oluşumu)
```

---

## 6. Algoritmik Akış

```
1. Her mumda EMA(20,50,200) ve ADX değerlerini kontrol ederek Trend Rejiminde olup olmadığımızı belirle.
2. Trend Bullish ise, son `Swing_Low` ve `Swing_High` değerlerini kullanarak Fibonacci bölgelerini çiz.
3. Fiyat `Swing_High`'dan aşağı dönüp (pullback) Discount bölgesine (0.50 - 0.786) girdiğinde izlemeye başla.
4. Fiyat bu bölgede yukarı yönlü bir Micro-BOS yapıp arkasında bir OB bırakırsa, bu OB'yi Continuation_OB olarak kaydet.
5. Fiyat bu OB'ye temas ettiğinde Long Limit emri ile gir.
6. TP olarak önceki zirveyi, SL olarak pullback dibini ayarla.
7. Fiyat EMA(50) altında kapanırsa veya yapıyı kırarsa bekleyen emri iptal et.
```

---

## 7. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `ADX_THRESHOLD` | 25 | Trend filtresi için minimum ADX |
| `MIN_PULLBACK` | 0.382 | Kabul edilebilir minimum geri çekilme (Fib) |
| `MAX_PULLBACK` | 0.786 | Kabul edilebilir maksimum geri çekilme (Fib) |
| `SL_BUFFER` | 0.5 ATR | Swing Low altına konan stop toleransı |

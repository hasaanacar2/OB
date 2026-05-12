# Liquidity Sweep + OB Stratejisi — Matematiksel Model

## 1. Amaç

Piyasa yapıcıların (Market Makers) yatırımcıların stop-loss'larını patlatmak (likidite toplamak) için yaptığı sahte hareketleri (sweep) yakalamak. Strateji, likidite alındıktan hemen sonra gerçek yönünde oluşan kırılımı (BOS/CHOCH) ve ardından gelen OB'yi trade eder. Core stratejiler arasında en yüksek başarı oranına (win rate) sahip olandır.

---

## 2. Likidite Havuzlarının (Liquidity Pools) Tespiti

Piyasanın likidite aradığı yerler belirgin eski tepeler (Swing High) ve diplerdir (Swing Low). En önemlisi Equal Lows/Highs (Eşit Dipler/Tepeler) dir.

### 2.1 Equal Lows / Highs Tespiti

İki dip (veya tepe) birbirine çok yakınsa, altlarında çok fazla stop (likidite) birikmiştir.

```
|Low_i - Low_j| < ATR(14) × 0.05
```
Bu bölge bir `Liquidity_Level` olarak tanımlanır.

---

## 3. Sweep (Süpürme / Likidite Alma) Olayı

Fiyat, bu `Liquidity_Level` altına iner, stopları patlatır ama orada kalamayıp (kapanış yapamayıp) hızla geri dönerse bu bir Sweep'tir.

### 3.1 Sweep Koşulları (Bullish Senaryo)

```
Sweep_Condition ⟺ 
    Low_t < Liquidity_Level_Low ∧   # Fiyat likidite bölgesinin altına iğne attı
    Close_t > Liquidity_Level_Low   # Ama mum bölgenin üstünde kapandı (Reddedildi)
```

### 3.2 Sweep Kalitesi / İğne Boyu (Wick Size)

Sweep'in inandırıcı olması için iğnenin (wick) yeterince uzun olması gerekir.

```
Wick_Size = Liquidity_Level_Low - Low_t
Valid_Wick ⟺ Wick_Size ≥ ATR(14) × 0.5
```

---

## 4. Onay (Confirmation): CHOCH ve OB

Sadece sweep olması yetmez, piyasanın yukarı döneceğine dair iç yapı kırılımı (Lower Timeframe CHOCH) ve ardından Order Block (OB) oluşturması gerekir.

### 4.1 CHOCH (Change of Character) Tespiti

Sweep yapıldıktan sonraki fiyat hareketi, en son oluşan alt tepeyi (Recent Lower High) yukarı kırmalıdır.

```
CHOCH_Confirmed ⟺ Close_new > Recent_Swing_High_Before_Sweep
```

### 4.2 Onaylayıcı OB (Sponsor OB) Oluşumu

CHOCH hareketini başlatan, dipteki (sweep iğnesinin hemen yanındaki) son düşüş mumu Bullish OB olarak işaretlenir.

```
Sponsor_OB = Last_Bearish_Candle_Before(CHOCH_Move)
```

---

## 5. Entry, Stop-Loss ve Take-Profit

### 5.1 Entry (Giriş)

Fiyat, CHOCH (kırılım) yaptıktan sonra aşağı dönüp Sponsor OB'ye dokunduğunda işleme girilir.

```
Entry_Price = Sponsor_OB_High (veya OB_Mid)
```

### 5.2 Stop-Loss

Stop noktası mutlaktır: Sweep iğnesinin en ucu. Çünkü o iğne gerçek bir likidite alımıysa, fiyat bir daha oraya inmemelidir. İnerse analiz yanlıştır.

```
SL = Sweep_Wick_Low - ATR(14) × 0.1
```

### 5.3 Take-Profit

İlk hedef her zaman karşı tarafta bekleyen likidite noktalarıdır (Likiditeden likiditeye trade).

```
TP_1 = Karşı_Taraftaki_Equal_Highs veya Major_Swing_High
TP_2 = Fibonacci Extension 1.618
```

---

## 6. Algoritmik Akış

```
1. Grafik üzerindeki Liquidity Level'ları (Equal highs/lows, Session limits) tespit et ve izle.
2. Fiyat bu seviyelerden birinin altına (örn. Low) indiğinde Sweep durumunu başlat.
3. Mum kapandığında `Close > Liquidity_Level` ve `Wick_Size > ATR*0.5` koşullarını kontrol et.
4. Eğer sweep geçerliyse, fiyatın yukarı yönlü bir CHOCH (kırılım) yapmasını bekle.
5. CHOCH oluşursa, bu kırılımı başlatan son düşüş mumunu (Sponsor OB) belirle.
6. Fiyat Sponsor OB'ye geri çekildiğinde Limit Alım Emri gir (SL: Sweep altı, TP: Yukarıdaki likidite).
```

---

## 7. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `LIQ_EQ_TOLERANCE` | 0.05 ATR | Equal High/Low kabul edilmesi için tolerans |
| `MIN_SWEEP_WICK` | 0.5 ATR | Geçerli bir likidite alımı için min iğne boyu |
| `MAX_CANDLES_TO_CHOCH`| 10 | Sweep'ten sonra kırılımın gerçekleşmesi gereken maksimum süre |
| `SL_BUFFER` | 0.1 ATR | Stop noktasındaki küçük tolerans payı |

# Order Book Engine — Matematiksel Model

## 1. Amaç

Level 2 (Derinlik) ve Level 3 (Emir detayları) verilerini kullanarak gerçek zamanlı likidite ve piyasa mikro-yapısını (microstructure) analiz etmek. Fiyat hareketinden önce birikmiş limit emirleri (bid/ask duvarları) tespit eder. Yüksek gecikme (latency) hassasiyeti vardır.

---

## 2. Order Book (Emir Defteri) Yapısı

Bir emir defteri belirli bir anda `t` fiyat seviyelerindeki hacimleri içerir.

```
Bid_Profile = {(Price_i, Volume_i) | Price_i ≤ Best_Bid}
Ask_Profile = {(Price_i, Volume_i) | Price_i ≥ Best_Ask}
```

### 2.1 Depth (Derinlik) Hesaplaması

Belirli bir fiyat aralığındaki (örneğin fiyatın ±%1'i) toplam likidite:

```
Depth_Bid_1pct = Σ Volume_i   ∀ Price_i ∈ [Best_Bid × 0.99, Best_Bid]
Depth_Ask_1pct = Σ Volume_i   ∀ Price_i ∈ [Best_Ask, Best_Ask × 1.01]
```

---

## 3. Order Book Imbalance (OBI)

Emir defterindeki alış ve satış tarafının ağırlığını ölçer. 

```
OBI = (Depth_Bid_n - Depth_Ask_n) / (Depth_Bid_n + Depth_Ask_n)
```
- `n` genellikle ilk 10 seviye veya fiyatın %0.5'i olarak belirlenir.
- `OBI ∈ [-1, 1]` aralığındadır.
- `OBI > 0.3` → Alıcı tarafı (bid) daha kalın, aşağı yönlü hareket zor.
- `OBI < -0.3` → Satıcı tarafı (ask) daha kalın, yukarı yönlü hareket zor.

**Tahmin Gücü:** Kısa vadeli (tick veya 1s) fiyat yönünü tahmin etmekte kullanılır. Fiyat ağırlığın az olduğu (ince) yöne gitme eğilimindedir.

---

## 4. Emir Duvarları (Order Walls)

Piyasada belirli bir fiyata yığılmış büyük emirleri tespit etmek, destek/direnç noktalarını belirler.

### 4.1 Wall Tespiti

Bir fiyat seviyesindeki hacmin, ortalama hacmin çok üzerinde olması:

```
Mean_Vol_per_Level = Total_Depth / N_Levels

Wall_Condition ⟺ Volume_at_Price > Mean_Vol_per_Level × Wall_Multiplier
```

Önerilen `Wall_Multiplier = 5 ile 10` arası.

### 4.2 Duvara Etki (Impact)

Duvarların olduğu yere fiyat geldiğinde iki şey olur:
1. **Bouncing (Sekme):** Duvar emilemez (absorbe edilemez) ve fiyat döner.
2. **Eating (Yenme):** Market emirleri duvarı parçalar, ardından o yöne sert bir patlama (breakout) gelir.

```
if Price approaches Wall_Ask ∧ Buy_Market_Volume_Flow > Wall_Ask_Size / 2:
    Wall_is_Being_Eaten = True  → Breakout ihtimali yüksek!
```

---

## 5. Spoofing (Sahte Emir) Tespiti

Büyük oyuncular, deftere devasa limit emir koyup (görüntü yaratıp) fiyat o seviyeye gelmeden iptal ederler. Buna spoofing denir.

### 5.1 Spoofing Filtresi (İptal ve CVD Kontrolü)

Bir duvar fiyatın uzağındaysa ve sürekli eklenip/siliniyorsa, büyük ihtimalle sahtedir. Gerçek duvarlar fiyat onlara yaklaştığında orada kalmaya devam eder ve üzerine çarpan Market Emirleri yutar (Absorption).

```
Spoof_Risk = Rate_of_Cancellation(Wall_Price_Level)

if Price_Distance < 0.2% ∧ Wall_Volume_Drops_Suddenly > 50%:
    Detected_Spoofing = True
    Ignore_Wall()
```

**CVD (Cumulative Volume Delta) Onayı:** 
Eğer büyük bir Alım Duvarı (Buy Wall) varsa ve fiyat o duvara çarptığında *Market Satış* hacmi (Aggressive Sells) artmasına rağmen fiyat düşmüyorsa, duvar gerçektir (Absorption). Ama duvar fiyata yaklaşırken CVD negatifleşmeden duvar aniden eriyorsa %100 Spoofing'dir.

---

## 6. Order Flow Toxicity (VPIN)

Emir akışındaki "toksisite", içsel bilgiye sahip (informed) trader'ların piyasaya girdiği anları tespit eder. Hacim senkronizasyonu ile VPIN (Volume-Synchronized Probability of Informed Trading) hesaplanır. Basit versiyonu:

```
Imbalance_Bucket = |Buy_Volume - Sell_Volume|
Total_Volume_Bucket = Buy_Volume + Sell_Volume

Toxicity = EMA(Imbalance_Bucket / Total_Volume_Bucket, 50)
```
Eğer toksisite artıyorsa, piyasada yönlü ve sert bir hareket (trend) başlamak üzeredir. Market maker'lar (liquidity provider) bu durumda spread'leri açmalıdır.

---

## 7. Algoritmik Akış

```
1. Websocket üzerinden gerçek zamanlı Order Book snapshot'larını (ilk 20-50 seviye) al.
2. Belirli yüzdelik dilimler için (örn %0.5, %1, %2) Bid ve Ask Depth hesapla.
3. Order Book Imbalance (OBI) değerini hesapla ve momentum engine'e gönder.
4. Emir duvarlarını tespit et, mesafelerini kontrol et ve sahte olup olmadıklarını (spoofing) filtrele.
5. Duvarlara yaklaşıldığında, tick by tick işlem hacmini (aggressor flow) analiz ederek duvarın "yenip yenmediğini" kontrol et.
6. Execution Engine'e, "Burada büyük bir limit emir var, slippage artabilir veya sekeceğiz" bilgisini gönder.
```

---

## 8. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `DEPTH_PCT` | 1.0% | Derinlik hesaplama yüzdesi |
| `WALL_MULTIPLIER` | 5.0 | Duvar tespiti için ortalama çarpanı |
| `SPOOF_DIST_PCT` | 0.2% | Spoofing analizi için minimum mesafe |
| `OBI_THRESH` | 0.3 | Imbalance dikkate alma eşiği |
| `WS_UPDATE_RATE`| 100ms | Websocket defter güncelleme hızı |

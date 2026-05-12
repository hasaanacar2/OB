# Liquidation Engine — Matematiksel Model

## 1. Amaç

Kaldıraçlı vadeli işlem piyasalarında (Futures) katılımcıların likidasyon seviyelerini tahmin etmek ve bu likidasyonların tetikleyeceği zincirleme reaksiyonları (cascade moves/liquidations) trade etmek. Piyasa, likiditenin (stoplar ve likidasyonlar) yoğunlaştığı bölgelere bir mıknatıs gibi çekilir.

---

## 2. Likidasyon Fiyatı Tahmini

Borsalar likidasyon fiyatlarını açıklamaz, ancak açık pozisyon büyüklüğü (Open Interest - OI) ve tahmini kaldıraç oranları üzerinden matematiksel olarak hesaplanabilir.

### 2.1 Pozisyon Bazlı Likidasyon Formülü

Ortalama bir kullanıcının `Leverage` (Kaldıraç) ve `Maintenance_Margin` (Sürdürme Teminatı, genelde %0.4 - %0.5) değerleri ile:

**Long Pozisyon için Likidasyon:**
```
Liq_Price_Long = Entry_Price × (1 - 1/Leverage + Maintenance_Margin)
```

**Short Pozisyon için Likidasyon:**
```
Liq_Price_Short = Entry_Price × (1 + 1/Leverage - Maintenance_Margin)
```

*Örnek:* Entry = $50,000, Kaldıraç = 10x, MM = %0.4
`Liq_Price_Long = 50,000 × (1 - 0.10 + 0.004) = 50,000 × 0.904 = $45,200`

### 2.2 Sık Kullanılan Kaldıraç Kümeleri (Clusters)

Piyasada en sık kullanılan kaldıraç oranları 10x, 25x, 50x ve 100x'tir. Bu oranlar, birikim noktaları oluşturur. Yüksek hacimli bir mumun kapanışından (Entry) bu oranlar kadar uzağa "Likidasyon Kümeleri" (Liquidation Clusters) yerleştirilir.

| Kaldıraç | Fiyat Sapması (%) |
|---|---|
| 100x | ~0.5% |
| 50x | ~1.5% |
| 25x | ~3.5% |
| 10x | ~9.5% |

---

## 3. Liquidation Heatmap (Isı Haritası) Oluşturma

Geçmiş N gün içerisindeki yüksek hacimli bölgelerden varsayımsal pozisyonlar yaratılır ve bunların likidasyon noktaları bir haritada toplanır.

### 3.1 Yoğunluk (Density) Fonksiyonu

Belirli bir fiyat seviyesi `P` için tahmini likidasyon birikimi:

```
Liq_Density(P) = Σ (Volume_i × Weight_i)  
    ∀ i where Estimated_Liq_Price_i ∈ [P - ε, P + ε]
```
- `Volume_i`: Pozisyonun açıldığı andaki hacim (Open Interest değişimi dikkate alınarak)
- `Weight_i`: Zamanla bozunma (Time decay) katsayısı (Kapanan pozisyonları hesaba katmak için eskidikçe ağırlık düşer).

### 3.2 Temizlenme (Wipeout) Mekanizması

Fiyat bir likidasyon kümesinin üzerinden geçtiğinde o küme "patlamış" sayılır ve haritadan silinir.

```
if Current_Price crosses Cluster_P:
    Liq_Density(Cluster_P) = 0
```

---

## 4. Likidasyon Şelalesi (Cascade) Tespiti

Zincirleme likidasyon, bir likidasyonun fiyatı daha da iterek sıradaki likidasyonu tetiklemesiyle oluşur. 

### 4.1 Cascade Şartları

Cascade olması için likidasyon kümelerinin birbirine çok yakın (boşluksuz) dizilmiş olması gerekir.

```
Cascade_Probability ⟺ 
    Distance(Cluster_1, Cluster_2) < Market_Impact(Cluster_1)
```

Eğer `Cluster_1` likide olduğunda oluşacak piyasa satış/alış emri, fiyatı `Cluster_2`'ye kadar itebilecek büyüklükteyse zincirleme reaksiyon başlar.

---

## 5. Liquidation Sweep Stratejisi

Fiyat büyük bir likidasyon bölgesini patlattıktan sonra genellikle terse güçlü bir dönüş (reversal / mean reversion) yaşanır.

### 5.1 Setup

```
1. Fiyat büyük bir Liq_Cluster'a girer.
2. Borsadan "Liquidated Long/Short" websocket verileri akmaya başlar (Büyük likidasyonlar gerçekleşir).
3. Open Interest (OI) hızla düşer. (Pozisyonların kapandığı onaylanır).
4. Fiyat hızlıca bir önceki yapıya geri döner (Sweep).
```

### 5.2 Entry ve Exit

```
Entry: Likidasyon akışı yavaşladığında ve ilk ters yönlü mum (veya BOS) oluştuğunda.
SL: Likidasyon iğnesinin (wick) en ucunun hemen altı/üstü.
TP: Karşı yöndeki en yakın likidite bölgesi veya likidasyon kümesi.
```

---

## 6. Algoritmik Akış

```
1. Sürekli olarak yüksek hacimli ve delta'sı (yönü) net olan bölgeleri (örneğin breakout mumları) kaydet.
2. Bu bölgelerden 10x, 25x, 50x uzaktaki noktalara tahmini likidasyon kümeleri yerleştir.
3. OI değişimlerini izleyerek bu kümelerin boyutunu (density) güncelle. Zaman geçtikçe decay (bozunma) uygula.
4. Fiyat bir kümeye yaklaştığında (örn. %1 mesafe kaldığında) "Mıknatıs Etkisi" alarmı ver.
5. Fiyat kümeyi patlattığında OI düşüşünü ve websocket likidasyon verilerini izle.
6. Rejection (reddedilme) gerçekleşirse OB veya Liquidity Engine'e onay (confluence) gönder.
```

---

## 7. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `LEV_CLUSTERS` | [10, 25, 50, 100] | İzlenen varsayılan kaldıraçlar |
| `DECAY_RATE` | 0.05 / gün | Likidasyon ağırlığı zamanla bozunma hızı |
| `CLUSTER_TOLERANCE`| 0.5% | Küme gruplama fiyat toleransı (ε) |
| `MAGNET_ZONE` | 1.0% | Mıknatıs etkisinin başladığı mesafe |
| `MIN_OI_DROP` | 2.0% | Likidasyon sonrası teyit için gereken min OI düşüşü |

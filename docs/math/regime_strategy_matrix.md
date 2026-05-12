# Piyasa Rejimleri ve Strateji Matrisi

Piyasadaki en büyük kural şudur: **Yanlış piyasa rejiminde doğru strateji bile para kaybettirir.** Sistemdeki algoritmaların birbiriyle çakışmaması (conflict) ve kümülatif zarar üretmemesi için Regime Detection Engine (Rejim Tespit Motoru) tarafından belirlenen piyasa durumuna göre hangi stratejilerin "AÇIK" (ON) hangi stratejilerin "KAPALI" (OFF) olması gerektiği aşağıda listelenmiştir.

---

## 1. Rejimlere Göre Strateji Uygunluğu

| Piyasa Rejimi | Özellikleri | Aktif Edilecek Stratejiler (ON) | Kapatılacak Stratejiler (OFF) | Dikkat Edilmesi Gerekenler |
| :--- | :--- | :--- | :--- | :--- |
| **TRENDING** | ADX > 25, Güçlü yönlü hareket, HH/HL serisi. | - Trend Following<br>- Trend Continuation OB<br>- FVG + OB | - Mean Reversion | Trend yönünün aksine (Counter-trend) kesinlikle işlem açılmamalıdır. |
| **RANGING** | ADX < 20, Fiyat belli bir bantta sıkışmış. | - Mean Reversion<br>- Liquidity Sweep + OB<br>- Asian Session Sweep | - Trend Following<br>- Trend Continuation OB | Destek ve direnç sınırlarında (range edges) likidite avı işlemleri çok kârlıdır. |
| **HIGH_VOL** | ATR çok yüksek, sert ve geniş mumlar. | - Liquidation Engine (Şelale yakalama)<br>- Breaker Block | - Mean Reversion (Riskli) | Pozisyon büyüklükleri (Position Size) Risk Engine tarafından otomatik yarıya düşürülmelidir. |
| **LOW_VOL** | Sıkışma alanı, dar bantlar (Squeeze). | - Mean Reversion | - Trend Following | Fiyat patlamaya (breakout) hazırlandığı için her an rejim değişebilir. Sıkı stop konulmalıdır. |
| **NEWS_DRIVEN** | Haber veya makro veri anı (TÜFE, FED vb.) | **HİÇBİRİ** (Sistem İzlemeye Alınır) | Tüm Stratejiler | Sadece Execution Engine açık kalan pozisyonları korumak için stopları günceller. |

---

## 2. Çakışan (Conflicting) Stratejiler

Bazı stratejiler mantık gereği birbirine tamamen zıttır. Aynı rejimde ikisi birden açık kalırsa, bot kendi kendiyle işlem yapıp sadece borsaya komisyon öder (Whipsaw etkisi).

### 2.1 Trend Following vs. Mean Reversion
- **Çakışma Nedeni:** Trend stratejisi fiyat hareket ettikçe ekleme yapmak (veya pozisyonu tutmak) ister. Mean Reversion ise fiyat hareket ettikçe "ortalamadan uzaklaştı" diyerek ters yöne işlem açar.
- **Çözüm:** Rejim motoru **TRENDING** sinyali verdiğinde Mean Reversion anında **durdurulmalıdır (HALT)**.

### 2.2 OB Retest vs. Breaker Block
- **Çakışma Nedeni:** Fiyat bir Order Block'a (OB) geldiğinde; *Retest Stratejisi* o OB'nin çalışıp fiyatı geri çevireceğini varsayarak işlem açar. *Breaker Block Stratejisi* ise o OB'nin kırılmasını ve fiyata artık ters yönde tepki vermesini bekler. 
- **Çözüm:** OB Retest stratejisinde fiyat stop-loss'u vurup OB'yi ihlal ederse (Fail olursa), Retest stratejisi kapanır ve bölge o an itibariyle Breaker Block stratejisine devredilir. İkisi aynı anda aynı OB için aktif olamaz.



---

## 3. Optimizasyon Önerisi (Rejim Geçişleri)

Rejimlerin sınırları (örneğin RANGING'den TRENDING'e geçiş anı) her zaman net değildir. Bu gri bölgelerde (Transitional Phase):
1. **Confluence (Kesişim) Zorunluluğu:** Sadece tek bir engine sinyali yetmez. Örneğin "Liquidity Sweep" + "Volume Spike" + "FVG" üçlüsü aynı anda aranmalıdır.
2. **Kademeli Risk Alımı:** Rejim değişimi tam onaylanana kadar risk (0.25R veya 0.5R gibi) düşük tutulmalıdır.

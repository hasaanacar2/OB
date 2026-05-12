# Multi-Agent Claude Layer — Matematiksel ve Mantıksal Model

## 1. Amaç

Karmaşık, kesin sınırları olmayan ve deterministik algoritmalarla (veya basit ML modelleriyle) kodlanması zor olan analizleri (Haber okuma, makro sentiment, genel piyasa yapısı yorumu, kod/parametre optimizasyonu) Büyük Dil Modellerine (LLM - Claude) devretmek. 

Sistemi tek bir dev prompt yerine, uzmanlaşmış "Agent"lara bölerek halüsinasyonu (yanlış bilgi üretimini) azaltmak ve karar ağacını matematikselleştirmek.

---

## 2. Agent Hiyerarşisi

Sistem bir "Eksperler Paneli" gibi çalışır.

1. **Data Agent:** Ham veriyi (OHLCV, Funding, OI, Haber metinleri) JSON formatında özetler ve formatlar.
2. **Macro & Sentiment Agent:** Fed kararları, Elon Musk tweetleri, genel kripto haberlerini okur ve bir Sentiment Skoru (-1 ile +1 arası) üretir.
3. **Technical Analysis Agent:** Order Block'ları, FVG'leri ve likidite seviyelerini anlatan text verisini yorumlar. "Bu OB gerçekten mantıklı mı?" sorusunu cevaplar.
4. **Risk & Execution Agent:** Pozisyon büyüklüklerini, genel portföy riskini ve "Duplicate Trade" (Mükerrer İşlem) durumunu değerlendirir. Önüne gelen yeni sinyali mevcut aktif pozisyonlarla kıyaslar. Eğer sistem zaten aynı mantıkla açılmış bir işleme sahipse "Zaten bu yönde içerideyiz, aşırı risk (Over-exposure) alıyorsun" diyerek veto hakkını kullanır.
5. **Meta-Agent (Orchestrator):** Alt agentların skorlarını matematiksel olarak birleştirerek nihai kararı üretir.

---

## 3. Sentiment Skoru (Sentiment Analysis)

Macro Agent, haber ve metinleri okuyup deterministik bir skor çıkarır. 

```
Macro_Score = w₁ × News_Sentiment + w₂ × Twitter_Sentiment + w₃ × Onchain_Activity_Score
```

- `News_Sentiment` ∈ [-1, 1]
- `Twitter_Sentiment` ∈ [-1, 1]

Eğer `|Macro_Score| > 0.8` ise, deterministik teknik sinyalleri ezebilir (override). Örneğin, SEC'den kötü bir haber geldiyse ve teknik olarak `Bullish OB` oluştuysa, Meta-Agent bu işlemi reddeder.

---

## 4. Tetiklenme Mekanizmaları (Trigger Conditions)

Maliyet (API token) ve hız (latency) optimizasyonu nedeniyle Claude 7/24 çalışmaz (sürekli döngüde sorgu atılmaz). Yalnızca **"Event-Driven" (Olay Odaklı)** ve periyodik olarak tetiklenir.

### 4.1 Sinyal Bazlı Tetiklenme (Event-Driven)
Deterministik Python motoru (A ve B katmanları) bir OB stratejisi için `Kalite Skoru ≥ 60` (B veya A+ Setup) olan bir işlem fırsatı bulduğunda anlık olarak Claude'u tetikler.
- **Gönderilen Veri:** Setup detayları (Entry, SL, TP), güncel market rejimi ve o ana ait son dakika haberleri.
- **Claude'un Görevi:** İşlemi insan gözüyle/mantığıyla onaylamak (APPROVE) veya reddetmek (REJECT).

### 4.2 Periyodik Makro Analiz (Routine Review)
Sistemin genel piyasa yönünü (Macro Bias) anlaması için her **8 saatte bir** (örn. Asya, Londra ve NY açılışlarında) Claude tetiklenir.
- **Gönderilen Veri:** Son 8 saatin global haberleri, borsa duyuruları ve DXY/SPX verileri.
- **Claude'un Görevi:** `Macro_Score` (-1 ile +1 arası) üretmek. Bu skor 8 saat boyunca Python motorunda sabit bir filtre olarak kullanılır. Sürekli API sorgusuna gerek kalmaz.

### 4.3 Acil Durum / Volatilite Anomalisi (Emergency Trigger)
Piyasada hiçbir teknik sinyal yokken aniden devasa bir hacim veya volatilite patlaması olursa (örn. `ATR_Spike > 3.0`), Python motoru acil olarak X/Twitter ve News API'den verileri toplayıp Claude'u tetikler.
- **Claude'un Görevi:** Bunun bir Flash Crash (stop patlatma) mi yoksa yapısal bir haber mi (Savaş, Regülasyon, Hack) olduğunu algılayıp "Sistemi Durdur (HALT)" komutu vermek.
- **Veri Zehirlenmesi (Fake News) Koruması:** Eğer Claude aşırı panik yaratacak veya coşturacak bir haber okursa ("ETF Onaylandı" gibi), anında aksiyon almaz. Çapraz doğrulama (Cross-Validation) kuralı işletilir. Kaynak tek bir X(Twitter) hesabıysa ve Reuters/Bloomberg API'de eşleşmiyorsa haber *Yalan* kabul edilir ve tepkisiz kalınır.

---

## 5. Karar (Voting) Mekanizması

Python tarafı bir Trade Setup'ı hazırlayıp Meta-Agent'a yollar. Meta-Agent alt agent'lardan oy toplar.

### 4.1 Oylama Fonksiyonu

Her Agent bir puan (`S_i` ∈ [0, 100]) ve bir Güven (Confidence) seviyesi (`C_i` ∈ [0, 1]) döndürür.

Nihai Karar Puanı (Final Score):

```
Final_Score = Σ (W_i × S_i × C_i) / Σ (W_i × C_i)
```

burada `W_i` her ajanın ağırlığıdır.

| Ajan | Ağırlık (W) |
|---|---|
| Technical Agent | 0.5 |
| Macro Agent | 0.3 |
| Risk Agent | 0.2 |

### 5.2 Onay Eşiği ve Human-in-the-Loop (Telegram Onayı)

```
if Final_Score ≥ 80 ∧ Risk_Agent_Veto == False:
    Execution = "APPROVE"                 # Doğrudan gir
elif 60 ≤ Final_Score < 80 ∨ Risk_Agent_Veto == True:
    Execution = "PENDING_MANUAL_REVIEW"   # Emin değil veya Veto yedi
else:
    Execution = "REJECT"                  # Çöp sinyal
```

**Telegram Human Override (Yönetici Onayı):**
Eğer Claude işlemden tam emin olamazsa (Skor 60-80 arası) veya işlemi bir sebepten REJECT ederse (Veto), sistem işlemi doğrudan çöpe atmaz. 
Tüm setup detaylarını, grafiği ve Claude'un **Reddetme/Tereddüt Gerekçesini** bir Telegram mesajı olarak Yöneticiye (Size) atar.
Eğer Yönetici Telegram üzerinden `✅ ONAYLA` butonuna basarsa, **Claude'un ve diğer filtrelerin kararı EZİLİR (God Mode)** ve o işlem ne olursa olsun borsaya iletilir.

### 5.3 Aşırı Temkinlilik (False Negative) Koruması
Claude ve diğer LLM'ler doğası gereği aşırı riskten kaçınan (Conservative/Güvenli) bir tonda eğitilmiştir. Bizi korudukları gibi, bazen kârlı işlemleri sırf "Ufak bir belirsizlik var" diyerek reddedebilirler (Fırsat Maliyeti / Opportunity Cost).
Bunu kırmak için Claude'un ana *System Prompt*'una şu kesin kural eklenmiştir (Override Rule):
- *"Sen bir algoritmik tradersın, sadece riskten kaçan bir korumacı değilsin. Eğer Python'dan gelen Teknik Skor ≥ 85 ise (A+ Setup), makro tarafta çok ekstrem bir felaket (Savaş, Hack, İflas) yoksa işlemi KESİNLİKLE ONAYLA. Sıradan ufak tefek FUD haberleri için mükemmel teknik fırsatları reddetme."*

---

## 6. Tahmin Hafızası ve Hesap Verilebilirlik (Prediction Accountability)

LLM'lerin en büyük zaafı olan "unutkanlık" ve "sürekli kendini haklı görme" eğilimini kırmak için, Claude'un yaptığı tüm yönlü tahminler (Makro veya Teknik) bir veritabanında saklanır ve gerçekleşen piyasa koşullarıyla yüzleştirilir.

### 6.1 Prediction Log (Tahmin Kaydı)
Claude her `APPROVE` verdiğinde veya `Macro_Score` ürettiğinde, Python motoru bu tahmini kaydeder:
- **Tarih:** 12 Mayıs
- **Tahmin:** "Enflasyon verisi yüksek geldi, DXY yükselecek, BTC 60K'ya düşecek. Bias: BEARISH."

### 6.2 Reality Check (Yüzleşme ve Feedback Döngüsü)
Belirlenen hedef süre (örn. 24 saat) dolduğunda, Python motoru gerçekleşen fiyat hareketlerini alır ve Claude'a "Hesap Sorma" (Accountability) promptu yollar:
- **Sistem Mesajı:** *"Dün 'Enflasyon yüksek, BTC düşecek' demiştin. Ancak BTC son 24 saatte %4 yükselerek 65K oldu. Tahminin BAŞARISIZ. Nerede hata yaptığını açıkla ve bu hatayı bir dahaki sefere yapmamak için yeni bir çıkarım üret."*

### 6.3 Confidence Penalty (Güven Puanı Cezası)
Eğer Claude üst üste hatalı tahminler yaparsa, Risk Engine tarafından onun oylama ağırlığı otomatik olarak düşürülür (Cezalandırılır).
```
if AI_Prediction_Accuracy < 50%:
    Macro_Agent_Weight *= 0.7  (Otoritesi azaltılır)
```
Bu döngü, yapay zekanın kendi hatalarından ders almasını (Self-Correction) sağlar.

### 6.4 Context Window Yönetimi ve Hafıza (RAG Mimarisi)
Claude'un "Context Window" limiti (örn. 200K token) dolduğunda sistem çökmez veya aşırı pahalı bir API faturası çıkarmaz. Çünkü Python motorumuz, Claude ile sürekli uzayan bir sohbet akışı yerine **"Stateless" (Geçmiş sohbeti tutmayan)** bir RAG iletişim mimarisi kurar.
- Claude'un çıkardığı tüm dersler (Lessons Learned), hatalar ve geçmiş tahminler Python tarafındaki `SQLite / Vector DB` içine kaydedilir.
- Yeni bir işlem onaya giderken Python tüm aylık sohbet geçmişini API'ye YÜKLEMEZ. Veritabanından **sadece o o anki durumla ilgili (Örn: Sadece BTC düşüş trendi dersleri)** en önemli 3-4 kuralı (Rule) seçer ve küçük, sıkıştırılmış bir Prompt paketi içine yerleştirerek gönderir.
- *Sonuç:* Claude sonsuz bir hafızaya sahipmiş gibi nokta atışı hatırlar ama API'ye her seferinde sadece 500 - 1000 tokenlik (çok ucuz) veri gider. Context Window asla dolmaz.

---

## 7. Replay ve Optimizasyon Agent'ı

LLM'ler kod yazma ve analiz yeteneklerini kullanarak sistemin kendi kendini geliştirmesini sağlar.

### 5.1 Geri Bildirim Döngüsü (Feedback Loop)

Geçmiş N işlemin sonuçları (Win/Loss, MAE/MFE, Slippage) Claude'a yollanır. Claude şu optimizasyon fonksiyonunu maksimize etmeye çalışır:

```
Objective = Maximize(Sharpe_Ratio) subject to Max_Drawdown < 10%
```

Claude, parametrelerde (örn. `OB_ENTRY_LEVEL` veya `SL_ATR_BUFFER`) ne gibi değişiklikler yapılırsa metriğin iyileşeceğini matematiksel simülasyon (Replay Engine'e gönderdiği Python kodları) ile test eder.

### 5.2 Parametre Güncellemesi (Bayesian Updating)

Claude'un önerdiği yeni parametre kümesi `θ_new`, eski parametre kümesi `θ_old` ile kıyaslanır. Walk-Forward testinden geçerse:

```
θ_{t+1} = α × θ_new + (1 - α) × θ_old
```

Burada `α` (öğrenme katsayısı) sistemin ne kadar hızlı adapte olacağını belirler (örn 0.1).

---

## 8. Algoritmik Akış

```
1. (Python) Teknik Sinyali oluştur (Entry, SL, TP, RR).
2. (Python) İlgili veriyi JSON olarak hazırla. Bu JSON Prompt Paketi KESİNLİKLE şunları içerir:
   - `New_Setup`: {coin, direction, entry, sl, tp, strategy_name, timeframe}
   - `Active_Positions`: [{coin, direction, strategy_name, unrealized_pnl, duration}] (Claude'un "Mevcut İşlemlerle" kıyaslama yapması için)
   - `Market_Context`: {current_regime, recent_news_summary, HTF_bias}
3. (API) Veriyi Meta-Agent'a (Claude) gönder.
4. (Claude) Meta-Agent veriyi Macro, Technical ve Risk alt agent'larına böler.
5. (Claude) Alt agentlar bağımsız düşünür (CoT - Chain of Thought) ve S_i / C_i döndürür.
6. (Claude) Meta-Agent Final_Score hesaplar ve "APPROVE" veya "REJECT" yanıtı döner, yanında açıklama ekler.
7. (Python) Eğer yanıt APPROVE ise işlemi borsaya iletir.
8. (Haftalık) Optimizasyon Agent'ına işlem geçmişi yollanır, parametre güncelleme önerisi istenir.
```

---

## 9. Parametreler

| Parametre | Varsayılan | Açıklama |
|---|---|---|
| `APPROVAL_THRESHOLD` | 75 | İşleme girmek için LLM minimum puanı |
| `MAX_MACRO_IMPACT` | 0.8 | Makro sentimentin teknik veriyi ezme eşiği |
| `LLM_TIMEOUT` | 10s | Karar almak için maksimum api bekleme süresi |
| `CONFIDENCE_PENALTY` | 0.5 | Hallucination tespiti durumunda puan kesintisi |
| `LEARNING_RATE_ALPHA`| 0.1 | LLM önerisi ile parametre değiştirme katsayısı |

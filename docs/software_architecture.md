# Yazılım Mimarisi ve Sistem Tasarımı (Software Architecture)

Bu doküman, teorik olarak tasarlanan matematiksel OB (Order Block) ve Likidite sisteminin, **Python** üzerinde nasıl kodlanacağını, hangi teknolojilerin kullanılacağını ve proje klasör yapısının nasıl olacağını tanımlar.

---

## 1. Teknoloji Yığını (Tech Stack)

Sistemin "hızlı, deterministik ve çökmeye karşı dayanıklı" (fault-tolerant) olması için seçilen teknolojiler:

- **Ana Dil:** `Python 3.10+` (Tip güvenliği için `mypy` ve Pydantic ile)
- **Borsa Entegrasyonu:** `ccxt` (REST API için) ve `binance-futures-connector` / `websockets` (Canlı stream için)
- **Veri İşleme ve Matematik:** `pandas`, `numpy`, `pandas-ta` (Teknik indikatörler)
- **Veritabanı (State Management):** 
  - `SQLite` veya `PostgreSQL` (İşlem geçmişi, Prediction Log ve Trade Lifecycle kayıtları için)
  - `Redis` (O anki aktif rejim, fiyat anlık hafızası ve rate-limit yönetimi için - İsteğe bağlı)
- **Asenkron Yapı:** `asyncio` (Borsa websocket'ini dinlerken, aynı anda Claude API'sine istek atmak için kodun bloklanmaması şarttır)
- **Yapay Zeka API:** `Anthropic Claude 3 Opus / Sonnet API` (Hızlı analizler için Sonnet, derin makro analizler için Opus)
- **Human-in-the-Loop (Manuel Onay):** `python-telegram-bot` (Reddedilen veya arafta kalan işlemleri yöneticiye sormak ve onay almak için)

---

## 2. Klasör ve Modül Yapısı

Proje, Spagetti koda dönüşmemesi için kesin modüler (Domain-Driven) sınıflara ayrılmalıdır:

```text
OB_TRADING_BOT/
│
├── data/                       # Veritabanı dosyaları (SQLite vb.)
├── docs/                       # Dokümantasyon (math modelleri vs.)
├── logs/                       # Sistem logları (Hata, Debug)
│
├── src/                        # Ana Kaynak Kod
│   ├── main.py                 # Botu başlatan ana döngü (Orchestrator)
│   ├── config.py               # API keyleri, sabitler (Constants)
│   │
│   ├── data_ingestion/         # Veri Toplama Katmanı
│   │   ├── websocket_client.py # Canlı fiyat ve order book stream
│   │   └── rest_client.py      # Geçmiş OHLCV ve Funding Rate çekme
│   │
│   ├── engines/                # Çekirdek Motorlar (A ve B Katmanı)
│   │   ├── regime_engine.py    # ADX, Volatilite, Hysteresis hesabı
│   │   ├── liquidity_engine.py # Equal H/L, Sweep tespiti
│   │   ├── bos_choch_engine.py # Yapısal kırılım hesapları
│   │   └── scoring_engine.py   # Sinyal kalite puanlaması
│   │
│   ├── strategies/             # Aktif Stratejiler
│   │   ├── base_strategy.py    # Tüm stratejiler için ortak class
│   │   ├── ob_retest.py        # OB Retest mantığı
│   │   ├── liquidity_sweep.py  # Sweep mantığı
│   │   └── fvg_ob.py           # FVG ve OB birleşimi
│   │
│   ├── ai_layer/               # C Katmanı (Claude)
│   │   ├── claude_client.py    # Anthropic API bağlantısı
│   │   ├── prompt_manager.py   # Duruma göre üretilecek prompt şablonları
│   │   └── accountability.py   # Tahmin hafızası (Reality Check) sistemi
│   │
│   ├── execution/              # Emir ve Risk Yönetimi
│   │   ├── risk_engine.py      # Drawdown, Kelly pozisyon boyutu hesabı
│   │   ├── order_manager.py    # CCXT ile borsaya limit/market emir yollama
│   │   └── lifecycle_engine.py # Trailing stop, Kısmi TP, Stickiness
│   │
│   └── utils/                  # Yardımcı Araçlar
│       ├── logger.py           # Loglama mekanizması
│       └── math_helpers.py     # ATR, Z-Score gibi ortak fonksiyonlar
│
├── requirements.txt            # Python bağımlılıkları
└── .env                        # Gizli API Anahtarları (Binance, Anthropic)
```

---

## 3. Asenkron Çalışma Döngüsü (Event Loop)

Bot sıralı (senkron) çalışamaz. Çünkü fiyat verisi gelirken bir yandan önceki işlemin Stop-Loss'u kontrol edilmeli, bir yandan da AI'dan cevap beklenmelidir. `asyncio` mimarisi şu şekilde çalışır:

1. **Task 1 (Data Streamer):** Websocket üzerinden her 250ms'de bir Binance'ten güncel fiyatı alır ve RAM'de tutar.
2. **Task 2 (Strategy Runner):** Her mum kapanışında (Örn: 5dk, 15dk) veya fiyat kritik bir seviyeye geldiğinde stratejileri (Sweep, OB) çalıştırıp sinyal arar.
3. **Task 3 (Lifecycle Manager):** Sürekli olarak aktif pozisyonların kâr/zarar durumunu kontrol eder, gerekiyorsa Stop'u Breakeven'a çeker.
4. **Task 4 (AI Worker):** Sadece `Strategy Runner` kaliteli bir sinyal bulduğunda veya 8 saatte bir tetiklenir. API'ye asenkron istek atar, botu dondurmaz.
5. **Task 5 (Telegram Listener):** Yöneticiden (Sizden) gelecek "ONAYLA", "REDDET" veya "SİSTEMİ DURDUR" komutlarını asenkron olarak bekler ve `Execution Engine`'i zorla tetikler (Override).

---

## 4. Multi-Timeframe (Çoklu Zaman Dilimi) Mimarisi

Birçok botun battığı yer olan "aynı anda farklı zaman dilimlerinde saçmalaması" sorununu çözmek için sistem hiyerarşik (İzole ama Haberleşen) bir yapıda çalışır:

1. **İzole Tarayıcılar (Scanners):** 5 dakikalık, 1 saatlik ve 4 saatlik grafikler için `Strategy Runner` ayrı ayrı "Instance" (Süreç) olarak ayağa kalkar. 5m scalp ararken, 4h swing arar.
2. **HTF (Üst Zaman Dilimi) İzni:** 5 dakikalık sürecin işlem açabilmesi için 1 saatlik ve 4 saatlik süreçlerin verisini (HTF Bias) okuyup izin alması şarttır. Ana trend düşüşteyken (4h Bearish), 5m'deki yükseliş sinyalleri reddedilir.
3. **Merkezi Risk Havuzu:** İşlemlerin süresi (5m veya 4h) ne olursa olsun, hepsi tek bir **Risk Engine** süzgecinden geçer. 4h'de zaten büyük bir işlemimiz varsa, 5m'den gelen yeni işlem "Over-exposure" (Aşırı risk yüklenmesi) kuralına takılarak iptal edilir.
4. **Etiketleme (Tagging):** Borsaya giden her emir `[5m_Sweep]` veya `[4h_Continuation]` olarak etiketlenir. Böylece Lifecycle Engine kâr alma veya zaman dolumu (Time Decay) kurallarını bu etikete göre esnek işletir.

---

## 5. Veritabanı ve State (Durum) Yönetimi

Bot çökerse, elektrik kesilirse veya yeniden başlatılırsa **aktif işlemlerin unutulmaması** hayati önem taşır.
- Açılan her işlem, `ACTIVE` statüsü ile SQLite veritabanına yazılır.
- Bot yeniden başladığında önce Borsadan (Binance) açık pozisyonları sorgular, sonra veritabanındaki kayıtlarla eşleştirir ve `Lifecycle Manager` kaldığı yerden devam eder (Trail stoplar vb. kaybolmaz).
- Claude'un tahminleri `ai_predictions` isimli bir tabloya yazılır. `accountability.py` bu tabloyu okuyarak gerçekleşen sonuçlarla yüzleştirme yapar.

---

## 6. Hata Yönetimi (Failsafes & Fallbacks)

Canlı piyasada para yöneten bir yazılımda "Error Handling" her şeydir:
- **Borsa API Çökerse:** Bot `REST_MODE`'a geçer ve websocket yerine 1 dakikada bir istek atar. O da çalışmazsa `EMERGENCY_HALT` moduna girer.
- **Sessiz Bağlantı Kopması (Silent WS Drop):** Binance bazen bağlantıyı koparır ama hata fırlatmaz. Fiyat donar ve trailing-stoplar çalışmaz. Bunu engellemek için `websocket_client.py` her 3 saniyede bir borsaya `Ping` atar. `Pong` yanıtı gelmezse bağlantıyı zorla kapatır ve anında yeni bir kanal açar (Watchdog Mechanism).
- **Claude API Cevap Vermezse veya Hata Verirse (Timeout):** İşlem çok kârlı bir A+ Setup olsa bile risksiz ilerlemek adına "AI Onayı Yok" (No AI Consent) diyerek o işlemi **pas geçer**. Asla körü körüne işleme girmez.
- **Max Drawdown Aşılırsa:** Günlük zarar %3'ü geçerse, Python motoru hiçbir stratejiyi çalıştırmaz, kendisini gece 00:00'a (UTC) kadar uyku moduna (Sleep) alır (İntikam Trade'ini engeller).
- **Rate Limits:** Binance API saniye/dakika limitleri (Weights) takip edilir. Limit %90'a ulaştığında istekler yavaşlatılır.

---

## 7. Geliştirme (Deployment) Adımları

Sistem tek seferde canlıya alınmaz (No YOLO Deployment). Aşama aşama gidilmelidir:
1. **Paper Trading (Dry Run):** Borsa API'si bağlanır ama emirler borsaya gitmez, veritabanına "sanal" olarak kaydedilir. Claude entegre edilir.
2. **Walk-Forward Analiz:** Bot 1-2 hafta sanal parayla çalıştırılarak "Churn" (sürekli strateji değiştirme) ve "Rejim Geçişi" kurallarının doğru çalışıp çalışmadığı canlı canlı izlenir.
3. **Live with Micro-Risk:** Gerçek parayla, ancak normal riskin 1/10'u kullanılarak (örn. işlem başı 10 Dolar risk) canlı teste alınır. Slippage ve komisyonların matematiğe uygunluğu test edilir.
4. **Full Production:** Her şey kusursuz çalışıyorsa `Risk Engine` normal değerlerine çekilir.

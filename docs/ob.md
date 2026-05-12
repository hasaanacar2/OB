OB Trading Stratejileri

Not: Bu doküman yatırım tavsiyesi değildir. Amaç, OB stratejilerini Python + Claude hibrit sisteminde deterministik şekilde modellemektir.

1. Genel Mantık

Order Block, güçlü bir fiyat hareketinden önce oluşan son ters yönlü mum bölgesidir.

Basit tanım:

Bullish OB = Güçlü yükselişten önceki son bearish candle bölgesi
Bearish OB = Güçlü düşüşten önceki son bullish candle bölgesi

OB stratejisinde Claude karar verici değil, değerlendirici olmalıdır.

Python = OB tespiti, risk, emir, backtest
Claude = açıklama, haber yorumu, kalite değerlendirmesi
2. Gerekli Kütüphaneler
Ana Python kütüphaneleri
pip install pandas numpy ccxt ta python-binance matplotlib requests
Kullanım amaçları
pandas        → OHLCV veri işleme
numpy         → matematiksel hesaplamalar
ccxt          → Binance ve diğer borsalardan veri/işlem
python-binance→ Binance özel entegrasyon
ta / pandas-ta→ ATR, RSI, EMA, Bollinger vb.
matplotlib    → backtest grafikleme
requests      → haber/API veri çekme

CCXT, birçok kripto borsasına ortak API ile bağlanmayı sağlayan bir trading kütüphanesidir; Binance dahil market data, analiz, bot ve backtest altyapılarında kullanılabilir.

Binance tarafında kline/candlestick verileri resmi olarak open time ile tanımlanır ve futures için /fapi/v1/klines endpoint’i kullanılabilir.

Canlı takip için Binance futures websocket kline stream’i mevcut kline verilerini yaklaşık 250 ms aralıklarla güncelleyebilir.

3. Veri Nereden Alınır?
Ana veri kaynakları
1. Binance OHLCV
2. Binance Futures funding rate
3. Binance open interest
4. Binance order book
5. Binance mark price / index price
6. Haber kaynakları
7. Coin listing duyuruları
8. Sosyal medya / sentiment kaynakları
OB için minimum veri
open
high
low
close
volume
timestamp
Futures için ek veri
funding_rate
open_interest
long_short_ratio
mark_price
liquidation data

Binance futures tarafında open interest verisi ayrıca endpoint olarak sunulur; bu, pozisyon yoğunluğu ve crowded trade riskini yorumlamak için kullanılabilir.

4. OB Matematiksel Tanımı
Candle yapısı
body = abs(close - open)
range = high - low
upper_wick = high - max(open, close)
lower_wick = min(open, close) - low
ATR
ATR = average_true_range(high, low, close, window=14)
Impulse şartı

Bir hareketin güçlü kabul edilmesi için:

impulse_candle_range >= ATR * 1.5

veya:

move_size >= ATR * 2
BOS şartı

Bullish BOS:

close > previous_swing_high

Bearish BOS:

close < previous_swing_low
Bullish OB
1. Son bearish candle bulunur.
2. Ardından impulsive bullish move gelir.
3. Fiyat previous swing high kırar.
4. Son bearish candle bölgesi bullish OB olur.
Bearish OB
1. Son bullish candle bulunur.
2. Ardından impulsive bearish move gelir.
3. Fiyat previous swing low kırar.
4. Son bullish candle bölgesi bearish OB olur.
5. OB Bölgesi Nasıl Çizilir?
Full candle OB
OB High = candle high
OB Low  = candle low
Body OB
OB High = max(open, close)
OB Low  = min(open, close)
50% entry
OB Mid = (OB High + OB Low) / 2

Başlangıç için en deterministik yapı:

OB zone = full candle
Entry = OB 50%
Stop = OB dışı + ATR buffer
6. OB Retest Stratejisi
Mantık

Fiyat güçlü hareketten sonra OB bölgesine geri döner.

Şartlar
1. BOS oluşmalı
2. Impulse ATR * 1.5 üstü olmalı
3. OB yaşı max 80 candle olmalı
4. Fiyat OB’ye geri dönmeli
5. RR minimum 1:2 olmalı
Entry
Bullish OB:
Entry = OB 50%
SL = OB Low - ATR * 0.2
TP = en yakın liquidity / RR 2+
Ne zaman kullanılır?
Trend devamı beklenen piyasalarda
HTF yön net olduğunda
Volatilite normal seviyedeyken
Pozisyon süresi
5m setup  → 15 dakika - 2 saat
15m setup → 1 - 6 saat
1h setup  → 4 - 24 saat
4h setup  → 1 - 5 gün
7. Liquidity Sweep + OB
Mantık

Market önce stopları toplar, sonra gerçek yön başlar.

Bullish örnek
1. Equal lows oluşur
2. Fiyat equal lows altına iner
3. Hızlı reject gelir
4. Bullish BOS oluşur
5. OB retest alınır
Matematiksel sweep şartı
low < previous_equal_low
close > previous_equal_low
Ek kalite şartı
sweep_wick_size >= ATR * 0.5
Ne zaman kullanılır?
Range sonrası
London/New York açılışlarında
Yüksek likidite bölgelerinde
Fake breakout sonrası
Pozisyon süresi
Scalp: 5 dakika - 1 saat
Intraday: 1 - 6 saat

Bu strateji OB sisteminde en güçlü çekirdek stratejilerden biridir.

8. Breaker Block
Mantık

Çalışmayan OB ters yöne destek/direnç olur.

Bullish OB failure
1. Bullish OB oluşur
2. Fiyat OB altına kırar
3. OB artık bearish breaker olur
4. Fiyat bölgeye geri döner
5. Short aranır
Matematiksel şart
close < OB_low

Sonra:

retest_price ∈ [OB_low, OB_high]
Ne zaman kullanılır?
Trend dönüşlerinde
Failed OB sonrası
Güçlü momentum kırılımında
Pozisyon süresi
15m setup → 1 - 4 saat
1h setup  → 4 - 24 saat
9. FVG + OB
Mantık

OB bölgesi ile Fair Value Gap aynı bölgede birleşirse setup kalitesi artar.

Bullish FVG
candle_1_high < candle_3_low
Bearish FVG
candle_1_low > candle_3_high
Confluence şartı
OB zone ∩ FVG zone != empty
Ne zaman kullanılır?
Güçlü impulsive hareket sonrası
Temiz imbalance olduğunda
HTF yön ile uyumlu olduğunda
Pozisyon süresi
5m / 15m → 30 dakika - 4 saat
1h       → 4 - 24 saat
10. Asian Session Sweep
Mantık

Asya seansında range oluşur, London veya New York açılışında liquidity alınır.

Şartlar
1. Asia range high/low belirlenir
2. London open sonrası range dışına sweep gelir
3. Sweep sonrası BOS oluşur
4. OB retest beklenir
Matematiksel tanım
asia_high = max(high during Asia session)
asia_low  = min(low during Asia session)

Bullish setup:

low < asia_low
close > asia_low
bullish BOS
OB retest
Ne zaman kullanılır?
London open
New York open
Düşük volatil Asya seansı sonrası
Pozisyon süresi
15 dakika - 3 saat
11. Trend Continuation OB
Mantık

Ana trend yönünde pullback sonrası OB’den devam işlemi alınır.

Trend filtresi
EMA 50 > EMA 200 → bullish trend
EMA 50 < EMA 200 → bearish trend
Bullish continuation
1. EMA50 > EMA200
2. Higher high / higher low oluşuyor
3. Pullback gelir
4. Mini bullish OB oluşur
5. Trend yönünde long alınır
Ne zaman kullanılır?
Net trend piyasasında
BTC yönü güçlü olduğunda
Altcoinler BTC ile uyumluyken
Pozisyon süresi
15m setup → 1 - 6 saat
1h setup  → 4 - 24 saat
4h setup  → 1 - 7 gün
12. Hangi Durumda Hangi OB Stratejisi?
Market Durumu	Kullanılacak OB Stratejisi
Net trend	Trend Continuation OB
Trend sonrası pullback	OB Retest
Range sonrası fake breakout	Liquidity Sweep + OB
Güçlü impulse sonrası boşluk	FVG + OB
Başarısız OB	Breaker Block
London/New York açılışı	Asian Session Sweep
13. OB Kalite Skoru

Her sinyal için skor hesaplanmalı.

score = 0

if bos_confirmed:
    score += 20

if impulse_atr_ratio >= 1.5:
    score += 20

if liquidity_sweep:
    score += 20

if fvg_overlap:
    score += 15

if htf_aligned:
    score += 15

if volume_confirmed:
    score += 10
Karar
score >= 80 → güçlü sinyal
score 60-79 → orta sinyal
score < 60 → işlem yok
14. Risk ve Pozisyon Süresi
Minimum RR
RR >= 2.0
Pozisyon kapatma şartları
1. TP geldi
2. SL geldi
3. BOS ters yöne kırıldı
4. Maksimum pozisyon süresi doldu
5. News volatility başladı
6. Claude risk uyarısı verdi ama Python hard risk gördü
Maksimum süre
5m strategy  → max 2 saat
15m strategy → max 6 saat
1h strategy  → max 24 saat
4h strategy  → max 7 gün
15. Python OB Engine Akışı
1. Binance candle verisi çek
2. Swing high / swing low bul
3. BOS / CHOCH tespit et
4. Impulse mumlarını tespit et
5. OB zone oluştur
6. FVG overlap kontrol et
7. Liquidity sweep kontrol et
8. HTF bias kontrol et
9. RR hesapla
10. Score hesapla
11. Claude’a değerlendirme için gönder
12. Risk Engine’den geçir
13. Telegram approval gerekiyorsa beklet
14. Binance emri gönder
# non-ob-strategy-roadmap.md

# A / B / C Katmanı Strateji Yapısı
> (OB stratejileri hariç)

Bu doküman:
- hangi stratejilerin gerçekten gerekli olduğunu,
- hangi aşamada eklenmesi gerektiğini,
- hangi market koşullarında kullanıldığını,
- sistem karmaşıklığını,
- risk seviyesini

anlatır.

---

# A KATMANI — CORE FOUNDATION

> Sistemin çalışması için gerekli minimum strateji ve altyapılar.

Bu katmandaki amaç:

```text
stabilite
deterministik execution
risk kontrolü
replay/backtest
1. Liquidity Engine
Amaç

Marketin stop topladığı bölgeleri tespit etmek.

Mantık
equal highs
equal lows
session highs
previous highs/lows

gibi bölgeler liquidity alanıdır.

Market çoğu zaman:

liquidity → reaction

davranışı gösterir.

Kullanıldığı Yer
Liquidity Sweep
Reversal setup
Breakout validation
Matematiksel Yapı

Equal highs:

∣H
1
	​

−H
2
	​

∣<ϵ

Neden Gerekli?

Çünkü:

gerçek market hareketleri çoğu zaman liquidity driven’dır
2. BOS / CHOCH Engine
Amaç

Market structure değişimini tespit etmek.

BOS

Trend continuation.

Bullish:

close > previous_swing_high
CHOCH

Trend değişim sinyali.

Kullanıldığı Yer
Trend continuation
Reversal detection
Bias confirmation
Neden Gerekli?

Çünkü:

directional bias olmadan signal quality düşer
3. Risk Engine
Amaç

Sistemin hayatta kalmasını sağlamak.

Görevleri
position sizing
daily loss limit
hard/soft limits
drawdown control
leverage limit
Örnek
MAX_RISK_PER_TRADE = 0.5%
MAX_DAILY_LOSS = 3%
Neden Gerekli?

Çünkü:

edge + kötü risk yönetimi
=
hesap sıfırlanır
4. Execution Engine
Amaç

Deterministik emir yönetimi.

Görevleri
entry
stop-loss
take-profit
partial close
trailing stop
cancelation
Neden Gerekli?

Çünkü:

signal doğru olsa bile
execution kötü olabilir
5. Replay / Backtest Engine
Amaç

Gerçek edge’i bulmak.

Kullanımı
historical replay
performance testing
market regime analysis
Neden Gerekli?

Çünkü:

canlı markette öğrenmek pahalıdır
6. Trade Lifecycle Engine
Amaç

Trade açıldıktan sonraki yönetim.

Lifecycle
entry
↓
partial TP
↓
breakeven
↓
trailing stop
↓
exit
Neden Gerekli?

Çünkü:

profit management
=
edge’in önemli kısmı
A Katmanı Sonucu

Bu katman sonunda sistem:

deterministic
stable
replayable
safe

olur.

B KATMANI — EDGE BOOSTERS

Winrate ve consistency artıran stratejiler.

Bunlar:

sistem stabil olduktan sonra
veri toplandıktan sonra
eklenmelidir.
7. Trend Following Engine
Amaç

Güçlü trendleri takip etmek.

Kullanımı
EMA trend
ADX
market structure
pullback continuation
Kullanıldığı Market
strong trend
BTC directional move
momentum phase
Avantajı
trend markette çok güçlüdür
Dezavantajı
range markette ölür
8. Mean Reversion Engine
Amaç

Aşırı uzaklaşan fiyatın ortalamaya dönmesini trade etmek.

Kullanılanlar
VWAP
Bollinger
RSI
Z-score
Kullanıldığı Market
range market
low volatility
Avantajı
range markette güçlüdür
Dezavantajı
trend markette tehlikelidir
9. Volume Engine
Amaç

Gerçek hareketleri fake hareketlerden ayırmak.

Kullanılan Veriler
volume spike
delta
OI
bid/ask imbalance
Kullanımı
breakout validation
trend strength
momentum confirmation
Avantajı
fake breakout filtreler
10. Funding Engine
Amaç

Crowded long/short ortamlarını analiz etmek.

Kullanılan Veriler
funding rate
open interest
liquidations
long/short ratio
Kullanıldığı Market
overheated market
extreme leverage
squeeze environments
Avantajı
short squeeze / long squeeze yakalayabilir
11. Session Engine
Amaç

Market davranışını session bazlı analiz etmek.

Session Türleri
Asia
London
New York
Kullanımı
Asia range
London sweep
NY continuation
Avantajı
zaman bazlı edge sağlar
12. Regime Detection Engine
Amaç

Market tipini belirlemek.

Market Rejimleri
TRENDING
RANGING
HIGH_VOL
LOW_VOL
NEWS_DRIVEN
Kullanımı
regime
↓
strategy selection
Neden Çok Önemli?

Çünkü:

yanlış markette doğru strategy bile kaybettirir
B Katmanı Sonucu

Bu katman sonunda sistem:

adaptive
multi-regime
higher winrate
more stable

hale gelir.

C KATMANI — ADVANCED / QUANT

Production sonrası eklenmesi gereken ileri seviye stratejiler.

Bunlar:

complexity artırır
bakım zorlaştırır
optimization ister

16. Correlation Engine
Amaç

Piyasalar arası ilişkiyi analiz etmek.

Kullanılan Veriler
BTC dominance
NASDAQ
DXY
ETH/BTC
SPX
Kullanımı
macro correlation
risk-on / risk-off
Avantajı
global risk anlayışı sağlar
17. Order Book Engine
Amaç

Gerçek zamanlı liquidity analizi yapmak.

Kullanılan Veriler
depth
bid walls
ask walls
imbalance
spoofing
Avantajı
microstructure edge sağlar
Dezavantajı
yüksek latency hassasiyeti
18. Liquidation Engine
Amaç

Likidasyon bölgelerini trade etmek.

Kullanılan Veriler
liquidation heatmap
OI
funding
leverage clusters
Avantajı
cascade move yakalayabilir
Kullanıldığı Market
high leverage environments

20. Multi-Agent Claude Layer
Amaç

AI görevlerini uzman agent’lara bölmek.

Örnek Agentlar
Market Analyst
Risk Analyst
Macro Analyst
Replay Analyst
Avantajı
daha kontrollü AI davranışı
# Sistem Mimari ve Akış Diyagramı

Aşağıdaki diyagram; dışarıdan gelen verinin hangi motorlarda (engine) işlendiğini, rejim motorunun stratejileri nasıl kontrol ettiğini ve bir sinyalin hangi süzgeçlerden geçerek borsaya emir olarak iletildiğini adım adım göstermektedir.

```mermaid
graph TD
    %% Renklendirme ve Stiller
    classDef data fill:#f9f2ec,stroke:#d9b38c,stroke-width:2px,color:#333
    classDef core fill:#e6f3ff,stroke:#66b3ff,stroke-width:2px,color:#333
    classDef edge fill:#f0f9e8,stroke:#a1d99b,stroke-width:2px,color:#333
    classDef adv fill:#f2e6ff,stroke:#b366ff,stroke-width:2px,color:#333
    classDef strat fill:#fff0f5,stroke:#ff99c2,stroke-width:2px,color:#333
    classDef exec fill:#ffe6e6,stroke:#ff6666,stroke-width:2px,color:#333

    %% 1. VERİ KAYNAKLARI
    subgraph Data_Layer [1. VERİ KAYNAKLARI]
        D1(Binance OHLCV) ::: data
        D2(Order Book & OI) ::: data
        D3(Funding & Liq Verisi) ::: data
        D4(Haber / Makro Veri) ::: data
    end

    %% 2. A KATMANI - ÇEKİRDEK (TEMEL YAPI)
    subgraph Layer_A [2. A KATMANI - ÇEKİRDEK YAPI]
        A1(Liquidity Engine <br/> Equal Low/High, Swings) ::: core
        A2(BOS / CHOCH Engine <br/> Market Structure) ::: core
    end

    %% 3. B KATMANI - FİLTRELER VE REJİM
    subgraph Layer_B [3. B KATMANI - FİLTRE VE REJİM]
        B1(Trend & Volume Engine) ::: edge
        B2(Funding & Session Engine) ::: edge
        B3{Regime Detection Engine <br/> Hysteresis & Churn Koruması} ::: edge
    end

    %% 4. C KATMANI - İLERİ ANALİZ
    subgraph Layer_C [4. C KATMANI - İLERİ ANALİZ & AI]
        C1(Correlation Engine) ::: adv
        C2(Liquidation & OB Engine) ::: adv
        C3(Multi-Agent Claude <br/> Karar & Veto Sistemi) ::: adv
    end

    %% 5. STRATEJİLER
    subgraph Strategies [5. AKTİF OB STRATEJİLERİ]
        S1(Liquidity Sweep + OB) ::: strat
        S2(Trend Continuation OB) ::: strat
        S3(FVG + OB) ::: strat
        S4(Breaker Block) ::: strat
        S5(Asian Session Sweep) ::: strat
        S6(OB Retest) ::: strat
    end

    %% 6. RİSK VE UYGULAMA (EXECUTION)
    subgraph Risk_Exec [6. RİSK VE YÖNETİM AKIŞI]
        R1{OB Kalite Skoru <br/> A+, B, C Sinyal} ::: exec
        R2(Risk Engine <br/> Position Sizing & Drawdown) ::: exec
        R3(Execution Engine <br/> Borsaya Emir İletimi) ::: exec
        R4(Trade Lifecycle Engine <br/> SL, TP, Breakeven, Stickiness) ::: exec
    end

    %% BAĞLANTILAR
    D1 & D2 & D3 & D4 --> Layer_A
    D1 & D2 & D3 & D4 --> Layer_B
    D1 & D2 & D3 & D4 --> C1 & C2

    Layer_A --> |Temel Yapı| Strategies
    B1 & B2 --> B3
    B3 -.-> |"Hangi Rejimdeyiz? Hangi Stratejiler AÇIK?"| Strategies

    Strategies --> |"Taslak Sinyal (Entry, SL, TP)"| R1
    
    R1 --> |"Puanı Düşükse (ÇÖP) İPTAL"| Reject_1((İPTAL))
    R1 --> |"Puanı Yüksekse"| C3
    
    C1 & C2 -.-> |Ek Veri ve Sentiment| C3
    
    C3 --> |"Haber/Makro Kötüyse VETO"| Reject_2((İPTAL))
    C3 --> |"Claude Onayı (APPROVE)"| R2
    
    R2 --> |"Risk Limitleri Uygunsa Boyut Hesapla"| R3
    R2 --> |"Günlük Zarar Dolduysa"| Reject_3((İPTAL))
    
    R3 --> |"Limit Emir Gönderildi"| R4
    R4 -.-> |"Trail Stop, Kısmi Kar, Strateji Sadakati"| R4

```

### Akışın Özeti
1. **Veri Akar:** Borsa ve dış kaynaklardan tüm saniye bazlı veya mum bazlı veriler gelir.
2. **Rejim Belirlenir:** B Katmanı, veriyi işleyerek "Şu an hangi rejimdeyiz?" kararını verir. Rejim motoru, geçici gürültülerde sürekli fikir değiştirmez (Hysteresis kuralı).
3. **Strateji Seçilir:** Rejime uygun olan stratejiler aktif edilir. A Katmanı bu stratejilere destek/direnç, likidite seviyelerini besler.
4. **Sinyal Üretilir ve Puanlanır:** O anki bir strateji (örneğin Liquidity Sweep) sinyal üretirse, OB Kalite Skoru algoritmasından geçer. Skor düşükse çöpe atılır.
5. **AI Veto ve Risk:** Skor yeterliyse, veriler Multi-Agent Claude'a (C Katmanı) onaya gider. Ters bir makro haber veya anomali yoksa Risk Engine'e düşer. Risk Engine kasa kontrolü yaparak işlem büyüklüğünü (Position Size) belirler.
6. **Uygulama (Lifecycle):** Execution Engine emri borsaya girer. Emir aktifleştiği an *Trade Lifecycle Engine* yönetimi devralır (Breakeven'a çekme, SL'i kaydırma). Arka planda rejim kısa süreliğine değişse bile *Strategy Stickiness* gereği işlem sonuca ulaşana kadar kapatılmaz.

# USD/JPY × BQML × JEV 設計書

## 1. 目的

USD/JPY の15分足CSVだけを入力として、BQMLで短期の将来方向を予測し、その予測をJEVで分析・判断する実験基盤を構築する。

本設計では自動売買・発注を行わない。まず「過去の価格データから未来をどの程度予測できるか」を再現可能な形で検証する。

## 2. 基本構成

```
USD/JPY 15m CSV
      ↓
feature engineering
      ↓
BQML
      ↓
future prediction
      ↓
JEV
      ↓
analysis / decision context
      ↓
backtest
```

役割を分離する。

- **CSV**: 観測された市場データ
- **BQML**: 過去データから未来の値・方向を推定
- **JEV**: 予測値を市場状態の文脈に置き、分析結果を生成
- **Backtest**: 過去期間で予測を検証

## 3. 入力データ

最小CSV:

```csv
timestamp,open,high,low,close,volume
2026-09-18 09:00,155.42,155.51,155.39,155.48,1200
```

必須:

- timestamp
- open
- high
- low
- close

任意:

- volume

対象:

- pair: USD/JPY
- timeframe: 15m

## 4. 特徴量

最初は価格系列だけから生成する。

### Return

- return_1
- return_2
- return_4
- return_8

### Range

- high_low_range
- candle_body
- upper_wick
- lower_wick

### Trend

- SMA
- EMA
- close / SMA
- EMA slope

### Volatility

- rolling standard deviation
- rolling high-low range

volume が存在する場合:

- volume_change
- rolling volume

外部ニュースや金利等は最初の実験では入れず、価格CSVだけで成立するモデルを先に検証する。

## 5. 予測対象

### Direction

次の15分足が上昇するかを分類する。

```
target_15m =
  1 if close(t+1) > close(t)
  0 otherwise
```

### Return

必要に応じて次のリターンも予測する。

```
future_return_15m =
  close(t+1) / close(t) - 1
```

### Multi-horizon

同じ現在状態から複数の未来を見る。

| Horizon | 内容 |
|---|---|
| t+1 | 15分後 |
| t+2 | 30分後 |
| t+4 | 1時間後 |
| t+8 | 2時間後 |

## 6. BQML

BQMLではまず分類モデルを中心に実験する。

入力:

```
features(t)
```

出力:

```
P(up at t+h)
```

重要なのは、未来の値を特徴量に混ぜないこと。

train / validation / test は時間順に分割し、ランダムシャッフルによる未来情報の混入を避ける。

評価指標:

- accuracy
- precision
- recall
- log loss
- calibration
- confusion matrix

単純な方向予測ベースラインも保存する。

## 7. JEV

BQMLは「予測」を担当し、JEVは「予測をどう読むか」を担当する。

例:

```json
{
  "pair": "USD/JPY",
  "timeframe": "15m",
  "prediction": {
    "15m_up_probability": 0.68,
    "30m_up_probability": 0.61,
    "1h_up_probability": 0.51,
    "2h_up_probability": 0.43
  }
}
```

JEVの入力には予測だけでなく、現在の観測状態も含める。

```text
price
trend
momentum
volatility
support
resistance
prediction
    ↓
JEV
    ↓
UP / DOWN / RANGE
+ explanation
+ confidence
```

JEVの役割は、BQMLの予測確率を単純に売買シグナルへ変換することではない。

## 8. 「荒く作る → 絞る」

この実験ではJEVの判断構造を次のようにする。

```
大量の市場観測
      ↓
粗い候補
      ↓
trend / momentum / volatility
      ↓
BQML prediction
      ↓
候補を絞る
      ↓
JEV analysis
```

BQMLが「未来についての候補」を生成し、JEVが現在の状況との整合性を分析する。

## 9. Backtest

過去データを時系列順に流し、

```
過去データ
   ↓
その時点までの特徴量
   ↓
BQML prediction
   ↓
JEV analysis
   ↓
実際の次の足
   ↓
prediction と実績を比較
```

とする。

未来データを参照して特徴量を作らない。

保存する項目:

- timestamp
- actual_return
- predicted_probability
- predicted_direction
- JEV analysis
- outcome

## 10. チャート

基本表示:

- USD/JPY 15分ローソク足
- SMA / EMA
- support / resistance
- BQML prediction
- JEV analysis

時間軸:

- 5m
- 15m
- 1h
- 4h
- 1D

15mを主分析軸とし、上位足はコンテキストとして扱う。

## 11. 将来拡張

第一段階ではCSVのみ。

第二段階:

- 複数時間足
- 経済指標
- 金利
- ニュース
- market regime

第三段階:

- JEVによる候補選別
- paper trading
- より厳密なバックテスト

自動発注はさらに別レイヤーとし、この設計書のPOCには含めない。

## 12. 成果物

想定構成:

```
data/
  usdjpy_15m.csv

bqml/
  features.sql
  train.sql
  predict.sql

jev/
  schema.yaml
  analysis.yaml

backtest/
  README.md

README.md
DESIGN.md
```

## 13. 注意

このシステムは過去データに基づく予測実験であり、将来の為替価格や利益を保証するものではない。

モデル評価では、精度だけでなく時系列リーク、過学習、regime change、取引コストなどを別途検証する。

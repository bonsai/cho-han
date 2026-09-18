# 宣言

システムが採用するルール・制約・不変条件を宣言する。

- 未来情報を特徴量に混入させない
- train / validation / test は時間順に扱う
- BQMLはPredictionを生成する
- JEVはPredictionをContextとして整理する
- BacktestはPredictionとOutcomeを比較する
- POCでは自動発注を行わない
- 「荒く作る → 絞る → 検証する」を基本ループとする

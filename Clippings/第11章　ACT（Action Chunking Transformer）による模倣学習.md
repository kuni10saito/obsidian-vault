# 第11章 ACT（Action Chunking Transformer）による模倣学習 まとめ

## 模倣学習の2つの課題
- **人間デモの一貫性のなさ**：同じタスクでも経路が異なり、単純平均すると中途半端な方策になる
- **非マルコフ性**：現在の状態だけでは次の行動を決められず、無限ループに陥る可能性がある

## ACTの2つの鍵
- **Transformer（Self-Attention）**：過去の状態・行動の系列全体を考慮し、非マルコフ性を克服。デモのばらつきにも対応
- **Action Chunking**：1ステップずつでなく、複数ステップ分の行動をまとめて予測
  - 動作が滑らかになる
  - 誤差の累積が抑えられる
  - 実行効率が上がる

## アーキテクチャ
1. エンコーダー（CNN/ViT）：画像から特徴抽出
2. Transformer Encoder-Decoder：時系列の文脈を理解
3. Action Chunking機構：複数ステップ先を予測
- Conditional VAEでデモの多峰性を潜在変数として表現

## Temporal Ensemble
- 毎ステップ新チャンクを予測し、過去の複数チャンクの該当時刻を加重平均
- チャンクの切れ目のカクつきを平滑化

## SO-101での実装
- LeRobotで標準サポート、`--policy.type=act --policy.chunk_size=10` などで学習
- 推論はTemporal Ensembleで平滑化しつつ30Hzで実行

## 性能向上のテクニック
- データ拡張（ColorJitter、ぼかし、平行移動・回転）
- **Chunk sizeは10前後が最適**（実験では成功率92%がピーク）
- マルチモーダル学習（複数カメラ視点の融合）

## 評価とデバッグ
- 初期化→自律実行→成功判定を繰り返して成功率を測定
- 予測系列のプロット、Attentionのヒートマップ、予測分散の確認が有効
- よくある症状（損失が下がらない、振動する等）とその対処法を整理

## ACT vs Diffusion Policy
- ACTは**推論速度・学習安定性で優位**、SO-101では第一候補（◎）
- Diffusion Policyはマルチモーダル性に強いが実装が複雑（△）

## 結論
ACTは、Transformerによる非マルコフ性克服、Action Chunkingによる滑らかな動作、Temporal Ensembleによる平滑化を組み合わせ、SO-101での模倣学習における第一選択の手法である。
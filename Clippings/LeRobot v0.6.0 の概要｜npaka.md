# LeRobot v0.6.0 概要まとめ

2026年7月7日、Hugging Faceが「LeRobot v0.6.0」をリリース。テーマは**Imagine・Evaluate・Improve**。

## 1. ワールドモデル系ポリシー追加
- **VLA-JEPA**: Qwen3-VL-2Bベース、JEPAで将来フレームを潜在空間予測（推論時は軽量）
- **LingBot-VA**: 映像と行動を同時予測、24〜32GB GPUで推論可能
- **FastWAM**: 映像生成×行動生成の専門家モデル組み合わせ

## 2. VLAモデルの拡充
- **GR00T N1.7**（NVIDIA）: Flash Attention不要に
- **MolmoAct2**: SO-100/101でゼロショット可、bf16推論約12GB
- **EO-1**: Qwen2.5-VL-3Bベース、flow-matching
- **Multitask DiT**: CLIP埋め込み条件のdiffusion transformer
- **EVO1**: 0.77Bの軽量VLA

## 3. 報酬モデルAPI追加
- `lerobot.rewards`で統一API化
- **Robometer**: Qwen3-VL-4Bベースの汎用スコアリング
- **TOPReward**: 既存VLMでゼロショット報酬推定

## 4. データセット機能強化
- 動画エンコード設定の柔軟化（コーデック・GOP等）
- **深度データ対応**（RealSense、12bit深度動画）
- 言語アノテーション拡充（サブタスク・VQA等）、`lerobot-annotate`CLI追加
- 動画読み込み最大2倍高速化

## 5. 評価ベンチマーク統合
`lerobot-eval`で6ベンチマーク追加：
- LIBERO-plus / RoboTwin 2.0 / RoboCasa365 / RoboCerebra / RoboMME / VLABench
- 既存と合わせ計9ベンチマークファミリーに対応

## 6. lerobot-rollout（デプロイ専用CLI）
- 実行モード：**base / sentry / highlight / episodic / dagger**
- 特に`dagger`は人間による介入修正データ収集が可能

## 7. 学習基盤強化
- **FSDP**対応（Accelerate経由、大規模モデル学習）
- **HF Jobs**でクラウド学習（T4〜8×H200）

## 8. インストール・整理
- ベース依存関係を約40%削減、extras指定制に
- PyTorch 2.7〜2.11対応、uv.lockでCI管理

## 9. エコシステム
- **LeLab**: ブラウザUIでワークフロー操作（SO-ARM101対応）
- **Isaac Teleop**: NVIDIAとの協力、VRテレオペ対応

## まとめ
LeRobotは低コストロボット向け模倣学習ツールから、VLA・ワールドモデル・報酬モデル・評価・実機改善ループを含む**ロボット学習の総合基盤**へ進化。
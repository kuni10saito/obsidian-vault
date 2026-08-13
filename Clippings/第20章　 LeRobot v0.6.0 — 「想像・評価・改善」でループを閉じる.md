# 第20章要約：LeRobot v0.6.0 — 「想像・評価・改善」でループを閉じる

2026年7月7日リリースの **LeRobot v0.6.0** は、テーマ「**Imagine／Evaluate／Improve**」のもと、これまで手作業だったワークフローを本体機能として統合したアップデート。

## 全体像（7領域の強化）
- **ワールドモデル系ポリシー**：VLA-JEPA / LingBot-VA / FastWAM（学習時に未来予測、推論時は軽量）
- **VLAモデル群**：GR00T N1.7 / MolmoAct2 / EO-1 / Multitask DiT / EVO1
- **報酬モデルAPI**：`lerobot.rewards`（Robometer / TOPReward）
- **評価ベンチマーク**：`lerobot-eval` に6種追加、計9ファミリー統合
- **デプロイCLI**：`lerobot-rollout`（DAgger含む5戦略）
- **データセット**：深度対応・言語アノテーション・コーデック選択・約2倍高速化
- **学習基盤**：FSDP対応・HF Jobsによるクラウド学習

## インストール・破壊的変更
- pip依存が約40%削減、`[training]`等のextrasで機能追加
- GR00TはN1.7に置き換え（N1.5は`lerobot==0.5.1`で固定要）
- macOSでのキーボード操作がアクセシビリティ権限なしで動作

## Imagine：ワールドモデル系ポリシー
- **VLA-JEPA**：推論時にワールドモデル部分が消え、コスト増なし
- **LingBot-VA**：想像映像を保存し実際のロールアウトと比較可能
- **FastWAM**：推論時は映像生成をスキップし直接アクション生成

## VLAモデルの選び方（SO-101向け）
- ゼロショットで動かす→**MolmoAct2**
- 小型GPU向け→**EVO1**（0.77B）
- 自作学習で理解したい→**Multitask DiT**（約450M）
- ワールドモデルを試す→**VLA-JEPA**

## Evaluate：報酬モデルと評価ベンチマーク
- `lerobot.rewards`：**Robometer**（汎用スコアリング）、**TOPReward**（既製VLM活用）
- 用途：報酬付き行動クローニング、データセット品質チェック、進捗オーバーレイ動画生成
- `lerobot-eval`：LIBERO-plus、RoboTwin 2.0、RoboCasa365、RoboCerebra、RoboMME、VLABenchを統合、並列評価で最大2倍高速化

## Improve：lerobot-rolloutと改善ループ
- 戦略：base / sentry / highlight / episodic / **dagger**
- **DAgger戦略**：失敗時に人間が介入・修正動作を記録→ファインチューンに活用
- リーダーアーム構成がそのまま活用可能

## データセット機能強化
- コーデックを自由選択（ハードウェアエンコーダ対応、Macは自動VideoToolbox）
- 深度データ対応（RealSense、12bit圧縮保存）
- 言語アノテーションのリッチ化（`lerobot-annotate`でVLM自動アノテーション）
- 読み込み最大2倍高速化、学習の正確な再開が可能

## 学習基盤の強化
- **FSDP**：GPUを超えるモデルの分散学習、チェックポイントは単一ファイルに統合
- **HF Jobs**：`--job.target`フラグでクラウド学習可能（T4〜8x H200）

## エコシステム
- **LeLab**：CLI不要のブラウザUIツール（SO-ARM101対応）
- **Isaac Teleop**：VRコントローラでのテレオペ
- 計算機ハードウェアガイド新設、自作ポリシー・プラグイン対応拡充

## SO-101ユーザーの実践ルート
1. アップグレード→MolmoAct2でゼロショット実行
2. 既存データセットにRobometerで品質チェック
3. DAggerで修正データ収集
4. ファインチューン（必要ならHF Jobs）
5. 1〜4を繰り返す（lerobot-evalで定量評価）

## まとめ
v0.6.0は改善ループの各工程——**評価（報酬モデル・ベンチマーク）、修正収集（DAgger）、再学習（FSDP・HF Jobs）**——を標準機能として提供。個人・小規模チームでも短サイクルでの改善が可能になった。
# 第14章　Isaac SimによるReal-to-Simシミュレーション　要約

## 14.1 Isaac Sim / Isaac Labとは
- Isaac Sim：NVIDIA Omniverse上のロボティクス向け物理シミュレーター（PhysX・レイトレーシング・OpenUSD対応）
- 主要ワークフロー：①合成データ生成 ②ロボットスタック検証（SIL/HIL） ③ロボット学習（Isaac Lab経由）
- Isaac Lab：Isaac Sim上の学習用フレームワーク（環境管理・並列化・センサー抽象化・タスク定義）
- 階層構造：ロボット学習アプリ → Isaac Lab → Isaac Sim

## 14.2 SO-101との連携プロジェクト
- **isaac_so_arm101**（MuammerBay）：Reach/PickAndPlaceタスク、PPOによる強化学習
- **LeIsaac**（Seeed Studio）：実機リーダーでシミュ内フォロワーを操作し、LeRobot形式でデータ収集
- **Real-to-Sim Twin**（Vipin）：ROS2で実機とIsaac Simを双方向リアルタイム同期

## 14.3 GR00T N1.5との統合
- NVIDIAのVLA基盤モデル、SO-101と公式統合対応
- LeIsaacで収集→LeRobot形式に変換→ファインチューニング→推論サーバー/制御クライアントで分離デプロイ

## 14.4 Real-to-Sim-to-Realワークフロー
1. 実機で50エピソード程度のデモ収集
2. Isaac Labでシミュデータを大量生成（多様な初期条件・外乱）
3. 実機＋シミュデータを統合しACT/GR00Tで学習
4. Sim評価とReal評価を反復し、Sim-to-Realギャップを縮小

## 14.5 Sim-to-Realギャップの対処法
- 物理パラメータ調整（剛性・減衰・摩擦）
- Domain Randomization（摩擦・質量・照明などをランダム化）
- センサーノイズの追加（カメラへのガウシアンノイズ等）

## 14.6 トラブルシューティング
- FPS低下→拡張機能・レンダリング設定を軽量化
- Sim成功/実機失敗→物理パラメータ調整、Domain Randomization、ノイズ追加
- テレオペ遅延→CPU親和性・リアルタイム優先度・QoS最適化
- ROS2トピック混入→ROS_DOMAIN_IDで分離

## 14.7 まとめと学習パス
- **初級**：Isaac Sim基本操作、isaac_so_arm101のダミー/ランダムエージェント実行
- **中級**：LeIsaacでのテレオペ・データ収集、カスタムタスク作成
- **上級**：強化学習訓練、GR00Tファインチューニング、実機デプロイとギャップ最小化
- 次章はNVIDIA Cosmos（生成AIワールドモデル）との連携を解説
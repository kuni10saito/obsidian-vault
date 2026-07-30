# GR00T SO-101 作業ガイド

## 環境

| 役割 | ホスト | 備考 |
|---|---|---|
| FMV（実行端末） | ローカル Windows 11 | Claude Code、クライアントスクリプト |
| DGX Spark（学習・推論） | `saito@100.103.6.70` | GB10 GPU、119GB共有メモリ |

---

## DGX Spark 接続

```bash
ssh saito@100.103.6.70
```

---

## サーバー起動

### N1.6（port 5555 / groot conda env）

```bash
# 通常使用モデル
tmux new-session -d -s server_n16 "bash -c 'cd /home/saito/Isaac-GR00T && /home/saito/miniconda3/envs/groot/bin/python gr00t/eval/run_gr00t_server.py --model-path /home/saito/so101_ft_v2/checkpoint-1000 --port 5555 2>&1 | tee /tmp/server_n16.log'"

# ログ確認
tail -f /tmp/server_n16.log
```

### N1.7（port 5556 / .venv）

```bash
tmux new-session -d -s server_n17 "bash -c 'source /home/saito/Isaac-GR00T/.venv/bin/activate && source /home/saito/Isaac-GR00T/scripts/activate_spark.sh; cd /home/saito/Isaac-GR00T && python gr00t/eval/run_gr00t_server.py --model-path /home/saito/so101_ft_n17/so101_n17_v4/checkpoint-2000 --port 5556 2>&1 | tee /tmp/server_n17.log'"

# ログ確認
tail -f /tmp/server_n17.log
```

### サーバー停止

```bash
pkill -f run_gr00t_server
tmux kill-session -t server_n16
tmux kill-session -t server_n17
```

---

## クライアントスクリプト（FMV実行）

| スクリプト | 接続先 | 用途 |
|---|---|---|
| `Desktop/run_groot.py` | DGX Spark port 5555 | N1.6 自律動作 |
| `Desktop/run_groot_n17.py` | DGX Spark port 5556 | N1.7 自律動作 |

```bash
# FMVから実行
python Desktop/run_groot.py
python Desktop/run_groot_n17.py
```

### キー操作

| キー | 動作 |
|---|---|
| `g` | 自律動作開始 |
| `h` | ホームポジションに移動 |
| `o` | グリッパー強制開放 |
| `s` | 停止 |
| `q` | 終了 |

---

## 学習済みモデル一覧（DGX Spark）

| ディレクトリ | 最終CP | 状態 |
|---|---|---|
| `~/so101_ft_checkpoint/` | checkpoint-10000 | N1.6初期学習（97ep） |
| `~/so101_ft_v2/` | checkpoint-1000 | **現用** 完璧動作確認済み（2026-05-28） |
| `~/so101_ft_v3/` | checkpoint-3000 | v3データ追加学習（2026-06-04〜） |
| `~/so101_ft_v3_fixed/` | checkpoint-3000 | v3改良版（2026-06-03完成） |
| `~/so101_ft_n17/so101_n17_v4/` | checkpoint-2000 | N1.7版 |
| `~/so101_models_backup/` | checkpoint-1000_v2_20260528 | バックアップ |

---

## ファインチューニング

### N1.6（groot conda env）

```bash
# DGX Spark上で実行
nohup /home/saito/miniconda3/envs/groot/bin/python3 \
  /home/saito/Isaac-GR00T/gr00t/experiment/launch_finetune.py \
  --base-model-path /home/saito/so101_ft_checkpoint/checkpoint-10000 \
  --dataset-path /home/saito/so101_dataset_v3 \
  --embodiment-tag NEW_EMBODIMENT \
  --modality-config-path /home/saito/Isaac-GR00T/examples/SO101/so101_config.py \
  --num-gpus 1 \
  --output-dir /home/saito/so101_ft_v3 \
  --save-steps 500 \
  --max-steps 3000 \
  --global-batch-size 32 \
  > /tmp/groot_finetune.log 2>&1 &

# ログ確認
tail -f /tmp/groot_finetune.log
```

### データセット転送（FMV → DGX Spark）

```bash
# FMVから実行
scp -r Desktop/so101_dataset_v3 saito@100.103.6.70:~/so101_dataset_v3

# info.json に chunks_size:1000 が必要
ssh saito@100.103.6.70 "python3 -c \"
import json
with open('so101_dataset_v3/meta/info.json') as f: d=json.load(f)
d['chunks_size']=1000
with open('so101_dataset_v3/meta/info.json','w') as f: json.dump(d,f)
\""
```

---

## HOME_POSITION（確定値 2026-05-28）

```python
HOME_POSITION = {1: 1505, 2: 2242, 3: 1400, 4: 2375, 5: 2438, 6: 3100}
```

| モーター | 値 | 備考 |
|---|---|---|
| M1 shoulder_pan | 1505 | 25度ズレ補正済み |
| M2 shoulder_lift | 2242 | +10度が最適 |
| M3 elbow_flex | 1400 | アプローチ感 |
| M4 wrist_flex | 2375 | +30度 |
| M5 wrist_roll | 2438 | v2データ範囲内 |
| M6 gripper | 3100 | 開放位置 |

---

## 重要な注意事項

- **学習中はサーバーを落とすこと**（GPU OOM競合）
- **カメラ位置が最重要**：端末間で移動後は必ず学習時と同じ位置・角度に戻す
- `git pull` すると Eagle ファイルが消える → `stash@{1}` から復元必要
- N1.6 env: `groot` conda / N1.7 env: `.venv`（transformers バージョン競合のため分離）
- HOME を学習データの開始姿勢分布から外すとモデルが迷走する

---

## デバッグ用

```bash
# プロセス確認
ps aux | grep run_gr00t | grep -v grep

# tmuxセッション確認
tmux list-sessions

# GPU使用確認
nvidia-smi

# チェックポイント確認
ls -d ~/so101_ft_v3/checkpoint-* | sort -V
```

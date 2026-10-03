# ロボット・Omniverse 作業ガイド（vault）

> **このフォルダ（Obsidian vault）が「ロボット・Omniverse」分野の拠点。** 2026-10-02 にホームの CLAUDE.md から移設した。
> - Claude Code はこの vault で起動する（Claudian も同じ）。ホームの `~/CLAUDE.md`（マシン構成の共通ガイド）も自動で読まれる。
> - ファイルの実体はホームやデスクトップに散らばったまま。参照先は `.claude/settings.local.json` の additionalDirectories に登録済み。
> - 自動メモリは vault 専用（`~/.claude/projects/C--Users-saito-vault/memory/`）。講義・会社のメモリとは混ざらない。
> - ⚠️ vault は GitHub（kuni10saito/obsidian-vault）へ自動バックアップされる。**パスワード・APIキーはここに書かない。**

## 場所の索引（Omniverse・ACE・地図・シミュレーション）

| 分野 | 場所 |
|---|---|
| SO-101 × GR00T（実機・データ） | `~/isaac_so_arm101/`、`~/lerobot/`、`~/Desktop/`（`run_groot*.py`・較正json・診断スクリプト）、vault `SO101まとめ/` |
| Omniverse / Isaac Sim | `~/omniverse/`、`~/.nvidia-omniverse/` |
| ACE・アバター（Audio2Face） | `~/Audio2Face/`、`~/omniverse_avatar/`、`~/avatar_demo/`、`~/dgx_spark_ace_demo.md`、`~/note_ace_article.md` |
| 地図情報（PLATEAU 名古屋・Cesium GIS） | `~/create_nagoya_usd.py`、`~/nagoya_init.py`、`~/nagoya_autorun.py`、`~/query_plateau.py`、`~/plateau_omniverse_handoff.md`、`~/Desktop/Cesium_Omniverse_GIS.pptx` |
| 琵琶湖シミュレーション（PINN・Kit CAE） | `~/biwako_3d/`、vault `琵琶湖/`・`biwa/` |
| ロボット導入講義 | vault `ロボット導入講義/` |

---

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
tmux new-session -d -s server_n16 "bash -c 'cd /home/saito/Isaac-GR00T && /home/saito/miniconda3/envs/groot/bin/python gr00t/eval/run_gr00t_server.py --model-path /home/saito/so101_models_backup/checkpoint-1000_v2_20260528 --port 5555 2>&1 | tee /tmp/server_n16.log'"

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
pkill -f "[r]un_gr00t_server"
tmux kill-session -t server_n16
tmux kill-session -t server_n17
```

> **`[r]un_...` の角括弧は必須。** `pkill -f run_gr00t_server` と素で書くと、
> **SSH越しに実行したとき自分自身のコマンド行にマッチして接続ごと切れる**
> （リモートの `bash -c` のコマンド文字列に `run_gr00t_server` が含まれるため）。
> 角括弧で囲むと正規表現としては同じ文字列にマッチするが、
> コマンド行そのものは `[r]un_...` なのでマッチせず、自爆しない。

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

> **2026-09-06 に実測で棚卸し。ここに載っているパスは全て存在を確認済み。**
>
> ⚠️ `~/so101_ft_v2/` `~/so101_ft_v3/` `~/so101_ft_v3_fixed/` は**削除済み**（容量確保のため）。
> 旧「現用」だった v2 の実体は `~/so101_models_backup/checkpoint-1000_v2_20260528` に退避してある。
> **存在しないパスをここに書かないこと**（サーバーが無言で起動失敗する）。

### N1.6（`Gr00tN1d6` / base `nvidia/Eagle-Block2A-2B-v2`）

| パス | 最終step | データセット | 状態 |
|---|---|---|---|
| `~/so101_models_backup/checkpoint-1000_v2_20260528` | 1000 | `so101_dataset_v2`（10ep） | **現用**（旧 `so101_ft_v2`） |
| `~/so101_models_backup/checkpoint-3000_v3_20260606` | 3000 | `so101_dataset_v3`（173ep） | 旧 `so101_ft_v3` |
| `~/so101_ft_checkpoint/checkpoint-10000` | 10000 | `so101_dataset_train`（97ep） | N1.6初期学習 |

### N1.7（`Gr00tN1d7` / base `nvidia/Cosmos-Reason2-2B`）

| パス | 最終step | データセット | 備考 |
|---|---|---|---|
| `~/so101_ft_n17/so101_n17_v4/checkpoint-2000` | 2000 | `so101_dataset_v4`（122ep） | **N1.7 現用** |
| `~/so101_ft_n17_v6/checkpoint-3000` | 3000 | `so101_dataset_v6`（262ep） | |
| `~/so101_ft_n17_v7/checkpoint-2500` | 2500 | `so101_dataset_v7_noplen`（140ep） | |
| `~/so101_ft_n17_v8x/checkpoint-1500` | 1500 | `so101_dataset_v8x`（40ep） | |
| `~/so101_ft_n17_v8y/checkpoint-2000` | 2000 | `so101_dataset_v8y`（120ep） | |
| `~/so101_ft_n17_v8z/checkpoint-2000` | 2000 | `so101_dataset_v8z`（120ep） | |
| `~/so101_ft_n17_v8z2/checkpoint-2000` | 2000 | `so101_dataset_v8z2`（120ep） | |
| `~/so101_ft_n17_v18/checkpoint-10000` | 10000 | `so101_dataset_v6` | 派生元 `v17` は**削除済み**（再現不可） |
| `~/so101_ft_n17_v19/checkpoint-3000` | 3000 | `so101_dataset_v6` | `v18/checkpoint-10000` から継続 |
| `~/so101_ft_n17_c0_real/checkpoint-3000` | 3000 | `so101_dataset_v4_c0`（31ep） | |
| `~/so101_ft_n17_c0_cosmos62/checkpoint-3000` | 3000 | `so101_dataset_v4_c0_cosmos62`（62ep） | c0 を Cosmos で2倍に拡張 |

#### 合成データ pretrain → 実機データ real の2段構成

`*_pretrain` は Isaac Sim の合成データ（各300ep）、`*_real` はそれを起点に `so101_dataset_v4` で追加学習。

| pretrain | → real | 合成データ |
|---|---|---|
| `~/so101_ft_n17_color_pretrain/checkpoint-3000` | `~/so101_ft_n17_color_real/checkpoint-2000` | `so101_isaac_color` |
| `~/so101_ft_n17_color3_pretrain/checkpoint-3000` | `~/so101_ft_n17_color3_real/checkpoint-2000` | `so101_isaac_color3` |
| `~/so101_ft_n17_color5_pretrain/checkpoint-3000` | `~/so101_ft_n17_color5_real/checkpoint-2000` | `so101_isaac_color5` |
| （`color4_pretrain` は**削除済み**） | `~/so101_ft_n17_color4_real/checkpoint-2000` | 再現不可 |

上表は 2026-09-06 に `trainer_state.json` / `experiment_cfg/conf.yaml` を全走査して作成した実測値。
全21ラン・71チェックポイントが存在し、いずれも完走済み（`global_step == max_steps`）。

> **データセットの `repo_id` は全て `None`**（HF Hub に push していないローカル LeRobot 形式）。
> 識別子はローカルパスのみ。データセット14件は `~/` 配下に現存。

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
  --output-dir /home/saito/so101_ft_<新規ラン名> \
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
# プロセス確認（[r] は自分自身にマッチさせないため。grep -v grep は不要になる）
ps aux | grep "[r]un_gr00t"

# tmuxセッション確認
tmux list-sessions

# GPU使用確認（学習中はサーバーを落とすこと。OOM競合する）
nvidia-smi

# チェックポイント確認（全ラン横断）
find ~/ -maxdepth 4 -type d -name 'checkpoint-*' | sort

# optimizer.pt の残数（推論には不要。2026-09-06 に全71件・920GB を削除済み → 0 が正常）
find ~/ -maxdepth 6 -name optimizer.pt -path '*checkpoint-*' | wc -l

# モデルの容量ランキング
du -sh ~/so101_ft* ~/so101_models_backup 2>/dev/null | sort -rh | head
```

---
tags: [IsaacLab, IsaacSim, 強化学習, ヒューマノイド, 四足歩行, H1, Go2, DGX-Spark, PPO]
created: 2026-08-29
updated: 2026-08-30
---

# Isaac Lab 歩行学習（H1 / Go2）作業ガイド

DGX Spark 上で Unitree H1（二足）と Go2（四足）を強化学習させ、**シミュレーション内で歩行を獲得**するまでの記録。
関連：[[一覧/GR00T]]（SO-101 模倣学習側）／[[Clippings/第14章　Isaac SimによるReal-to-Simシミュレーション]]

---

## 結論

3タスクで歩行を獲得済み。いずれも DGX Spark 単体で完結。

| | **H1 Flat**（二足・平地） | **H1 Rough**（二足・凹凸） | **Go2 Flat**（四足・平地） |
|---|---|---|---|
| タスク | `Isaac-Velocity-Flat-H1-v0` | `Isaac-Velocity-Rough-H1-v0` | `Isaac-Velocity-Flat-Unitree-Go2-v0` |
| 達成日 | 2026-08-29 | 2026-08-30 | 2026-08-30 |
| 学習時間 | 約20分（1000 iter） | 約38分（1500 iter） | **約4分**（300 iter） |
| スループット | 59,000〜69,000 steps/s | 約65,000 steps/s | **約104,000 steps/s** |
| Mean reward | −6.47 → **+34.0** | −5.47 → **+25.5** | −6.08 → **+34.9** |
| 報酬が反転 | iteration **150** | — | iteration **75** |
| 実質収束 | iteration **350** | iteration **600**（地形上限） | iteration **250** |
| 地形レベル | （平地） | **0.0 → 5.81** | （平地） |

> **① Rough の報酬が Flat より低い（+25.5 vs +34.0）のは性能が悪いのではなく、難易度が高いため。**
> 比較すべきは絶対値ではなく、それぞれの収束の仕方。
>
> **② 四足（Go2）は二足（H1）より圧倒的に速い。**
> 反転が150→75、収束が350→250と**必要イテレーションが半分以下**。
> 加えて1イテレーションあたりも速いため、実時間では **20分 → 4分（5倍速）**。
> 四足は静的に安定（常に3脚接地できる）なのに対し、二足は本質的に動的バランス問題であるため。

動画（すべて H.264 / 1280x720 / 50fps）：

**Go2（四足・平地）**
- 30秒版（25体）：![[その他/go2_flat_20260830.mp4]]

**H1 Rough（二足・凹凸・階段）**
- 30秒版（16体）：![[その他/h1_rough_20260830.mp4]] ← 岩場・ブロック段差・階段・スロープを踏破

**H1 Flat（二足・平地）**
- **学習過程（40秒）**：![[その他/h1_学習過程_20260829.mp4]] ← 転倒→反転→歩容獲得を4段階で
- 30秒版（36体）：![[その他/h1_walk_30sec_20260829.mp4]]
- 10秒版（16体）：![[その他/h1_walk_20260829.mp4]]

---

## 環境構成

| 項目 | 値 |
|---|---|
| ホスト | DGX Spark `saito@100.103.6.70`（GB10 / aarch64 / 統合メモリ122GB） |
| Isaac Sim | **5.1.0-rc.19** → `/home/saito/isaacsim`（21GB、導入済み） |
| Isaac Lab | **2.3.0** → `/home/saito/IsaacLab` |
| NVIDIA driver | **580.95.05** |
| 物理エンジン | **標準 PhysX（GPU動作OK）** — Newtonブランチ不要 |

> **重要**：GB10 では「PhysX の GPU 加速が非対応」という既知issueがあるが、
> **Isaac Sim 5.1 + driver 580.95.05 の組み合わせでは発生しない**。実測で確認済み。
> `Using device: cuda:0` が出て `nvrtc: invalid value for --gpu-architecture` は出ない。

---

## セットアップ手順

### 1. Isaac Lab 取得（Isaac Sim 5.1 に対応するのは v2.3.0）

```bash
cd /home/saito
git clone --branch v2.3.0 --depth 1 https://github.com/isaac-sim/IsaacLab.git IsaacLab
cd IsaacLab
ln -sfn /home/saito/isaacsim ./_isaac_sim
./isaaclab.sh --install
```

※ 依存の dev パッケージ（libx11-dev 等6点）は導入済み。sudo は不要。

### 2. ⚠️ コアパッケージ `isaaclab` の手動修復（必須）

**`./isaaclab.sh --install` は exit 0 で終わるのに、コアの `isaaclab` だけインストールされない。**
`isaaclab_assets` / `_tasks` / `_rl` / `_mimic` は入るため気づきにくい。
放置すると `ModuleNotFoundError: No module named 'isaaclab'` で学習が起動しない。

3段構えの依存問題なので、以下の順序で解消する：

```bash
PY=/home/saito/IsaacLab/_isaac_sim/kit/python/bin/python3

# ① ビルド依存 toml（隔離ビルドを切ると自前で用意が必要）
$PY -m pip install toml

# ② quadprog は Cython が要るので「隔離ビルド有効」で単体先行インストール
$PY -m pip install quadprog

# ③ isaaclab 本体は「隔離ビルド無効」で入れる
$PY -m pip install --no-build-isolation -e /home/saito/IsaacLab/source/isaaclab
```

**なぜこうなるか**
1. `source/isaaclab/setup.py` が `pkg_resources` を import するが、
   pip の隔離ビルド環境は最新 setuptools（81で `pkg_resources` 削除済み）を引く
   → `--no-build-isolation` で手元の setuptools 78.1.1 を使わせる
2. しかし隔離を切ると、ビルド依存の `toml` が解決されなくなる → 手動install
3. さらに依存 `quadprog` が Cython 不在で `quadprog.c` を生成できず失敗
   → quadprog だけは隔離ビルド有効で先に入れる

成功すると `isaaclab-0.47.2` が入る。

### 3. 動作確認（スモークテスト）

```bash
cd /home/saito/IsaacLab
export TERM=xterm
export LD_PRELOAD="$LD_PRELOAD:/lib/aarch64-linux-gnu/libgomp.so.1"
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
  --task=Isaac-Velocity-Flat-H1-v0 --headless --num_envs=64 --max_iterations=10
```

---

## 学習の実行

スクリプト：`/home/saito/run_h1_train.sh`

```bash
cd /home/saito/IsaacLab
export TERM=xterm
export LD_PRELOAD="$LD_PRELOAD:/lib/aarch64-linux-gnu/libgomp.so.1"
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
  --task=Isaac-Velocity-Flat-H1-v0 \
  --headless --num_envs=4096 --max_iterations=1000 --seed=42
```

tmux 経由での起動：

```bash
tmux new-session -d -s h1_train /home/saito/run_h1_train.sh
tail -f /tmp/h1_train.log
```

出力先：`logs/rsl_rl/h1_flat/<timestamp>/model_*.pt`（50イテレーションごと保存）

---

## 歩行の可視化（動画出力）

スクリプト：`/home/saito/run_h1_play.sh`

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/play.py \
  --task=Isaac-Velocity-Flat-H1-Play-v0 \
  --headless --num_envs=16 --video --video_length=500
```

- `--video` を渡すと `enable_cameras` は自動で有効化される（明示指定は不要）
- 出力：`logs/rsl_rl/h1_flat/<timestamp>/videos/play/rl-video-step-0.mp4`
- チェックポイントは最新（`model_999.pt`）が自動で読まれる

FMV への回収：

```bash
scp saito@100.103.6.70:/home/saito/IsaacLab/logs/rsl_rl/h1_flat/TIMESTAMP/videos/play/rl-video-step-0.mp4 .
```

### 学習過程を1本の動画にまとめる

スクリプト：`/home/saito/run_h1_progress.sh`（録画）→ `/home/saito/compose_progress.sh`（ラベル合成・結合）

任意のチェックポイントを `--checkpoint` で指定して個別に録画できる：

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/play.py \
  --task=Isaac-Velocity-Flat-H1-Play-v0 \
  --headless --num_envs=16 --video --video_length=500 \
  --checkpoint=RUN_DIR/model_150.pt
```

出力は毎回 `videos/play/rl-video-step-0.mp4` に**上書き**されるので、1本録るたびに退避する。
その後 ffmpeg でラベルを焼き込み、`concat` で結合する。

> ⚠️ **ffmpeg drawtext で日本語は使わないこと。**
> Noto CJK は `.ttc`（フォントコレクション）のため、`fontfile=` でも `font=`（fontconfig）でも
> グリフ解決に失敗して**2文字目以降が豆腐（□）になる**。`textfile=` でも同じ結果。
> ラベルは **DejaVuSans-Bold + 英語**にするのが確実（`/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf`）。
> なお drawtext の値の中では `:` をエスケープする必要があるが、シェル経由だと壊れやすいので
> **コロン自体を使わない**のが安全。

---

## タスクの中身：`Isaac-Velocity-Flat-H1-v0`

Unitree H1（19関節）に「指令された速度で移動せよ」を PPO で学習させる。
**歩き方は一切教えていない。** 以下の報酬設計だけで二足歩行の歩容が創発する。

| 報酬項 | 重み | 意味 |
|---|---|---|
| `track_lin_vel_xy_exp` | **+1.0** | 前後左右の速度指令に追従 ← 本命 |
| `track_ang_vel_z_exp` | **+1.0** | 旋回速度指令に追従 |
| `feet_air_time` | **+1.0** | 足を0.6秒以上宙に浮かせる ← **歩容を作る項** |
| `termination_penalty` | **−200.0** | 転倒に大ペナルティ |
| `flat_orientation_l2` | −1.0 | 上体を傾けない |
| `action_rate_l2` | −0.005 | ガクガク動作の抑制 |
| `dof_acc_l2` | −1.25e-7 | 加速度の抑制 |
| `joint_pos_limits`(ankle) | −1.0 | 足首の可動域超過を抑制 |

学習設定：`num_steps_per_env=24` / `max_iterations=1000` / `save_interval=50` / `learning_rate=1.0e-3`

**SO-101の模倣学習との根本的な違い**：お手本データが1本も要らない。
GR00T側が「人間のデモを真似る」のに対し、こちらは「報酬だけ与えて自力で発見させる」。

---

## 学習曲線（実測）

| iteration | Mean reward | 状態 |
|---|---|---|
| 0 | −0.16 | 初期 |
| 50 | **−4.83** | **転倒フェーズ**（termination_penalty を踏み続ける） |
| 100 | −1.39 | 立ち始める |
| **150** | **+13.10** | **反転。「転ばない」を獲得** |
| 200 | +22.86 | 歩容形成中 |
| 250 | +29.23 | |
| **350** | **+32.94** | **実質収束** |
| 500〜999 | +33〜34.5 | プラトー |

> **知見**：報酬が序盤に下がるのは正常。ここで異常と判断して止めないこと。
> **1000イテレーションは過剰**。次回から `--max_iterations=400` で十分（約8分に短縮できる）。

---

## Rough地形版：`Isaac-Velocity-Rough-H1-v0`

平地版との違いは3点だけ。環境設定は Rough が親で、Flat がそれを継承して簡略化している。

| 要素 | Flat | Rough |
|---|---|---|
| 地形 | `terrain_type = "plane"` | **手続き生成**（岩場・ブロック段差・階段・スロープ） |
| 高さセンサ | なし | **`height_scanner`**（torso_link に装着、足元の地形形状を観測） |
| カリキュラム | なし | **`terrain_levels`**（難易度を自動調整） |

### 実行

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
  --task=Isaac-Velocity-Rough-H1-v0 \
  --headless --num_envs=4096 --max_iterations=1500 --seed=42
```

スクリプト：`/home/saito/run_h1_rough.sh` → 録画は `/home/saito/run_h1_rough_play.sh`
出力先：`logs/rsl_rl/h1_rough/<timestamp>/`

### 学習曲線（実測）

| iteration | Mean reward | terrain_levels | 状態 |
|---|---|---|---|
| 0 | −0.14 | 3.53 | 初期（全レベルに分散配置） |
| **150** | +3.09 | **0.86** | **降格**。歩けないので易しい地形に引き戻される |
| 300 | +8.27 | 3.20 | 再上昇 |
| 450 | +13.15 | 5.38 | |
| **600** | +14.79 | **6.28** | **地形難易度が上限に到達** |
| 900 | +20.33 | 5.82 | レベル固定、習熟フェーズへ |
| 1200 | +22.68 | 5.82 | |
| **1499** | **+25.46** | 5.81 | 完了 |

> **カリキュラムの読み方**：`terrain_levels` は「今どの難易度で訓練しているか」の平均値。
> **序盤に下がるのが正常**（失敗した環境は易しい地形へ降格される自己調整機構）。
> 600以降レベルが頭打ちでも報酬が伸び続けるのは、**同じ地形での習熟が進んでいる**ということ。
> つまり `terrain_levels` の停滞＝学習停滞ではない。判断は必ず reward と併せて行う。

---

## 四足版：Unitree Go2

タスク名は H1 と命名規則が違い、**メーカー名が入る**ので注意：

| | タスク名 | デフォルト iter |
|---|---|---|
| 平地 | `Isaac-Velocity-Flat-Unitree-Go2-v0` | **300** |
| 凹凸 | `Isaac-Velocity-Rough-Unitree-Go2-v0` | 1500 |

Play 側は `-Play-v0` を付ける。ログは `logs/rsl_rl/unitree_go2_flat/`（`unitree_go2_rough/`）。

### 実行

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
  --task=Isaac-Velocity-Flat-Unitree-Go2-v0 \
  --headless --num_envs=4096 --max_iterations=300 --seed=42
```

スクリプト：`/home/saito/run_go2_flat.sh` → 録画は `/home/saito/run_go2_flat_play.sh`

### 学習曲線（Flat・実測）

| iteration | Mean reward | 状態 |
|---|---|---|
| 0 | −0.52 | 初期 |
| 25 | **−6.08** | 転倒フェーズ（H1より遥かに短い） |
| 50 | −1.17 | |
| **75** | **+5.96** | **反転**（H1 Flatは150で反転） |
| 150 | +23.39 | |
| **250** | +33.45 | **実質収束**（H1 Flatは350） |
| 299 | **+34.90** | 完了 |

### ⚠️ Go2 の Rough は極端に重い（実測 10倍差）

**`Isaac-Velocity-Rough-Unitree-Go2-v0` は H1 Rough の約6分の1の速度しか出ない。**

| | スループット | 1500 iter のETA |
|---|---|---|
| H1 Rough | 約65,000 steps/s | 約38分 |
| **Go2 Rough** | **約10,300 steps/s** | **約3時間45分** |
| Go2 Flat | 約104,000 steps/s | （300 iterで約4分） |

原因は `go2/rough_env_cfg.py` の地形設定：

```python
# 小型ロボット用に地形を縮小している
self.scene.terrain.terrain_generator.sub_terrains["random_rough"].noise_step = 0.01
self.scene.terrain.terrain_generator.sub_terrains["boxes"].grid_height_range = (0.025, 0.1)
```

Go2 は体高40cm程度なので H1（身長180cm）と同じ地形では歩けず、凹凸を細かく作り直している。
その `noise_step = 0.01`（1cm刻み）が**衝突メッシュの三角形数を激増させ**、PhysXの衝突判定コストを押し上げる。
GPU使用率は91%で正常稼働しており**誤設定ではない**——「小型ロボットほど地形が細かくなり重くなる」という構造的トレードオフ。

> **対処**：Go2 で凹凸地形をやるなら、①`noise_step` を 0.02〜0.03 に緩める、
> ②`--max_iterations` を 600 程度に減らす（四足は収束が速いので十分）、
> ③時間があるならバックグラウンドで放置、のいずれか。

---

## トラブルシューティング

| 症状 | 原因・対処 |
|---|---|
| `ModuleNotFoundError: No module named 'isaaclab'` | 上記「コアパッケージの手動修復」を実施 |
| `ModuleNotFoundError: No module named 'omni'` | 素の python で import した場合に出る。**必ず `./isaaclab.sh -p` 経由**で実行する |
| tmux セッションが即終了する | スクリプトファイルに書き出してから `tmux new-session -d -s <名前> <script>` で起動する（ネストしたクォートは壊れる） |
| SSH セッションが突然切れる | `pkill -f run_gr00t_server` が**自分のコマンドラインに自己マッチ**して親シェルを殺している。`pkill -f "run_gr00t_serve[r]"` と書くか、事前に `ps` で確認して実行しない |
| `rl-games` の psutil 依存エラー | rsl_rl を使う限り無害。無視してよい |
| GPU競合 | 学習前に GR00T サーバーを停止（[[一覧/GR00T]] の鉄則） |

---

## 次にできること

| やりたいこと | タスク名 / 方法 |
|---|---|
| ~~凹凸・階段を歩かせる~~ | ✅ **2026-08-30 完了**（上記 Rough 節） |
| 別のヒューマノイド | `Isaac-Velocity-Flat-G1-v0` / `Isaac-Velocity-Rough-G1-v0`（Unitree G1）、`digit`、`cassie` も同梱 |
| 四足歩行 | `anymal_b/c/d`、`go1`、`go2`、`spot`、`a1` |
| 学習曲線をグラフで見る | `logs/rsl_rl/h1_flat/` または `h1_rough/` を TensorBoard で開く |
| Rough の学習過程を動画化 | `h1_rough` の `model_150 / 450 / 600 / 1499.pt` を個別録画（地形レベルの降格→上昇が映像で追える） |

利用可能なロコモーション環境の一覧：

```bash
ls /home/saito/IsaacLab/source/isaaclab_tasks/isaaclab_tasks/manager_based/locomotion/velocity/config/
# a1 anymal_b anymal_c anymal_d cassie digit g1 go1 go2 h1 spot
```

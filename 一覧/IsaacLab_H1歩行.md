---
tags: [IsaacLab, IsaacSim, 強化学習, ヒューマノイド, H1, DGX-Spark, PPO]
created: 2026-08-29
---

# Isaac Lab ヒューマノイド歩行学習（H1）作業ガイド

DGX Spark 上で Unitree H1 を強化学習させ、**シミュレーション内で歩行を獲得**するまでの記録。
関連：[[一覧/GR00T]]（SO-101 模倣学習側）／[[Clippings/第14章　Isaac SimによるReal-to-Simシミュレーション]]

---

## 結論（2026-08-29 達成）

| 項目 | 結果 |
|---|---|
| 学習時間 | **約20分**（1000イテレーション） |
| スループット | **59,000〜69,000 steps/s** |
| Mean reward | **−6.47 → +34.0** |
| 速度追従誤差 (xy) | 0.396 → **0.180** |
| 実質収束 | **イテレーション350付近**（1000は過剰） |

歩行動画：
- **学習過程（40秒）**：![[その他/h1_学習過程_20260829.mp4]] ← 転倒→反転→歩容獲得を4段階で
- 30秒版（36体）：![[その他/h1_walk_30sec_20260829.mp4]]
- 10秒版（16体）：![[その他/h1_walk_20260829.mp4]]

いずれも H.264 / 1280x720 / 50fps。

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
| 凹凸・階段を歩かせる | `Isaac-Velocity-Rough-H1-v0`（4096環境 / 1500 iter、地形カリキュラム＋height_scan付き） |
| 別のヒューマノイド | `Isaac-Velocity-Flat-G1-v0`（Unitree G1）、`digit`、`cassie` も同梱 |
| 四足歩行 | `anymal_b/c/d`、`go1`、`go2`、`spot`、`a1` |
| 学習曲線をグラフで見る | `logs/rsl_rl/h1_flat/<timestamp>/` を TensorBoard で開く |

利用可能なロコモーション環境の一覧：

```bash
ls /home/saito/IsaacLab/source/isaaclab_tasks/isaaclab_tasks/manager_based/locomotion/velocity/config/
# a1 anymal_b anymal_c anymal_d cassie digit g1 go1 go2 h1 spot
```

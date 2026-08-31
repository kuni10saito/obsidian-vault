---
tags: [IsaacLab, SO-101, 強化学習, sim-to-real, 実機化, DGX-Spark, PPO, ONNX]
created: 2026-08-31
---

# Isaac Lab → SO-101 実機化ガイド

**目的**：Isaac Lab で学習した強化学習ポリシーを、**手持ちの SO-101 実機で動かす**。
Go2 EDU（$11,190〜）を買う前に、所有済みの SO-101 で sim-to-real パイプラインを一周する。

関連：[[一覧/IsaacLab_H1歩行]]（H1/Go2の歩行学習）／[[一覧/GR00T]]（SO-101 模倣学習側）

---

## なぜ SO-101 から始めるのか

歩行ポリシーの実機化で最大の壁だった項目が、腕ではほぼ消える。

| 壁 | 四足歩行（Go2） | **SO-101（腕）** |
|---|---|---|
| `base_lin_vel` の推定 | **最大の難関**（測るセンサーが無い） | **観測に含まれない** |
| 転倒リスク | 高い（吊り下げ治具が必要） | 無い |
| 失敗のコスト | 機体破損（$11,000〜） | 動きがおかしいだけ |
| 制御方式 | トルク制御 | **位置制御**（サーボ目標値に直結） |
| 低レベルSDK | **Go2 EDU が必須** | 既存の `run_groot.py` がそのまま使える |

---

## 環境（H1/Go2 とは別物なので注意）

`isaac_so_arm101` は **uv で独自に Isaac Sim を pip 導入**しており、`~/IsaacLab`（ソース版）とは完全に別環境。

| | `~/IsaacLab`（H1/Go2） | `isaac_so_arm101`（SO-101） |
|---|---|---|
| Isaac Sim | ソースビルド版 5.1.0 | **pip版** 5.1.0 |
| torch | 2.9.0+**cu130** | 2.7.0+**cu128**（`isaacsim` が厳密固定） |
| nvrtc | CUDA 13（sm_121対応） | CUDA 12.8（**sm_121 非対応**） |
| 実行方法 | `./isaaclab.sh -p <script>` | `.venv/bin/python <script>` |

---

## ⚠️ 踏んだ罠3つ（連鎖する）

### ① EULA プロンプトで無言停止

pip版 Isaac Sim は初回起動時に使用許諾の同意を対話で求める。
ヘッドレスでは入力できず、**エラーも出さずに停止したまま**になる。

```bash
export OMNI_KIT_ACCEPT_EULA=YES
```

### ② `nvrtc: invalid value for --gpu-architecture`

```
RuntimeError: nvrtc: error: invalid value for --gpu-architecture (-arch)
  at .../reach/mdp/rewards.py, position_command_error_tanh
```

GB10 は compute capability **(12, 1) = sm_121**。torch cu128 の `get_arch_list()` は
`['sm_90','sm_100','sm_120']` で sm_121 を含まない。

- **行列積などは動く** → 事前コンパイル済み sm_120 カーネルが流用されるため
- **報酬関数で落ちる** → TorchScript が**実行時に nvrtc でカーネルを生成**し、sm_121 を渡して拒否される

### ③ `PYTORCH_JIT=0` は学習を通すが **export を壊す**

②の回避に `PYTORCH_JIT=0` を使うと学習は完走するが、`play.py` が落ちる：

```
AttributeError: '_TorchPolicyExporter' object has no attribute 'save'
```

`torch.jit.trace()` が無効化され素のモジュールを返すため、`.save()` が存在しない。
**実機デプロイに必要な ONNX/JIT 書き出しがまさに壊れる**ので、この回避策は使えない。

### ✅ 正解：融合(fuser)だけ無効化し、`jit.trace` は生かす

nvrtc を呼んでいるのは JIT の**カーネル融合**だけ。融合のみ切ればよい。

`/home/saito/nofuse_launch.py`：

```python
"""GB10(sm_121)対策: JIT融合(nvrtc)だけ無効化し、jit.trace は生かす"""
import sys, runpy, torch

def _try(fn, *a):
    try:
        fn(*a); return True
    except Exception:
        return False

_try(torch._C._jit_override_can_fuse_on_gpu, False)
_try(torch._C._jit_override_can_fuse_on_cpu, False)
_try(torch._C._jit_set_texpr_fuser_enabled, False)
_try(torch._C._jit_set_nvfuser_enabled, False)
_try(torch._C._jit_set_profiling_executor, False)
_try(torch._C._jit_set_profiling_mode, False)

script = sys.argv[1]
sys.argv = sys.argv[1:]
runpy.run_path(script, run_name="__main__")
```

使い方：`.venv/bin/python /home/saito/nofuse_launch.py <本来のスクリプト> <引数...>`

> **torch を cu130 に上げる案は不可**。`isaacsim` が `torch==2.7.0` を厳密固定しているため、
> アップグレードするとパッケージ依存が壊れる。

---

## 学習の実行

スクリプト：`/home/saito/run_so101_reach.sh`

```bash
cd /home/saito/isaac_so_arm101
export OMNI_KIT_ACCEPT_EULA=YES
export LD_PRELOAD="$LD_PRELOAD:/lib/aarch64-linux-gnu/libgomp.so.1"
.venv/bin/python /home/saito/nofuse_launch.py \
  src/isaac_so_arm101/scripts/rsl_rl/train.py \
  --task Isaac-SO-ARM101-Reach-v0 \
  --headless --num_envs 4096 --max_iterations 400 --seed 42 \
  --export_io_descriptors
```

- **`--export_io_descriptors` は必須**。実機で観測を組むための定義が出力される
- 利用可能タスク：`Isaac-SO-ARM101-Reach-v0` / `Isaac-SO-ARM101-Lift-Cube-v0`（+ `-Play-v0`）
- 出力先：`logs/rsl_rl/reach/<timestamp>/`
- スループット **125,670 steps/s**（4096環境）、1000イテレーションで約7分

### ⚠️ 学習曲線：200〜300でピーク、その後**劣化する**

| iteration | reward | 手先位置誤差 | 姿勢誤差 |
|---|---|---|---|
| 0 | −0.04 | 29.7cm | 0.176 |
| 100 | 0.55 | 3.04cm | 2.207 |
| **200** | 0.67 | **2.33cm** ← 最良 | 2.617 |
| **300** | **0.69** | 2.46cm | 2.607 |
| 500 | 0.69 | 2.84cm | 2.546 |
| **800** | **0.52** | **4.13cm** ← 約2倍に悪化 | 2.538 |
| 999 | 0.56 | 3.65cm | 2.524 |

**原因は報酬の重み配分**：

```python
end_effector_position_tracking              weight = -0.2   # 位置
end_effector_position_tracking_fine_grained weight = +0.1   # 位置（微調整）
end_effector_orientation_tracking           weight = -0.1   # 姿勢（位置の1/3）
```

位置の重みが姿勢の3倍のため、学習が進むほど「姿勢を捨てて位置を詰める」方向に寄り、
姿勢誤差が 0.176 → 2.5 に張り付く。最終的には位置精度まで巻き添えで悪化する。

> **結論：`model_999.pt` ではなく `model_300.pt` を使うこと。**
> 次回から `--max_iterations 400` で十分。
> 姿勢も詰めたい場合は `end_effector_orientation_tracking` の重みを −0.2 以上に上げて再学習する。

---

## ONNX 書き出しと可視化

スクリプト：`/home/saito/run_so101_play.sh`

```bash
.venv/bin/python /home/saito/nofuse_launch.py \
  src/isaac_so_arm101/scripts/rsl_rl/play.py \
  --task Isaac-SO-ARM101-Reach-Play-v0 \
  --headless --num_envs 16 --video --video_length 900 \
  --checkpoint <run_dir>/model_300.pt
```

出力：
- `<run_dir>/exported/policy.onnx`（**25.8 KB**）
- `<run_dir>/exported/policy.pt`（TorchScript、35.4 KB）
- `<run_dir>/videos/play/rl-video-step-0.mp4`

動画：![[その他/so101_reach_20260831.mp4]]

---

## 🔧 実機デプロイ仕様（ここが本題）

### ポリシー入出力

```
INPUT :  obs      [1, 25]  float32
OUTPUT:  actions  [1, 6]   float32
```

ネットワークは MLP：`Linear(25→64) → ELU → Linear(64→64) → ELU → Linear(64→6)`

### 観測ベクトル 25次元の内訳

| 順序 | 項目 | 次元 | 実機での取得元 |
|---|---|---|---|
| 1 | `joint_pos_rel` | 6 | サーボのエンコーダ（デフォルト姿勢からの相対値） |
| 2 | `joint_vel_rel` | 6 | エンコーダ値の差分 |
| 3 | `generated_commands` | 7 | **こちらが指定する目標姿勢**（位置3 + クォータニオン4） |
| 4 | `last_action` | 6 | 前回のポリシー出力（自分で保持） |
| | **合計** | **25** | |

> **カメラ不要・状態推定器不要。** すべてサーボから読める値と、自分で決める目標値だけ。
> Go2 で最大の難関だった `base_lin_vel`（測るセンサーが存在しない）に相当するものが無い。

### 行動の適用

```yaml
full_path:   isaaclab.envs.mdp.actions.joint_actions.JointPositionAction
joint_names: [shoulder_pan, shoulder_lift, elbow_flex, wrist_flex, wrist_roll, gripper]
offset:      [0.0, 0.0, -0.0, 1.5700000524520874, -0.0, 0.0]
scale:       0.5
```

**関節目標角 = action × 0.5 + offset**（単位：ラジアン）

- `wrist_flex` にだけ **1.57（= π/2）** のオフセット → 実機のゼロ点定義との突き合わせが必要
- **制御周期 30 Hz**（`decimation = 2` × `sim.dt = 1/60`）
- 参考PDゲイン：`default_joint_damping = [80, 65, 45, 30, 20, 20]`

### 関節の並び順は CLAUDE.md と一致

| 順 | IO_descriptors | CLAUDE.md HOME_POSITION |
|---|---|---|
| 1 | shoulder_pan | M1 shoulder_pan = 1505 |
| 2 | shoulder_lift | M2 shoulder_lift = 2242 |
| 3 | elbow_flex | M3 elbow_flex = 1400 |
| 4 | wrist_flex | M4 wrist_flex = 2375 |
| 5 | wrist_roll | M5 wrist_roll = 2438 |
| 6 | gripper | M6 gripper = 3100 |

---

## 次のステップ：実機クライアントの実装

`Desktop/run_groot.py` が既に **SO-101 の関節状態を読み、目標値を書き込む**構造になっている。
GR00T サーバーへの問い合わせを、ローカルの `policy.onnx` 推論に差し替えるのが最短。

| 段階 | 作業 |
|---|---|
| 1 | `policy.onnx` を FMV に転送、`onnxruntime` で推論できることを確認 |
| 2 | サーボ値（0–4095）↔ ラジアンの変換式を確定。**`wrist_flex` の π/2 オフセット**に注意 |
| 3 | 25次元の観測ベクトルを**学習時と同じ順序・スケール**で組む |
| 4 | 30Hz ループで推論 → `action × 0.5 + offset` → サーボへ書き込み |
| 5 | **まず目標姿勢を現在位置の近傍に置いて**小さく動かし、暴走しないことを確認 |
| 6 | 徐々に目標を遠ざけ、シムとの挙動差を評価 |

> **鉄則**：[[【まとめ】いまの作業の意味づけと次の工程]] で6週間かけて体得した
> 「**収集環境と評価環境を揃える**」が、ここでは「**学習環境と実機環境を揃える**」になる。
> 観測の順序・スケール・制御周期のどれか一つでもズレると、モデルは正常でも動作は破綻する。

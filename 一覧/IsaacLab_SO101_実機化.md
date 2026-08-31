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

## 実機クライアント（実装済み）

`C:\Users\saito\Desktop\so101_rl\`

| ファイル | 内容 |
|---|---|
| `so101_rl_deploy.py` | 実機クライアント本体 |
| `policy.onnx` | 学習済みポリシー（model_300 由来、25.8KB） |
| `IO_descriptors.yaml` | 観測・行動の定義 |

**安全のため既定では動かさない。3段階で確認する。**

```bash
cd C:\Users\saito\Desktop\so101_rl

# 1) サーボを読むだけ。HOME姿勢なら rel が全て ≒0 になるのが正常
python so101_rl_deploy.py --probe

# 2) 推論まで実行し、送るはずの目標値を表示（送信しない）
python so101_rl_deploy.py --dry-run

# 3) 実際に動かす。1周期あたり最大20カウント(≒0.92rad/s)に制限
python so101_rl_deploy.py --run --max-step 20
```

### オフライン検証で確認済み

| 項目 | 結果 |
|---|---|
| ONNX推論速度 | **0.016 ms/回**（30Hzの予算33.3msの0.05%） |
| HOME → `joint_pos_rel` | 全て 0（自己整合） |
| サーボ⇔ラジアン往復変換 | 完全一致 |
| 分解能 | 651.9 counts/rad（1カウント = 0.0879度） |
| 分布外ガード | 範囲外の目標で `--run` を自動中止 |

---

## ⚠️ 実機投入前に必ず理解すべき2つの落とし穴

### ① 目標姿勢が学習分布を外れると、可動端に叩きつけられる

学習時の目標コマンド範囲（`reach_env_cfg.py`）：

```python
pos_x = (-0.1, 0.1)
pos_y = (-0.25, -0.1)   # ← 常に負
pos_z = (0.1, 0.3)
roll = pitch = yaw = 0  # 姿勢は固定 → quat は常に (1,0,0,0)
```

**実測**：`x=0.25, y=0.0`（範囲外）を与えたところ、**全ステップで可動端に飽和**し、
`wrist_roll` が 649 ⇄ 4290（振り幅3641カウント≒320度）を30Hzで往復した。
実機なら確実に破損する。スクリプトに検査を入れてあるが、**範囲は必ず意識すること**。

> 学習時 `roll/pitch/yaw` が全て0固定だったことは、
> 訓練中に `orientation_error` が 2.5 のまま下がらなかった理由でもある。
> 5自由度の腕で「任意位置＋固定姿勢」は幾何的に両立しない。

### ② 関節の追従が速すぎると発散する（オフライン検証の教訓）

最初の検証で「指令した瞬間に関節がそこへ移動する」と仮定したところ、
関節速度が **167 rad/s** という非現実的な値になり、観測が分布外になって暴走した。
**これは検証ハーネス側のバグ**であり、ポリシーの欠陥ではない
（Isaac Lab の play では正常に動作している）。

一次遅れ（追従係数α）を入れて再検証した結果：

| α | クランプ | 最大関節速度 | 後半20stepの振れ幅 | 判定 |
|---|---|---|---|---|
| 1.00（瞬時） | 60/60 | 167 rad/s | 3641 | 発散 |
| 0.30 | 47/60 | 29 rad/s | 2181 | 不安定 |
| **0.15** | 32/60 | 9.2 rad/s | **144** | **安定** |
| **0.08** | 23/60 | 3.0 rad/s | **17** | **ほぼ静止** |

収束先 servo = `[2720, 2076, 1421, 2343, 2049, 2987]`（全て MOTOR_LIMITS 内）。

> **結論：`--max-step` で関節速度を制限すれば安全に動く。**
> シム側は stiffness [200,170,120,80,50,60] / damping [80,65,45,30,20,20] の
> PD制御で追従に遅れがある。実機でもこの「鈍さ」を再現する必要がある。

---

## ⚠️ 未確定：サーボ値とラジアンの対応

**現在の仮定**：実機の HOME 姿勢 = シムの `default_joint_pos`

```python
servo = SERVO_AT_DEFAULT[j] + SIGN[j] * (angle - DEFAULT_JOINT_POS[j]) * 651.9
SERVO_AT_DEFAULT = HOME_POSITION = {1:1505, 2:2242, 3:1400, 4:2375, 5:2438, 6:3100}
SIGN = {1:+1, 2:+1, 3:+1, 4:+1, 5:+1, 6:+1}   # ← 未検証
```

この仮定は**自己整合的だが、物理的な正しさは未確認**。`--probe` で実機と突き合わせて確定させる。

- `Desktop/follower_calibration.json` は**古く、現状と矛盾**している
  （`shoulder_lift` の可動域が 3311〜3812 = 501カウント≒44度しかなく、
  `run_groot.py` の `MOTOR_LIMITS`(2050, 4095) と整合しない）。**参照しないこと**
- `SIGN` は1関節ずつ手で動かして `--probe` の値の増減方向を見て確定する

### シムと実機で可動域が食い違う関節

| 関節 | シム範囲→サーボ換算 | MOTOR_LIMITS | 重なり |
|---|---|---|---|
| shoulder_pan | [253, 2757] | (1600, 3050) | 1600–2757 |
| shoulder_lift | [1104, 3380] | (2050, 4095) | 2050–3380 |
| elbow_flex | [298, 2502] | (100, 2100) | 298–2100 |
| wrist_flex | [271, 2432] | (100, 2350) | 271–2350 |
| wrist_roll | [649, 4290] | (100, 3500) | 649–3500 |
| **gripper** | **[2986, 4238]** | **(2000, 3350)** | **2986–3350（シムの約3割のみ）** |

---

## 🔬 実機検証の結果（2026-08-31 実施）

### 接続構成（重要：FMVではなく Spark に繋がっている）

| 項目 | 値 |
|---|---|
| 接続先 | **DGX Spark**（FMVではない） |
| フォロワーアーム | **`/dev/ttyACM1`** |
| もう1本 | `/dev/ttyACM0`（リーダー） |
| 判別方法 | **自動判別不可**。両方とも6モーター応答する |
| カメラ | `/dev/video0`（俯瞰）、`/dev/video2`（側面） |
| 実行環境 | `~/miniconda3/envs/lerobot/bin/python`（onnxruntime 1.23.2 + pyserial 3.5） |
| 権限 | `saito` は `dialout` / `video` グループ所属済み（sudo不要） |

作業ディレクトリは Spark 側の `~/so101_rl/`。

### ✅ 確認できたこと

- 安全トルクON手順（目標=現在位置を書いてから通電）で**飛び上がりゼロ**
- HOME復帰は誤差 1〜17カウント（`elbow_flex` を除く）
- 推論 0.06〜0.09 ms（30Hz予算の0.3%）
- カメラによる遠隔での動作確認・動画記録が可能

動画：
- ![[その他/so101_実機テスト1_20260831.mp4]]（max-step 5、ほぼ動かず）
- ![[その他/so101_実機テスト2_20260831.mp4]]（max-step 20、shoulder_pan が約100度旋回）

---

### 🔴 発見1：`--max-step` が小さいと重力負荷のかかる関節は動かない

`--max-step 5` では **無負荷の `wrist_roll` だけが動き、他は全て静止**した。
5カウントの位置偏差ではP制御の出力が重力＋静止摩擦を超えられない。

| max-step | 動いた関節 |
|---|---|
| 5 | `wrist_roll` のみ |
| 20 | `shoulder_pan`, `wrist_roll`, `gripper` |
| 35 | 同上（`elbow_flex` は依然動かず） |

> **教訓**：不感帯の検証を**無負荷の関節で行うと誤った結論に至る**。
> 実際に一度「不感帯は問題ない」と誤判定した。負荷のかかる関節で検証すること。

サーボ内部設定（実測）：`P=16`（`wrist_roll` のみ32）／不感帯1カウント／
MaxTorque 1000・TorqueLimit 1000（gripper のみ500）／内部角度制限なし（0〜4095）。
**`wrist_roll` だけ P ゲインが倍**だったことも、この関節だけ動いた理由。

---

### 🔴 発見2：`elbow_flex` は展開姿勢で前腕を持ち上げられない

単独テスト（1秒ごとに指令→計測→復帰）：

| 指令Δ | 実移動 | 判定 |
|---|---|---|
| −10 | **+37** | 逆行（重力で落ちる） |
| −50 | ±0 | 動かない |
| **−100** | **−5** | **ほぼ動かない** |
| +10 | **+63** | 行き過ぎ（重力で落ちる） |
| +100 | +172 | 同上 |

**カウント減少方向（＝持ち上げ方向）へは −100 を指令しても動かない。**
増加方向へは指令以上に落下する。機械的干渉はなく、アームは自由空間にあった
（前腕とグリッパーが水平に片持ちで張り出した、elbowのモーメント最大姿勢）。

HOME（折り畳み姿勢）では elbow が 2016→1482 と534カウント動けたので、
**姿勢依存**の問題。展開するとモーメントが増えてサーボ出力を超える。

---

### 🔴 発見3：`MOTOR_LIMITS` と `HOME_POSITION` が矛盾している

`run_groot.py` から引き継いだ2つの定数が互いに整合していない：

| 関節 | HOME | MOTOR_LIMITS | 判定 |
|---|---|---|---|
| **shoulder_pan** | 1505 | (**1600**, 3050) | HOMEが下限を95下回る |
| **wrist_flex** | 2375 | (100, **2350**) | HOMEが上限を25超える |

そのため HOME 姿勢のままでもクランプが常時作動する。
`MOTOR_LIMITS` は**GR00Tの模倣学習データ由来**であり、RLポリシーの可動要求とは別物。

---

## 💡 結論：これが sim-to-real ギャップの実例

シム側のアクチュエータ設定は妥当に作られている：

```python
"arm": ImplicitActuatorCfg(
    effort_limit_sim=1.9,       # STS3215 のストールトルク ≒1.86 N·m (7.4V) 相当
    velocity_limit_sim=1.5,
    stiffness={"shoulder_pan":200, "shoulder_lift":170, "elbow_flex":120,
               "wrist_flex":80, "wrist_roll":50},
    damping={...},
)
```

**それでも実機は動けない。** 考えられる差：

| 要因 | 内容 |
|---|---|
| **ストールトルク ≠ 連続トルク** | 1.9 N·m は瞬間値。連続で出せるのは通常その30〜50% |
| 実機質量 > URDF | ケーブル、3Dプリント充填率など |
| 摩擦・バックラッシュ | シムでモデル化されていない |
| 電圧降下 | 負荷時に電圧が下がりトルクが落ちる |

> **ポリシーは「シムなら出せるトルク」を前提に姿勢を選ぶ。**
> 実機で出ないトルクを要求する姿勢に入ると、そこで止まる。
> Go2 EDU（$11,190〜）でこれを発見していたら、はるかに高くついた。

### 対策の方向

| 案 | 内容 |
|---|---|
| **A. effort_limit を実測値まで下げて再学習** | 連続トルク基準（例 0.6〜0.9 N·m）にすれば、ポリシーは腕を伸ばしきらない姿勢を学ぶ |
| **B. 目標範囲を絞る** | `pos_z` を高め・`pos_y` を手前にして、展開しない範囲だけ狙わせる |
| **C. `MOTOR_LIMITS` を実測で引き直す** | まず前提を正す。A/B いずれにせよ必要 |
| D. 重力補償を入れる | 実機側で姿勢に応じた補正トルクを加算。実装は重い |

**Aが本命**。シムの `effort_limit_sim` を実測の連続トルクに合わせることが、
そのまま Domain Randomization の第一歩になる。

---

## ✅ 対策A実施：`effort_limit_sim` を下げて再学習（成功）

### 必要トルクの見積もり

設定ファイルのコメントより **肘が動かす質量 = 0.38kg**（前腕＋手首＋グリッパー）。
前腕の重心を肘から約0.10mとすると、水平展開時の保持トルクは

```
0.38 kg × 9.81 m/s² × 0.10 m ≒ 0.37 N·m
```

シムの 1.9 N·m なら5倍の余裕があるはずだが実機は動けなかった。
「連続トルクはストールの30〜50%」の経験則から **0.6 N·m** を採用。

### 変更内容

`src/isaac_so_arm101/robots/trs_so101/so_arm101.py`
（バックアップ：`so_arm101.py.bak_20260831`）

```python
"arm":     effort_limit_sim = 1.9 → 0.6
"gripper": effort_limit_sim = 2.5 → 1.0
```

再学習：`/home/saito/run_so101_reach_v2.sh`（400 iterations、約4分）

### 学習曲線の比較

| iteration | v1 (1.9 N·m) 位置誤差 | **v2 (0.6 N·m) 位置誤差** |
|---|---|---|
| 100 | 3.04cm | 5.76cm |
| 200 | **2.33cm** | 6.26cm |
| **250** | 2.84cm | **4.11cm** |
| **300** | 2.46cm | 4.30cm |
| 399 | 2.66cm | 5.38cm |

シム上の精度は落ちる（2.3cm → 4.1cm）が、これは
**実機で本当に出せるトルクの範囲で達成可能な精度**である。

### 🎉 実機での結果：`elbow_flex` が動くようになった

同じ `--max-step 20` での比較：

| 関節 | v1 (1.9 N·m) | **v2 (0.6 N·m)** |
|---|---|---|
| shoulder_pan | ✅ 動作 | ✅ 動作 |
| shoulder_lift | ❌ 下限2050で停止 | ✅ **下限到達後に回復**（→2382） |
| **elbow_flex** | ❌ **完全に静止** | ✅ **動作**（1511→1854→1709） |
| wrist_roll | ✅ 動作 | ✅ 動作 |
| gripper | ✅ 動作 | ✅ 動作 |

動画：![[その他/so101_実機テスト3_v2_20260831.mp4]]

**姿勢の違いが本質。** v1のポリシーは腕を水平に伸ばしきる姿勢を選び、
そこで肘が力尽きた。v2のポリシーは**肘を曲げたまま前腕を立てる**低トルク姿勢を選ぶ。
シムに正しい制約を与えると、ポリシーは自然に実機で実行可能な動きを学ぶ。

> **これが sim-to-real の基本サイクル**：
> 実機で失敗 → 原因を物理量で特定 → シムのモデルを修正 → 再学習 → 実機で検証。
> 今回は「効かない設定を推測で直す」のではなく、
> **実測した物理限界（肘が持ち上がらない）をシムのパラメータに翻訳**した点が重要。

### 残る課題

| 項目 | 内容 |
|---|---|
| `wrist_flex` | 依然 `MOTOR_LIMITS` 上限2350でクランプ（ポリシーは2432を要求） |
| `gripper` | 上限3350でクランプ（ポリシーは3466〜3921を要求） |
| `MOTOR_LIMITS` | GR00T由来のため実測で引き直しが必要（発見3参照） |
| effort_limit の追い込み | 0.6 は経験則。実測すればさらに正確にできる |

---

## 次のステップ（実機が必要）

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

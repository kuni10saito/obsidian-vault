---
tags:
  - OpenUSD
  - Omniverse
  - SimReady
  - CAD
  - エージェント
  - デジタルツイン
created: 2026-09-21
source: https://www.youtube.com/watch?v=zW0wzeAWLvo
channel: NVIDIA Omniverse
published: 2026-09-16
duration: 1:13:22
---

# CADモデルから OpenUSD シーンを作る ― Vertiv と PTC、そして GPT-6 Astra

> [動画](https://www.youtube.com/watch?v=zW0wzeAWLvo)（NVIDIA Omniverse・2026-09-16・1時間13分・8,399回視聴）の要約。
> **自動生成字幕（英語・13,420語）から作成。** 公式の書き起こしではないので、固有名詞の綴りに誤りが混じる可能性がある。

**登壇者**

| 所属     | 名前          | 担当                                     |
| ------ | ----------- | -------------------------------------- |
| NVIDIA | Adam Hughes | データセンター／AIファクトリ（DSX ブループリント）           |
| NVIDIA | Renato      | Isaac チーム。ロボティクスと USD 規格、エージェント型ワークフロー |
| Vertiv | Adash       | 上級ディレクター。シミュレーションとエンジニアリング支援           |
| Vertiv | Hesh        | エンジニアリング部長。デジタルエンジニアリング                |
| PTC    | Steve       | 製品エンジニアリング。USD 全般                      |

---

## 1. 中心にある主張

**CAD ファイルから SimReady な OpenUSD 資産を作る工程を、AI エージェントに任せた。**

```
STEP ファイル
  → HOOPS コンバータ → 素の USD ジオメトリ
  → Blender で見た目を作る（デカール、ロゴ、機器の表記）
  → Isaac の SimReady スキーマを適用
  → 検証。通らなければもう一周
```

この全工程を **Astra が90分で**実行したと述べている。**発表に使った PowerPoint の画像込み**で90分。

> 昔はボタンを押して変換をクリックしていた。いまは Codex に「このファイルを、このテンプレートに向けて変換して」と言えば走る。

なお変換経路は固定ではない。**PTC のコネクタでも自動化できる**し、そちらは追加の処理も入る。今回の STEP ファイルにはこの経路が合っていた、という位置づけ。

---

## 2. なぜ「ただの変換」ではないか

ここが要点。

> **CAD データは見た目の資産を持っていない。** 今回の例は STEP なので、**視覚的な情報もメタデータも何も無い。**

だから3段構えになる。

| 段                | 何をするか                | 誰が                            |
| ---------------- | -------------------- | ----------------------------- |
| ① 変換             | 形状を USD にする          | 従来からある                        |
| ② **材質化**        | 見た目を作る               | **AI。「ついでにやってくれるので、ただで手に入る」** |
| ③ **SimReady 化** | シミュレーションに要るメタデータを付ける | **AI。検証まで回す**                 |

**形だけ移してもシミュレーションには使えない**、というのが全体の動機。

---

## 3. SimReady が可能にすること

Adam の説明がいちばん具体的だった。

> USDC を AI で拡張して、難しい問いを投げられるようになる。**「AIファクトリの中で音響はどうなるか」**と訊けば、AI が音響ソルバを書き、各 SimReady 資産のメタデータを拾って音を伝播させる。夜か昼か、気温は何度か。
>
> CFD でも同じ。HVAC を見るのか、ホットアイルを見るのか、冷却プレートの配管内の流体を見るのか。あるいは工場の外で、砂漠の真ん中で外気が猛烈に暑くて風がこの向きのとき、チラーはどうなるか。

**メタデータが付いているから、後から問いを変えられる。** ここが「資産」と呼ぶ理由。

---

## 4. PTC の主張 ― 管理し続けるのは PLM

Steve（PTC）の力点は別のところにあった。

> **すべてがリリース管理され、関連づけられている。** 部品を1つ変えれば下流に全部流れる。全員が知る。リリースゲートも管理ゲートも通せる。**繰り返し確実に届けられること**が本質。

そして重要な但し書き：

> **CAD に無い情報がたくさんある。** 接続点のようなものは MBD ツールで注記して USD へ流せる。だが多くの属性は CAD にも PLM にも無い。それを集めるのは面白い課題で、**Windchill の MCP サービス**を使えばエージェントが外部から情報をかき集めて PLM 経由で束ねられる。
>
> **鍵は、資産を管理し続けるのは PLM 側だということ。**

Creo からは USD を直接書き出せる（今回のデモは STEP 経由だった）。

---

## 5. 質疑から

### 世界の物体のうち剛体と変形体の割合は？

> **ほぼ100%が変形体である。** だがシミュレーションでは多くの場合その変形を気にしないので、剛体として扱うと決める。衝突試験なら曲がったり割れたりするので FEM のような高度な手法が要る。Nova Carter のような用途ではそういう条件にならないので、**計算しやすいほうを選ぶ。**

> **モデル化とは「何を気にしないか」を決めること**、という当たり前だが忘れやすい話。

### タイヤと床の摩擦や材質はどう扱うか

> 物理マテリアルは**メッシュ単位ではなくサブサーフェス単位**で割り当てられる。片面がゴムで片面がガラスの剛体なら、同じ剛体に別々の摩擦を持たせられる。床にもマテリアルがあり、**2つの接触マテリアルの平均を取るか、小さいほうを取るか、大きいほうを取るか**を選ぶ。

---

## 6. 締め

Vertiv の Adash：

> **一つの道具の力より、協働の力のほうが大きい。** Vertiv・PTC・NVIDIA の3社が組んだことで、SimReady 資産の作成が速く・簡単に・規模を持てるようになった。CAD ファイルが、**軽いけれども完全な OpenUSD** に変換されてデジタルツインに使える。

告知：**GTC Berlin（2026年10月20〜22日）**

---

## 7. 関連する道具

### `NVIDIA-Omniverse/usd-convert-cad`

CAD → OpenUSD の変換ツール。**OpenUSD ランタイムを同梱**しているので外部依存なしで動く。

```bash
python -m pip install usd-convert-cad
usd-convert-cad -i input.step -o output.usdc
```

対応形式は30以上。

| 種別  | 形式                                                         |
| --- | ---------------------------------------------------------- |
| CAD | JT, CATIA V5/V6, NX, Creo, SolidWorks, Inventor, Parasolid |
| AEC | IFC, Revit, DGN, AutoCAD (DWG/DXF)                         |
| 交換  | STEP, IGES, STL, FBX, OBJ, glTF, Rhino                     |

> **Linux aarch64 では AutoCAD と Revit が非対応。** DGX Spark（GB10）で使うならこの2つは落ちる。

`skills/omniverse-cad-to-usd/SKILL.md` は **AI エージェント向けのスキル定義**で、変換オプションの一覧が入っている。エージェントから自律的に呼べるようにする設計。

### PTC の動き

Creo（CAD）と Windchill（PLM）に Omniverse を統合。PTC は **AOUSD（Alliance for OpenUSD）に加盟**。

---

## 8. 琵琶湖プロジェクトとの関係

**入口が違う。**

```
この動画    CAD の「形」   → USD ＋ 意味情報（SimReady）
Kit CAE     CGNS/VTK の「場」 → USD
```

ただし **③ SimReady 化の考え方は共通**である。琵琶湖でいえば、水温の VTK を出すだけでなく、

```
このセルは水深何m / これは観測点（今津沖中央）/ これは未循環域 /
この面は観測限界 / この年度は不成立
```

という**意味情報を USD に持たせる**発想に対応する。そうすれば「未循環面積が何%か」「この年度と比べてどう違うか」を後から問える。

そして `SKILL.md` によるエージェントからの操作は、Kit CAE の **Agent skills** と同じ思想。

**逆に、この動画から持ち込めないもの:**

- `usd-convert-cad` は CAD 形式の変換器で、**VTK も CGNS も読まない**
- SimReady スキーマは機械部品向け（剛体・材質・接続点）で、**場のデータ用ではない**

> 関連: [[琵琶湖/HANDOFF_琵琶湖引き継ぎ1.md]] §42（Kit CAE の再評価）

---

## 出典

- [動画](https://www.youtube.com/watch?v=zW0wzeAWLvo)
- [usd-convert-cad](https://github.com/NVIDIA-Omniverse/usd-convert-cad)
- [PTC × NVIDIA Omniverse](https://www.ptc.com/en/news/2025/ptc-nvidia-omniverse)
- [SimReady Assets for DSX Digital Twins](https://docs.omniverse.nvidia.com/dsx/latest/simready-assets.html)

字幕の原文は `C:\Users\saito\Downloads\yt\transcript.txt`（13,420語）および時刻つき版 `transcript_timed.txt` に保存。

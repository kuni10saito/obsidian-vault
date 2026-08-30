## 4.7 周囲地形の取り込みと地形USD【2026-08-30】

湖底だけを描くと湖が宙に浮いて見える。**周囲の陸地（国土地理院DEM）と
湖底を1枚の標高場に繋いで USD にした。**

### 4.7.1 データ源 —— 国土地理院タイル

| | URL / 値 |
|---|---|
| 標高DEM | `https://cyberjapandata.gsi.go.jp/xyz/dem/{z}/{x}/{y}.txt` z=13 |
| 地図 | `https://cyberjapandata.gsi.go.jp/xyz/{kind}/{z}/{x}/{y}.png` z=13 |
| `kind` | `pale`（淡色・**既定**）/ `std`（標準）/ `seamlessphoto`（航空写真） |
| 範囲 | N35.05〜35.62 / E135.78〜136.42 |
| タイル | 16×17 = 272枚、**実在271枚（欠測2.7%）** |
| 標高レンジ | 0.0 〜 1375.7 m |

範囲は**比良山地（武奈ヶ岳1214m・西）と伊吹山（1377m・北東）を含む**ように取った。

DEMタイルは `.txt` で 256行×256列のカンマ区切り標高[m]。**欠測は `e`。**
湖面は 84.60 前後（琵琶湖の水位）で平らに埋まっている。

取得済みタイルは `terrain/cache/` に残るので、再実行しても再取得しない。

> **【出典表示が義務】** 地理院タイルの利用規約により、成果物には
> **「国土地理院タイルを加工して作成」**等を明記すること。
> https://maps.gsi.go.jp/development/ichiran.html

### 4.7.2 標高の合成

| 領域 | 標高 |
|---|---|
| 陸 | DEM の標高をそのまま [m] |
| 湖 | **湖面標高 84.4m から水深を引く**（`biwa_bathymetry.npz`） |

DEM は湖面を平らな面として持っているので、そこを**湖岸線 mask で湖底に
置き換える**。琵琶湖の基準水位は 84.371m（B.S.L.±0）、ここでは 84.4m を採用。

DEM の欠測（海・範囲外にわずかに出る）は scipy の
`distance_transform_edt` で最近傍から埋めている。

### 4.7.3 座標 —— 既存USDと同じ原点

原点は**湖底図と同じ** lat0,lon0 = **35.2806N, 136.0710E**（湖底図 mask の重心）。

```
x = (lon - lon0) * 111320 * cos(lat0)   東向き [m]
y = (lat - lat0) * 111320               北向き [m]
z = 標高 [m] * ve                        鉛直誇張は焼き込む
```

### 4.7.4 湖面（半透明の別メッシュ）

地形メッシュは湖底まで凹んでいるので、その上に平らな水面 `z = 84.4 * ve` を張る。

- プリムは **`/Biwa/WaterSurface`**（別プリムなので不要なら Outliner で消せる）
- **4隅すべてが湖内の四角形だけ**を面にする。**§4.5 の罠③「湖底を構造格子で
  書くと湖外まで描かれる」と同じ対処**である
- `UsdPreviewSurface`：diffuseColor (0.13, 0.36, 0.55) / opacity 0.55 /
  roughness 0.08 / metallic 0、`doubleSided`、`subdivisionScheme none`
- `--no-water` で省略、`--water-alpha` で調整

### 4.7.5 生成物

| ファイル | stride | ve | 地形頂点 | 地形面 | 湖面頂点 | サイズ |
|---|---:|---:|---:|---:|---:|---:|
| `biwa_terrain.usdc` | 4 | 3 | 1,114,112 | 1,112,001 | — | 22.3 MB |
| **`biwa_terrain_mid.usdc`** | **2** | **3** | **4,456,448** | **4,452,225** | **733,087** | **98.1 MB** |
| `biwa_terrain_hi.usdc` | 1 | 5 | 17,825,792 | 17,817,345 | 2,932,630 | 392.3 MB |

中間データ

| ファイル | サイズ | 内容 |
|---|---|---|
| `terrain/dem.npz` | 44.1 MB | `elev[H,W]` 標高[m]（欠測NaN）、`lat`/`lon`、bbox、z |
| `terrain/map.png` | 6.7 MB | 地図テクスチャ 4096×4352 |

- 水平 **東西 63.9 km × 南北 67.8 km**、標高 −20〜1376 m、湖内格子 **16.5%**
- **`map.png` は USD と同じフォルダに必須。** 移動するときは一緒に運ぶ

### 4.7.6 鉛直誇張は 2〜3 倍

`fig_terrain_ve.png`（Vault `biwa/`）に ve=2/3/4/5 を並べた。

| ve | 見え方 |
|---|---|
| 2〜3 | **自然な尾根**。既定は 3 |
| 4 | 尖り始める |
| 5 | **針山。地形として破綻**している |

山1377m・湖底−104m で起伏1481m、水平58kmに対して 1:39。
**`biwa_terrain_hi.usdc` だけ ve=5 で作ってあり、単体では不自然。**
使うなら `--ve 3` で作り直すこと。

### 4.7.7 【訂正】stride 1 は440万頂点ではない

作業中に「`--stride 1` で440万頂点」と述べたが**誤り**。
実際は **1,780万頂点**（DEM全解像度 4096×4352）で、**4倍の見誤り**だった。
440万は `--stride 2` のほう。

### 4.7.8 【制約1】南湖の南端が DEM 範囲で切れている

**文書化にあたって実測して見つけた、未修正の欠陥である。**

| | 緯度範囲 |
|---|---|
| 湖底図 mask | 34.9595 〜 35.5162 N |
| DEM | **35.0301** 〜 35.6394 N |

`fetch_terrain.py` の `LAT0 = 35.05` が南湖の南端を切っており、
**湖セルの 3.19%（1053 / 32990）が地形USDから欠落**している。
湖面メッシュの南端 y=−27866 は湖岸線ではなく**DEM格子の縁**である。

北湖の全層循環には影響しないが、**湖全体を描くと南端が直線で切れる。**
直すなら `LAT0` を 34.94 に下げて `fetch_terrain.py` を再実行
（キャッシュがあるので追加行だけ取得される）。

### 4.7.9 【制約2】粒子USDとは鉛直方向に重ならない

**水平は一致する**（原点が同じ）。実測：

| USD | プリム | z レンジ | ve | 湖面の z |
|---|---|---|---:|---:|
| `biwa_terrain_mid.usdc` | `/Biwa/Terrain` | −59.0 〜 4127.0 | 3 | — |
| 〃 | `/Biwa/WaterSurface` | 253.2（平ら） | 3 | **+253.2** |
| `biwa_convection.usdc` | `/LakeBiwa/LakeBottom` | −20809.7 〜 −0.9 | 200 | **0** |
| 〃 | `/LakeBiwa/Convection` | −20429.6 〜 −1.8 | 200 | 0 |

**基準面が違う。** 地形は湖面を `84.4 * ve` に置き、粒子は湖面を `0` に置いて
下向きに `-depth * ve` を取る。重ねるには

1. **ve を揃える**（`make_convection.py --ve` と `make_terrain_usd.py --ve`。
   既定は 200 と 3 で**66倍違う**）
2. さらに**粒子を `+84.4 * ve` だけ持ち上げる**

の2つが必要。`make_terrain_usd.py` の docstring は「既存の粒子USDとそのまま
重なる」と書いているが、**それは水平だけ**である。

### 4.7.10 実行例

```bash
cd C:\Users\saito\Downloads\NVIDIAmodulus_scripts\sym
python fetch_terrain.py                          # DEM＋地図（キャッシュ有効）
python fetch_terrain.py --dem-only
python make_terrain_usd.py --stride 2 --ve 3     # 推奨
python make_terrain_usd.py --stride 1 --ve 3 --out biwa_terrain_hi.usdc
python make_terrain_usd.py --no-water            # 湖面なし
python make_terrain_usd.py --water-alpha 0.3     # もっと透ける
```

### 4.7.11 追加ファイル

| ファイル | 役割 |
|---|---|
| `fetch_terrain.py` | 地理院タイル → `terrain/dem.npz` / `terrain/map.png` |
| `make_terrain_usd.py` | DEM＋湖底 → 地形USD（湖面・UV・テクスチャ付き） |
| `_terrain_check.py` | Blender でUSDを読み、プリム構成を出して静止画を撮る |

---


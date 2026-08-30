### 5.8 地理院タイルは種類によって z=13 が存在しない

地図テクスチャを `--map-kind seamlessphoto`（航空写真）で取ろうとしたところ、
**272枚すべて 404** で `実在 0 / 272` となり、**0.1 MB の空画像**ができた。
それでも `map.png` は生成されるので、**ログの「実在 N / M」を見ないと気づけない。**

既定の `pale`（淡色地図）は z=13 で全枚取れる（6.7 MB）。
`seamlessphoto` を使いたいなら z を上げる必要がある。

### 5.9 1,780万頂点を Python ループで組んではいけない

`--stride 1` の 1,780万頂点・1,781万面を素朴な Python ループで
`Vt.Vec3fArray` に詰めると数分かかる。**`Vt.Vec3fArray.FromNumpy()` /
`Vt.IntArray.FromNumpy()` を使うこと。**
`make_terrain_usd.py` は頂点・面・UV・湖面すべて numpy で組んでから一括で渡す。

なお **`Vt.IntArray` は numpy の int64 を受け付けない**（§5.4）。
`FromNumpy` に渡す前に `np.int32` へ落とすこと。


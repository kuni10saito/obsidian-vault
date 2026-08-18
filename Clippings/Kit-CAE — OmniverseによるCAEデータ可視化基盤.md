# Kit-CAE の概要

NVIDIA 公式の Omniverse Kit サンプルアプリ。CAE（数値解析）データを **OpenUSD 上で変換せずに扱い、GPU で可視化する**リファレンス実装＋開発者ツールキット。
リポジトリ：https://github.com/NVIDIA-Omniverse/kit-cae

## 1. 中核となる考え方
- 従来は CFD/FEM のソルバ出力を可視化ツール用フォーマットへ**変換・コピー**する必要があった
- Kit-CAE は USD の File-Format Plugin として解析データを直接読み込み、**遅延ロード（lazy access）で USD ステージに合成**する
- ソルバ出力・AI サロゲート・センサーデータがそのまま「真実の情報源（source of truth）」として機能する

## 2. 技術要素

| 要素                                   | 役割                                             |
| ------------------------------------ | ---------------------------------------------- |
| OpenUSD Schemas / File-Format Plugin | CGNS・EnSight・VTK・OpenFOAM をネイティブ読み込み（独自形式も拡張可） |
| NVIDIA Warp                          | GPU 加速の可視化アルゴリズム。大規模データを対話的に探索                 |
| RTX / IndeX                          | サーフェス・ボリューム・パーティクルの高品質レンダリング                   |
| Kit Application Framework            | UI 拡張、ピクセルストリーミング、Omniverse ライブラリ群との統合         |

## 3. 主な機能
- 大規模データセットの対話的探索
- GPU 加速の可視化オペレータ
- **Agent skills**：エージェントから可視化・キャプチャ・ストリーミングをプログラム制御
- ピクセルストリーミング（リモート配信）

## 4. ビルドと起動

```bash
./repo.sh build -r          # Linux
./repo.sh launch -n omni.cae.kit
```

## 5. ライセンス
- NVIDIA Omniverse License Agreement に準拠が必要
- 同梱 OSS（CGNS, HDF5, VTK, Zlib, h5py 等）のライセンスは `tpl_licenses/` に個別記載

## 6. 自分のワークフローとの関係
- **対象領域が違う**：Kit-CAE は解析データの可視化。Isaac Sim / Isaac Lab はロボットの物理シミュレーション
- **土台は共通**：どちらも Omniverse Kit 上のアプリ。拡張機能の作り方や `.kit` ファイルによる構成定義は同じ流儀なので、片方を覚えればもう片方に流用が効く
- Isaac Sim 側の理解は [[Clippings/第14章　Isaac SimによるReal-to-Simシミュレーション.md]]、[[Clippings/第15章　Isaac SimとNVIDIA Cosmosの統合関係.md]] を参照
- シミュレーションからのデータ収集〜学習の流れは [[Clippings/「LeIsaac」入門：シミュレーションでのデータ収集からポリシー学習まで.md]]

## 参考リンク
- [GitHub: NVIDIA-Omniverse/kit-cae](https://github.com/NVIDIA-Omniverse/kit-cae)
- [Kit-CAE User Guide](https://docs.omniverse.nvidia.com/guide-kit-cae/latest/index.html)
- [Launch ドキュメント](https://github.com/NVIDIA-Omniverse/kit-cae/blob/main/docs/Launch.md)

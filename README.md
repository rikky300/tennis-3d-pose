# tennis-3d-pose

単眼のテニス動画から、選手の 3D 骨格（26 関節）を推定する拡散モデル。

- **デモ**: https://tennis-3d-pose-308871972947.asia-northeast1.run.app （1 人 1 日 3 回まで）
- コードは非公開（紹介用のリポジトリ）

![demo](docs/demo.png)

## 背景
- テニスを始めて、プロの体の細かい動きや自分のフォームを 3D で見たくなった
- 大規模モデル（SAM 3D Body）は高精度だが重い（1 コマ数秒）
- テニスの 3D の正解データは公開されていない。速い動き・体で隠れる関節が多い
- → 軽い 2D → 3D の拡散モデルを、公開モーションで事前学習し、大規模モデルの出力で蒸留する

## 参考文献・モデル・データ
- **手法**
  - [D3DP](https://arxiv.org/abs/2303.11579)（Shan+, ICCV 2023）— 2D 条件の拡散モデル、複数候補の集約
  - [MixSTE](https://arxiv.org/abs/2203.00859)（Zhang+, CVPR 2022）— 空間・時間の注意を交互に掛けるネットワーク
  - [DDPM](https://arxiv.org/abs/2006.11239)（Ho+, 2020）/ [DDIM](https://arxiv.org/abs/2010.02502)（Song+, 2021）/ [Improved DDPM](https://arxiv.org/abs/2102.09672)（Nichol & Dhariwal, 2021）
  - [知識蒸留](https://arxiv.org/abs/1503.02531)（Hinton+, 2015）
  - [One Euro filter](https://gery.casiez.net/1euro/)（Casiez+, CHI 2012）— 時間方向のならし
- **モデル**
  - [SAM 3D Body](https://github.com/facebookresearch/sam-3d-body)（Meta, 2025）— 蒸留の先生
  - [ViTPose](https://arxiv.org/abs/2204.12484)（Xu+, NeurIPS 2022）— 2D 骨格
  - [RT-DETR](https://arxiv.org/abs/2304.08069)（Zhao+, CVPR 2024）— 人の検出
- **データ**
  - [CMU Motion Capture](http://mocap.cs.cmu.edu/)・[AIST++](https://google.github.io/aistplusplus_dataset/)（[Li+, ICCV 2021](https://arxiv.org/abs/2101.08779)）— 事前学習
  - [Wikimedia Commons のテニス動画](https://commons.wikimedia.org/wiki/Category:Videos_of_tennis)（CC BY-SA 3.0）— 蒸留

## 手法
```
動画 → 人の検出・追跡（RT-DETR）→ 2D 骨格 17 点（ViTPose）→ 拡散モデル → 3D 骨格 26 点 → 時間方向にならす → 骨の長さをそろえる
```
- MixSTE 型（約 28 万パラメータ）、81 コマの窓、x0 予測
- 生成は DDIM 10 ステップ × 候補 5 本の平均
- ガタつきは One Euro filter を前向き・後ろ向きにかけて平均（遅れなし）。ゆっくりな動きは強く、速いスイングは弱くならす（デモの動画で加速度の平均 12.8 → 5.4 m/s²）
- 見えない関節は「空」のトークンとして入力し、損失に入れない（親の関節で埋めると足が崩れたため）

## 学習
1. **事前学習**: CMU・AIST++ の 3D を仮想カメラで 2D に写し、欠け・ずれ・外れ値を加えて学習
2. **蒸留**: テニス動画に SAM 3D Body を掛けた 3D を正解と仮定し、同じ映像の 2D から再現するよう学習
- 2D と食い違うコマは除外（Commons: 10,057 コマ中 8,293 コマを使用）
- 修正したバグ: 切り抜き映像に元の画角を渡しており、奥行きが関節あたり約 5 cm ゆがんでいた

## 結果
学習に使っていないプロ選手の 4 本で、SAM 3D Body の 3D との誤差。

| モデル | 学習データ | MPJPE | PA-MPJPE |
|---|---|---:|---:|
| 事前学習のみ | CMU ＋ AIST++ | 110 mm | 77 mm |
| ＋ 蒸留（公開中） | ＋ Commons のテニス動画 10 本 | 107 mm | 73 mm |
| ＋ 蒸留（研究用） | ＋ プロ選手 7 人の練習動画 | **84 mm** | **60 mm** |

- 公開データだけでは改善が小さい（量・画質が不足）。ただし足先・かかとまで 26 関節を出せるようになった
- プロの動画では誤差が約 24% 下がる。YouTube 由来のため公開サービスには使っていない
- 課題: 足・手の誤差が大きい／自分で撮ったテニス動画を増やす

## 構成
![system](docs/system_diagram.png)

- 学習は Mac（Apple シリコンの GPU）、推論だけ Cloud Run（CPU、使わない時は 0 台）
- W&B: 学習の記録・モデル登録・本番の印・データの来歴
- Hugging Face（非公開）: 重みとデモの結果。重みの更新はコンテナを作り直さずに済む
- 商用で使えるデータだけの重みしか公開できない仕組み
- 守り: 1 日の回数制限（Firestore）、Cloudflare Turnstile、予算の上限、推論ごとの品質ログ

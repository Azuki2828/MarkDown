# 河原電子ビジネス専門学校<br>ゲームクリエイター科　　2年　東　浩樹

### 目次
* [1.概要](#概要)
* [2.操作説明](#操作説明)
* [3.担当ソースコード](#担当ソースコード)
* [4.改造したエンジンコード](#改造したエンジンコード)
* [5.技術要素](#技術要素)
  * [1.PBR（物理ベースレンダリング）](#1pbr物理ベースレンダリング)
  * [2.デプスシャドウ](#2デプスシャドウ)
    * [2.1.シャドウマップの作成](#21シャドウマップの作成)
    * [2.2.モデルに影を落とす](#22モデルに影を落とす)
  * [3.川瀬式ブルーム](#3川瀬式ブルーム)
  * [4.輪郭線の実現](#4輪郭線の実現)
* [6.こだわりと苦労した点](#こだわりと苦労した点)
  * [1.磁力について](#1磁力について)
  * [2.ライトについて](#2ライトについて)<br><br><br>

# 概要
## 作品名：磁力反転コロコーロ
<img src = "image/title.png" ald = "ゲーム概要" width = "384" height = "256">
<img src = "image/play.png" ald = "ゲーム概要" width = "384" height = "256">

## 使用ゲームエンジン：学内エンジンを改造したもの
## 使用言語：C++、HLSL
## 使用ツール：Visual Studio 2019、3ds Max
## 使用ライブラリ：BulletPhysics、Effekseer、DirectXTK12
## 環境：Windows10、DirectX12
## 制作人数：3人
## 制作期間：4ヵ月半
## GitHub : [https://github.com/Azuki2828/MyGame](https://github.com/Azuki2828/MyGame "GitHub URL")<br><br>
# 操作説明
## Jキー or Aボタン：自機の磁極を変換<br><br>
<img src = "image/howTo.png" ald = "操作説明" width = "512" height = "256"><br><br><br>

# 担当ソースコード
BackGround.h<br>
BackGround.cpp<br>
ConstTriggerBoxValue.h<br>
ConstValue.h<br>
DeathBlock.h<br>
DeathBlock.cpp<br>
DirectionLight.h<br>
DirectionLight.cpp<br>
FontRender.h<br>
FontRender.cpp<br>
HUD.h<br>
HUD.cpp<br>
Key.h<br>
Key.cpp<br>
LightBase.h<br>
LightBase.cpp<br>
LightCamera.h<br>
LightManager.h<br>
LightManager.cpp<br>
Magnet.h<br>
Magnet.cpp<br>
main.h<br>
MainCamera.h<br>
MainCamera.cpp<br>
Player.h<br>
Player.cpp<br>
RuleSceneConstData.h<br>
SaveData.h<br>
SaveData.cpp<br>
Seesaw.h<br>
Seesaw.cpp<br>
SkinModelRender.h<br>
SkinModelRender.cpp<br>
SoundManager.h<br>
SoundManager.cpp<br>
SpriteRender.h<br>
SpriteRender.cpp<br>
TreasureBox.h<br>
TreasureBox.cpp<br><br>

deferredLighting.fx<br>
gaussianBlur.fx<br>
model.fx<br>
postEffect.fx<br>
sprite.fx<br>
drawShadowMap.fx<br>
shadowReciever.fx<br>
Zprepass.fx<br>

また、上記含め、全てのコードのリファクタリングを担当。<br><br><br>

# 改造したエンジンコード<br>
RenderTarget.h(G-Buffer用のレンダーターゲットの作成及び取得を行う関数、データメンバを追加)<br>
RenderTarget.cpp(静的メンバの初期化を行うコードを追加)<br>
tkSoundSource.cpp(ファイルパスが通らなかった時に警告を出すコードを追加)<br>
Vector.h(BulletPhysicsに対応する関数、インクルードを追加)<br><br><br>

# 技術要素
## 1.PBR（物理ベースレンダリング）

本作品ではディズニーベースのPBRを実装しており、**フレネル反射を考慮した拡散反射**、**クックトランスモデル利用したスペキュラ反射**を用いています。ランバート拡散反射では、入射してきた光よりも強い光を返すため、物理的に正しくはありませんでした。その値はπ倍であることが知られています。そのため、本作品ではその値をπで割った**正規化ランバート拡散反射**を採用しています。鏡面反射では、物体の金属度、なめらかさのパラメータを元に鏡面反射率を求めています。フォン鏡面反射ではこのパラメータが無かったため、視点からサーフェイスに伸びるベクトルと反射ベクトルしか鏡面反射に使うネタがありませんでした（絞り率は定数とする）。今回は各物体ごとにパラメータを設定し、不自然な金属感を出さないようにしています。これにより、エネルギー保存則に従ったライティング処理を実装しました。<br><br><br>
<img src = "image/albedo.png" ald = "アルベド" width = "480" height = "256"><br>
▲テクスチャカラーによるゲーム画面<br>
<img src = "image/PBR.png" ald = "PBR" width = "480" height = "256"><br>
▲PBR実装後のゲーム画面<br><br><br>

## 2.デプスシャドウ
<br>
影を生成する手法として、デプスシャドウを採用しています。投影シャドウではグレースケールをシャドウマップに描き込んでいましたが、ライトスクリーン空間でのZ値（深度値）をシャドウマップに描き込むように切り替えることにより、投影シャドウの欠点であった、「影が落ちないはずの場所に影が落ちてしまう」現象を克服しました。<br><br><br>
以下がデプスシャドウの手順になります。<br><br><br>

### 2.1.シャドウマップの作成

1.シャドウマップ用のレンダーターゲットを作成する<br>
2.ライトの位置にカメラを設置する<br>
3.シャドウマップ描画用のモデルを用意する<br>
4.シャドウマップ用のピクセルシェーダーを用意する<br>
5.シャドウマップに用意した影用モデルを描画する<br><br>

今回はデプスシャドウなので、シェーダー側では、**ライトからみた深度値を書き込みます**。<br>

▼深度値が書き込まれたシャドウマップ<br>
<img src = "image/shadowMap.png" ald = "ゲーム概要" width = "256" height = "256">

### 2.2.モデルに影を落とす
<br>
シャドウマップが出来たら、いよいよ影の描画に入ります。
手順は以下のようになります。<br><br>
1.定数バッファにライトカメラのビュープロジェクション行列を登録する<br>
2.シャドウマップをシェーダーリソースビューに登録する<br>
3.ライトビュースクリーン空間での座標を計算する<br>
4.正規化スクリーン座標系に変換する<br>
5.正規化スクリーン座標系からUV座標系に変換する<br>
6.ライトビュースクリーン空間でのZ値を計算する<br>
7.範囲内なら、シャドウマップに書き込まれているZ値と比較して影を落とす<br>

<br><br>
▼シャドウなし<br>
<img src = "image/noShadow.png" ald = "ゲーム概要" width = "256" height = "256"><br>
▼デプスシャドウあり<br>
<img src = "image/depthShadow.png" ald = "ゲーム概要" width = "280" height = "256"><br><br>

## 3.川瀬式ブルーム
<br>
ぼかし表現にはガウシアンブラーを使用していますが、このブラーを一度だけでなく、複数回使用しています。その際に、ダウンサンプリング用のレンダリングターゲットを用意して、ガウスフィルターをかけて縮小します。それらを同解像度に拡大合成することであふれテクスチャを生成し、絶妙なブルームに仕上げています。<br><br>
<img src = "image/Game.png" ald = "ゲーム概要" width = "460" height = "256"><br>
▲ブルームをかける前の画面<br><br><br>
<img src = "image/luminance.png" ald = "ゲーム概要" width = "460" height = "256"><br>
▲輝度テクスチャ<br><br><br>
<img src = "image/blur_1.png" ald = "ゲーム概要" width = "460" height = "256"><br>
<img src = "image/blur_2.png" ald = "ゲーム概要" width = "460" height = "256"><br>
<img src = "image/blur_3.png" ald = "ゲーム概要" width = "460" height = "256"><br>
<img src = "image/blur_4.png" ald = "ゲーム概要" width = "460" height = "256"><br>
▲輝度テクスチャを使用して４回ガウシアンブラーをかける。<br><br><br>
<img src = "image/KawaseBloom.png" ald = "ゲーム概要" width = "460" height = "256"><br>
▲ブルームをかけた後のゲーム画面
<br><br><br>

## 4.輪郭線の実現
<br>
輪郭線の実現のために、法線テクスチャ（G-Buffer）を生成しています。この法線テクスチャでは、あるテクセルの法線情報と周囲８テクセルの法線情報を比較し、その内積で、オブジェクトの角に近いか判定しています。<br><br>
<img src = "image/edge_1.png" ald = "ゲーム概要" width = "540" height = "360"><br>
<img src = "image/edge_2.png" ald = "ゲーム概要" width = "540" height = "360"><br>
<img src = "image/edge_3.png" ald = "ゲーム概要" width = "540" height = "360"><br>
<img src = "image/edge_4.png" ald = "ゲーム概要" width = "540" height = "360"><br>
▼法線テクスチャ<br>
<img src = "image/normal.png" ald = "ゲーム概要" width = "384" height = "256"><br>
▼磁石の縁が黒くなっている。<br>
<img src = "image/Game.png" ald = "ゲーム概要" width = "384" height = "256">
<br><br><br>

# こだわりと苦労した点

## 1.磁力について

プレイヤーは、磁石から受ける磁力で自機を進めていきます。そのため、ゴールするためには磁力の強さや自機の質量、摩擦などを調整する必要がありました。しかしそれ以上に問題だったのが磁力の影響範囲です。かつては自機と磁石の距離で磁力計算をしていました。そのため、ステージ全体に磁石が設置されていることもあり、想定していない磁石からも磁力を受けてしまい、まともに進めないということが起きてしまいました。そこで、**磁力の影響範囲を、距離ではなくトリガーボックスを用いて行う**ことにしました。<br><br><br>
<img src = "image/TriggerBox.png" ald = "ゲーム概要" width = "540" height = "360"><br>▲赤線が影響範囲を示している<br><br><br>
各磁石ごとに影響範囲を決めることで、ステージギミックも作りやすくなり、不自然な動きも抑えることができました。<br><br>

## 2.ライトについて

このゲームは、各モデルごとに固有のライトを割り当てています。そのため、ステージが回転したときに全てのライトを半回転させる必要がありました。そこで、**事前に半回転用のイベントを作成しておき、カメラが半回転すると同時にそのイベント（関数）を呼ぶ**ことで、無駄の少ない処理にすることができました。
# 【ProbSpace】花粉飛散量予測【5th_place_solution】

## 目次<br>
[コンペ概要](#コンペ概要)<br>
[結果](#結果)<br>
[解法まとめ](#解法まとめ)<br>
[解法詳細](#解法詳細)<br>
[CV推移](#CV推移)<br>
[できなかったこと](#できなかったこと)<br>
<br>
## コンペ概要<br>
コンペサイト：https://comp.probspace.com/competitions/pollen_counts<br>
<br>
気象データをもとに3地点の花粉飛散量を1時間単位で予測<br>
評価指標はMAE<br>
<br>
<br>
## 結果<br>
PublicLB：9位(スコア：11.72637)<br>
PrivateLB：5位(スコア：8.42503)<br>
<br>
<br>
## 解法まとめ<br>
![image](https://user-images.githubusercontent.com/118031932/211530285-18b44e7a-6f1a-4c13-97a6-eae1d567e3f5.png)
## 解法詳細<br>
●トレーニングデータの処理<br>
・欠損値補間：<br>
　　欠測となっている部分を近くの値や近くの値の平均値で補間<br>
・基本特徴量生成：<br>
　　・datetimeからyear, month, day, hourを生成<br>
　　・winddirectionを南北軸、東西軸に変換し、windspeedをかけてベクトルに変換<br>
　　・累積気温から飛散量予測値を算出(参考：https://agriknowledge.affrc.go.jp/RN/2010762207.pdf)<br>
　　　　y：1年の花粉飛散量の累積値の相対値（花粉飛散量の累積値/1年の花粉飛散量の合計値）<br>
　　　　x：累積気温<br>
　　　　y=exp{a*exp(b*x)}としてLevenberg-Marquardt法で係数a,bを計算<br>
　　　　2020年は合計値cがわからないので、y'=花粉飛散量の累積値/cとして線形回帰でフィッティングして、<br>
　　　　合計値cと花粉飛散量の累積値予測値を算出<br>
　　　　花粉飛散量の累積値予測値を時間微分して1時間当たりの飛散量の形に戻す<br>
・Targetの補正：<br>
　　各年の飛散量に2020年の合計飛散量/各年の合計飛散量をかけて、2020年のスケールに合わせる<br>
　　学習時はlog(y+0.5)で補正<br>
  <br>
●null-importanceによる特徴量削減<br>
・そのままのデータで学習、Targetをランダムにシャッフルして100回学習して、特徴量重要度が閾値よりも乖離しているもののみ残す<br>
<br>
●データの分割<br>
・各年の4/1～4/15をCV確認用のtestデータとして利用<br>
・それ以外を学習データとして利用<br>
・学習データは大きい値が少ないので、大きい値が学習しにくいと考え、大きい値の比率が高くなるようにデータセットを作成<br>
　　log(y+0.5)＜3の行をStratified k-foldで3分割し、うち2つとlog(y+0.5)≧3の行を結合し、3種類のデータセットを作成<br>
・それらの3つのデータセットをtrain:valid=7:3で分割<br>
<br>
●LightGBM<br>
・評価指標はhuberを使用<br>
・null-importanceで残った特徴量で学習<br>
・データセット3つで3モデルを作成し、testデータの予測値3つの平均値を算出し、CVを確認<br>
<br>
●特徴量追加<br>
・仮説をもとに特徴量を追加し、上記の処理をおこない、CVの推移をみて、その特徴量を採用するか判断<br>
・最終的に採用した特徴量<br>
　・降水量ゼロがどれぐらい続いているか<br>
　・時刻を三角関数に<br>
　・太陽の位置<br>
　・降水量の移動平均(1～6時間)<br>
　・気温×風速ベクトル<br>
　・降水量の逆数<br>
　・降水量の逆数×風速ベクトル<br>
　・気温の差分(1～6時間)<br>
　・風速ベクトルのlag(1～6時間)<br>
　・気温×風速ベクトルのlag(1～6時間)<br>
　・降水量の逆数×風速ベクトルのlag(1～6時間)<br>
**`※2023/1/12追記：ma,max,minの特徴量作成のコードが間違っており、すべて降水量のma,max,minを取得するコードになっていました。`**<br>
**` 　　　　　　　　ma,max,minが採用されなかったのはこれが原因の可能性があります。`**<br>
**` 　　　　　　　　このコードでの推論結果を提出していますので、修正はしていません。使用される際はご注意ください。`**<br>
<br>
●交互作用作成<br>
・特徴量が出揃ったら、全組み合わせで特徴量の交互作用を作成<br>
・特徴量上位n個で学習し、地域ごとに使用する特徴量を決定（宇都宮100個、千葉150個、東京300個）<br>
<br>
●最終モデル作成<br>
・seedを1～10まで変更し、Optunaでハイパーパラメーターを最適化したモデルを30個作成<br>
<br>
●最終予測<br>
・testデータで地域ごとに特徴量を生成し、予測値を30個算出　Predict.ipynb<br>
・seed=1のものをSubmissionデータ　Final_sub07_seed1.ipynb<br>
・すべての平均をとったものをsubmissionデータ　Final_sub07_Averaging.ipynb<br>
・いずれも負の値は0に、0以上の値は4の倍数に丸める<br>
<br>
## CV推移<br>
![image](https://user-images.githubusercontent.com/118031932/211542276-c2b0d4e5-0b54-4dd3-82b8-f31668f25713.png)
<br>
## できなかったこと<br>
CVでグラフ形状を毎回確認しながら進めていましたが、どうしても大きい値がうまく予測できませんでした。<br>
いろいろ工夫しましたが、最後まで改善することはできませんでした。<br>
<br>
・trainデータの各年・各都市の4/1～15をtestデータとしたときの予測結果<br>
![新規 ビットマップ イメージ](https://user-images.githubusercontent.com/118031932/211548385-b8f5f0fd-aeb2-456e-af96-0d5554e3c0da.jpg)
<br>
<br>
<br>
# Lisence
<br>
This project is licensed under the MIT License, see the LICENSE file for details


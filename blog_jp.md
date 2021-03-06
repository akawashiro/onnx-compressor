# ONNXファイルから不要な枝を削ってMNISTの推論を高速化してみる
この記事の中のソースコードは全て[https://github.com/akawashiro/sonnx](https://github.com/akawashiro/sonnx)にあります。
## 概要
- ニューラルネットワークから要らなそうな枝を80%削除しても精度が変わらなかった
- ONNXの中身をいじるのが大変だった
- [onnxruntime](https://github.com/Microsoft/onnxruntime)には勝てなかった
## 背景
機械学習の学習済みモデルを小さなデバイスで動かす、というのが最近流行っているそうです。機械学習では、学習には大きな計算コストがかかりますが、推論はそれほど大きな計算コストがかかりません。このため、学習だけを別のコンピュータで行っておいて、実際の推論は小さなデバイスで行うということが可能です。

ただし、推論だけでもそれなりに計算資源が必要です。そこで、学習済みのモデルの高速化が重要になります。Raspberry Piに搭載されているGPUを使う[Idein](https://blog.idein.jp/post/172704317595/idein%E3%81%AE%E6%8A%80%E8%A1%93%E3%82%84%E4%BA%8B%E6%A5%AD%E3%81%AE%E7%B4%B9%E4%BB%8B)とか有名です。

僕も学習済みモデルの推論を高速化できそうな方法を思いついたので実験してみました。
## アイデア
今回はMNISTを分類する学習済みモデルを高速化します。今回使用するモデルは次の図のようなものです。画像は28*28(=784)pxなので入力は784個、出力は各数字の確率なので10個あり、中間層が2つ挟まっています。各層間は全結合しており、活性化関数としてReluを使います。

<img src="https://raw.githubusercontent.com/akawashiro/sonnx/master/original_NN.png" width=300px>

このモデルを教師データを使って学習すると、枝の重みが変わってこんな感じになります。  <br/>
<img src="https://raw.githubusercontent.com/akawashiro/sonnx/master/learned_NN.png" width=300px>

僕のアイデアは学習後のネットワークから重みの小さい枝を取り去ってもちゃんと動くんじゃないか、というものです。重みの小さい枝を取り去るとこんな感じになります。
<img src="https://raw.githubusercontent.com/akawashiro/sonnx/master/compressed_NN.png" width=300px>
枝の本数が少なくなれば、推論を高速化できそうな気がします。

## アイデアの裏付け
実際に学習後のモデルで赤丸で囲った部分の重みの分布を確認します。
<img src="https://raw.githubusercontent.com/akawashiro/sonnx/master/designate_layer.png" width=400px>

分布はこのようになっています。
<img src="https://raw.githubusercontent.com/akawashiro/sonnx/master/140406443536680_figure.png" width=400px>
重みが0の部分が非常に多いです。まず、学習済みモデルから重みが0の枝を削除しても推論結果に影響しないはずです。また、グラフが左右対称になっているので、絶対値の小さい順に削除していけば、各パーセプトロンへの入力はそれほど変化しない気がします。

## 手法
やることは非常に単純です。ニューラルネットワーク中の各層間枝の重みの統計を取り、重み上位何%かを残して残りを削除します。つまり、何らかの方法で学習済みのモデルから枝の重みのデータを取り出し、枝をカットし、さらに加工後のモデルのデータを使って推論できるようにします。

幸い今はONNXという良いものがあります。ONNXとは学習済みモデルのデータを出力する形式の一つで、多くのフレームワークが対応しています。

今回はChainerで書いたモデルからONNXデータを出力し、そのデータを加工することにしました。また加工後のデータはC++で書いた俺俺ONNXランタイムに実行してもらうことにしました。

纏めると、
1. Chainerでニューラルネットワークを書いて学習する
2. 学習済みのニューラルネットワークからONNXデータを出力する
3. ONNXデータを俺俺ONNXランタイムに読み込んで加工、実行する

となります。一つづつ何をやるのかを説明します。

#### 1. Chainerでニューラルネットワークを書いて学習する
#### 2. 学習済みのニューラルネットワークからONNXデータを出力する
1と2は簡単です。[onnx-chainer](https://github.com/chainer/onnx-chainer)を使えばすぐにできます。
```
python3 learn_mnist.py
```
で`mnist.onnx`というファイルができます。

#### 3. ONNXデータを俺俺ONNXランタイムに読み込んで加工、実行する
ここが大変でした。ディープラーニングフレームワークからのONNXモデルの出力は多くの人が試しているのですが、出力したONNXモデルをチューニングしようとする人はほとんどいないようです。
##### 3.1 ONNXデータを解析する
とりあえず[netron](https://github.com/lutzroeder/netron)というONNXの可視化ツールで`mnist.onnx`を可視化してみました。
<img src="https://raw.githubusercontent.com/akawashiro/sonnx/master/mnist.png" width=250px>
`Gemm`はGeneral matrix multiplyの略です。各Gemmノードは行列`B`とベクトル`C`を持ち、ベクトル`x`を入力として`Bx+C`を出力します。`Relu`は[活性化関数](https://ja.wikipedia.org/wiki/%E6%B4%BB%E6%80%A7%E5%8C%96%E9%96%A2%E6%95%B0)です。

Reluは`max(0,x)`で定義されている関数のでONNXから抽出する必要は無く、各Gemmノードの行列`B`とベクトル`C`の情報だけをを抽出できれば良いです。

今回は各Gemmノードの`B`と`C`をテキストファイルとして抽出します。
```
> python3 analyze_mnist_onnx.py
```
は次のファイルを出力します。
- `*************_matrix.txt`
`mnist.onnx`の全てのGemmノードの`B`行列と`C`行列
- `*************_matrix.png`
各行列の中の重みの分布
- `mnist.onnx.json`
ONNXファイルをJSONに変換したもの
- `mnist_train.txt`
- `mnist_test.txt`
MNISTをC++から読み込みやすくするためにテキストファイルに変換したもの

各`*************_matrix.txt`がどのGemmノードに対応するのかは`mnist.onnx.json`を睨むとわかります(←ここ超不親切)。

##### 3.2 抽出したデータを加工、実行する
```
g++ -O3 -mtune=native -march=native -mfpmath=both sonnx.cpp && ./a.out
```
で出力した重みのデータを読み込み、推論を実行します。`sonnx.cpp`は簡単なONNXランタイムになっており、`mnist.onnx`から抽出した行列のデータとMNISTの画像データのテキストファイルから推論を行います。デフォルトでは`mnist_test.txt`の10000枚について推論を行います。

出力はこのようになります。
```
accuracy, time
0.9817000031, 15.93887043
compress_ratio, accuracy, time
0, 0.9817000031, 36.66616821
0.05, 0.9817000031, 34.61940384
0.1, 0.9818000197, 32.64707184
0.15, 0.9818000197, 30.78090286
0.2, 0.9817000031, 29.01914406
..........................
```
2行目は読み込んだデータを加工せずで隣接行列表現で保持したときの推論の精度と計算時間、4行目以降は読み込んだデータを加工したときの推論の圧縮率(compress_ratio)、精度と計算時間です。圧縮率が0のときは全く加工しないのと同じ、圧縮率が0.3のときは枝の重みの絶対値が小さい30%を削除したときの結果です。圧縮率が1になると全ての枝が削除されます。

データを加工して枝を削除するとき、`sonnx.cpp`では行列データを非零成分のインデックスとその値の組の配列として保持しています。これは枝を削除したときに保持するデータ量が減らし、高速に計算するためです。
## 結果
圧縮率を変化させたときの精度と計算時間のグラフです。圧縮率を0.8まで上げても推論の精度が変わっていません。これは学習済みモデル中の8割の枝を削除しても、推論精度が保てるということです。驚きですね!
<img src="https://raw.githubusercontent.com/akawashiro/sonnx/master/sonnx_result.png" width=500px>

一方、計算時間はほぼ枝の数に反比例しています。これは予想通りでした。
<img src="https://raw.githubusercontent.com/akawashiro/sonnx/master/compression_rate_time.png" width=500px>

表にしてみます。推論精度を1%犠牲にするだけで2倍も高速化できています。やったね!
| 圧縮率 | 推論精度 | 計算時間(秒) |
|:-----------:|:------------:|:------------:|
| 0 | 0.981 | 15.9 |
| 0.8 | 0.971 | 7.04 |
※ 圧縮率0のときの値は重みのデータを隣接行列表現で保持したときのものです。

## おまけ
最後に[onnxruntime](https://github.com/Microsoft/onnxruntime)で元の学習済みモデルを実行したときの時間を計ってみます。
```
> python3 analyze_mnist_onnx.py
Accuracy rate =  0.9817 , time =  3.285153865814209
```
え、3秒?? 普通に考えると16秒ぐらいになるはずです。なぜこんなに速いんだ...

理由は色々あると思いますが、僕が思いつくのは以下の2つです。
- AVXなどのSIMD拡張命令を使っている
  (僕もAVXを使ってみようとは思いましたが、scatter命令が僕のPCで使えないので諦めました。)
- CPUのキャッシュに乗るようなプログラムになっている

## 関連記事・研究
- [深層学習のモデル圧縮・高速化に関する論文80本ノック](http://madoibito80.hatenablog.jp/entry/2018/04/06/121059)
学習済みのモデルを圧縮する論文がまとめられています。このブログ記事の内容はリンク先の「スパースなモデル」の章に該当します。
- [モデルアーキテクチャ観点からのDeep Neural Network高速化 ](https://www.slideshare.net/ren4yu/deep-neural-network-79382352)
DNNの高速化に関するスライドです。Pruningの章がこのブログ記事に該当します。
- [Reducing the Model Order of Deep Neural Networks Using Information Theory](https://arxiv.org/abs/1605.04859)
上のリンク中の論文で特にこの記事に近いものです。このブログ記事でやっているのは、この論文のmagnitude-based pruningです。
- [Deep Compression: Compressing Deep Neural Networks with Pruning, Trained Quantization and Huffman Coding](https://arxiv.org/abs/1510.00149)
CPUでの実行速度もオリジナルのモデルより速くなってるっぽくてすごい。何をどう実装したのか知りたい。
- [keras-surgeon](https://github.com/BenWhetton/keras-surgeon)
kerasのモデルをpruningするためのソフトウェア。

## まとめ
- ONNXランタイムを自作したらナイーブな実装の二倍の速度で推論ができた
- `onnxruntime`には勝てなかった
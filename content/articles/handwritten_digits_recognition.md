+++
date = "2016-03-27T10:46:59+09:00"
draft = false
title = "numpy で Neural Network"

+++

1月から [Coursera の Machine Learning のコース](https://www.coursera.org/learn/machine-learning/)を受講していて、先日すべての課題を提出した。
前評判通り、分かりやすくてとても良いコースだった。

<br>

[![https://gyazo.com/d85bca5456116786cd7fa9e8b2a4f99c](https://i.gyazo.com/d85bca5456116786cd7fa9e8b2a4f99c.png)](https://gyazo.com/d85bca5456116786cd7fa9e8b2a4f99c)

<br>

このコースでは [GNU Octave](https://www.gnu.org/software/octave/) を使うんだけど、
やっぱり numpy つかってやってみたいよねー、ってことで、復習も兼ねて numpy で Neural Network を作ってみた。

最終結果は github と heroku にあげておいた。

- https://github.com/tyamagu2/nnhw
- https://nnhw.herokuapp.com/


### Neural Network で学習

題材はコースと同じく手書きの数字認識。学習データには定番の [MNIST のデータセット](http://yann.lecun.com/exdb/mnist/)を使った。

Neural Network の構成は input layer、hidden layer 1 つ、output layer の 3 層。
input layer には 28 x 28 の画像をそのまま渡すことにしたので、bias unit も合わせて 28 * 28 + 1 = 785 ユニット。
feature reduction とかはやってない。
hidden layer は 30 ~ 100 ユニットぐらいで試してみた。output layer は数字の分の 10 ユニット。
Regularization の lambda とか learning rate はいくつか試して適当に決めてしまった。あんまり検証はしてない。

最初は batch gradient descent、つまり全学習データを一気に Neural Network に丸投げしていたんだけど、
それだと全然精度が出なかったので、mini-batch gradient descent で 500 個ずつ画像を渡すようにした。
batch gradient descent の時もコスト関数は収束しているようだったので、local optima にハマったのかなー。

そんな感じで自分の MacBook Pro で 20 分も学習させたところ、MNIST のテストセットで 90 ~ 95% ぐらいは精度が出た。
ちなみに、数字ごとの各ピクセルの平均値を出して、ピクセルごとの差分を使って最小二乗法で分類、
というシンプルなやり方だと 82% ぐらい。なのでそれよりはマシな結果になった。

### 手書き文字を Canvas で入力

やっぱりインタラクティブじゃないとねー、ということで、Canvas に書いた文字を認識させるようにした。
Canvas の内容を 28 x 28 にダウンサイズして Neural Network に渡している。
Web アプリのフレームワークには flask を使ってみた。
本当はダウンサイズした画像を表示したり、output layer の数値を表示したり、とかやろうかと思ったけど、
フロントエンドでいい感じに表示するのもめんどくさくて、シンプルに結果だけ表示することにした。

<br>

![gif](https://camo.githubusercontent.com/8d63997affda3ed7108119e2ac513af5f52279c3/687474703a2f2f692e696d6775722e636f6d2f6f4978614931422e676966)

実際には上の gif のようにバッチリ分類はしてくれない。それでも、ピクセルごとの差分の最小二乗法よりはマシな精度になったけど。

### Heroku にデプロイ

せっかくなので Heroku にデプロイすることにしたけど、ここで予定外と時間を使ってしまった。

実は今回の Neural Network では scipy も使っている。
具体的には、sigmoid 関数として `scipy.special.expit` を使っている。そこだけ。
これは `1.0 / (1.0  + np.exp(-x))` ってやれば numpy でも計算できる。
でも x がある程度大きい負の数だと `np.exp(-x)` がオーバーフローしてしまって精度が出なかった。
`math.exp(-x)` でも同様だった。

そういうわけで、heroku でも scipy をインストールしないといけないんだけど、試しに deploy したら、
heroku には scipy をインストール出来ないと言われてしまった。
調べたら [third party 製の buildpack を使えよ](https://devcenter.heroku.com/articles/python-c-deps)、とのことだったので、
[conda-buildpack](https://github.com/kennethreitz/conda-buildpack) なるものを使うことにした。
そうすると今度は conda というパッケージマネージャーを使う必要が出てきた。
それまでは全部 pip で管理してたので、pyenv で miniconda を入れて、conda でパッケージインストールして、
とやったところ、今度は flask が起動しなくなった。werkzeug がインストール済みなのにインポートできないと言われた。
しょうがないから flask は pip で管理して、numpy と scipy は conda で管理することにした。

さて改めて heroku にデプロイ、したら今度は slug size が 300MB を超えてデプロイできなかった。
調べていたら[こんな Issue](https://github.com/kennethreitz/conda-buildpack/issues/21) が見つかった。
[この Announcement](https://www.continuum.io/blog/developer-blog/anaconda-25-release-now-mkl-optimizations) にも色々書いてあるけど、
mkl optimized な numpy/scipy を、optimize されてない nomkl version の numpy/scipy にしたらサイズが小さくなるので大丈夫とのこと。
というわけで nomkl 版にしてデプロイ成功。

### 感想

やっぱり実データだと全然精度が出ない。どこかにしょーもないバグがあるかもしれないけど、まあこんなもんなんだろうなと思う。

いくつか推測した原因：

1. 学習データの偏り<br>
いくつか学習データを表示してみた感じだと、日本人の書く数字とはちょっと感じが違う気がした。
たまたま表示したデータがそうなのかもしれないし、実際のところは分からない。
もっと regularization の lambda の調整が必要なのかもしれないし、さらに学習データを増やすべきなのかもしれない。

1. 入力画像のプリプロセス不足<br>
不足というか何もしてない。中心やサイズの調整をしてあげると結果も違ってきそう。

結局実用まで持って行くには、lambda や hidden layer の数やユニット数などのパラメータの調整、
それを助けるためのグラフを書いたりするスクリプト、実用的な速度を出すための feature selection、
色々な学習データの収集やらプリプロセスなどの泥臭い作業、なんかが必要になってくるんだろうなーと思った。
当然だけど Machine Learning とか Neural Network とか Deep Learning とかの響きに惹かれてるだけじゃ難しいですね。
数学ももっと勉強しないと。

MNIST は簡単すぎるので、[CIFAR-10](https://www.cs.toronto.edu/~kriz/cifar.html) というデータセットでやってみると良いと教えてもらった。
こっちだと 50% ぐらいしかでない。これで 90% 出せたらすごいらしい。

あと [Neural Networks and Deep Learning](http://neuralnetworksanddeeplearning.com/) っていう Online Book が良いらしい。
この本で色々勉強しつつ、CIFAR-10 にチャレンジするなり、もうちょっと違う何かに取り組むなりやってみようかなー。

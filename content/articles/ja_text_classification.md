+++
date = "2016-09-05T22:44:24+09:00"
draft = false
title = "MeCab と scikit-learn で日本語テキストを分類する"

+++

scikit-learn を使って日本語テキストの分類をやった時に色々調べたメモ。

基本的なテキスト分類のやり方は、[scikit-learn のチュートリアル](http://scikit-learn.org/stable/tutorial/text_analytics/working_with_text_data.html)を参考にした。
簡単に説明すると、以下のようなやり方。

1. `CountVectorizer` でテキスト内の単語の出現回数をカウントして
1. `TfidfTransformer` で単語の出現回数から tf-idf を計算して
1. 適当な分類器（`MultinomialNB`、`SGDClassifier` など）に投げて分類する

ただ、上記のチュートリアルそのままだと2点ほど問題があった。

1. `CountVectorizer` がテキストを単語に分割してくれるが、日本語の（MeCab を使った）分割はどうすればよいのか？
1. テキスト以外の特徴量も使うにはどうすればよいのか？

### 日本語の（MeCab を使った）分割はどうすればよいのか？

`CountVectorizer` はテキストを与えたら勝手に単語に分割して出現回数を数えてくれる。
しかし、これは英語のようにスペースで単語が分割可能な言語の話であって、日本語ではそうはいかない。

そこで `CountVectorizer` の[ドキュメント](http://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.CountVectorizer.html)を調べたところ、
コンストラクタの `analyzer` 引数にメソッドを渡せば良いことが分かった。

> analyzer: string, {'word', 'char', 'char_wb'} or callable
>
Whether the feature should be made of word or character n-grams. Option ‘char_wb’ creates character n-grams only from text inside word boundaries.
>
If a callable is passed it is used to extract the sequence of features out of the raw, unprocessed input.
...

例えば以下の `WordDividor` のようなクラスを作って、`extract_words` インスタンスメソッドを `CountVector` のコンストラクタに `analyzer` 引数として渡す。

```
import MeCab
from sklearn.feature_extraction.text import CountVectorizer

class WordDividor:
    INDEX_CATEGORY = 0
    INDEX_ROOT_FORM = 6
    TARGET_CATEGORIES = ["名詞", " 動詞",  "形容詞", "副詞", "連体詞", "感動詞"]

    def __init__(self, dictionary="mecabrc"):
        self.dictionary = dictionary
        self.tagger = MeCab.Tagger(self.dictionary)

    def extract_words(self, text):
        if not text:
            return []

        words = []

        node = self.tagger.parseToNode(text)
        while node:
            features = node.feature.split(',')

            if features[self.INDEX_CATEGORY] in self.TARGET_CATEGORIES:
                if features[self.INDEX_ROOT_FORM] == "*":
                    words.append(node.surface)
                else:
                    # prefer root form
                    words.append(features[self.INDEX_ROOT_FORM])

            node = node.next

        return words

if __name__ == '__main__':
    data = [
        '蛙の子は蛙',
        '親の心子知らず'
    ]

    wd = WordDividor()
    cv = CountVectorizer(analyzer=wd.extract_words)

    counts = cv.fit_transform(data)
    print(cv.vocabulary_)
    print(counts)
```

これを実行すると、以下のようにちゃんと単語に分割してカウントされている。

```
{'蛙': 3, '知らず': 2, '子': 0, '心子': 1, '親': 4}
  (0, 3)        2
  (0, 0)        1
  (1, 4)        1
  (1, 1)        1
  (1, 2)        1
```

ちなみにこの例では `mecab-python3` を使っている。`mecab-python3` の使い方は割愛。
ポイントとして、特定の品詞の単語だけを使うようにしている。また、原形が提示されている場合は原形を使っている。

あと、この例ではやってないけど、[mecab-ipadic-neologd](https://github.com/neologd/mecab-ipadic-neologd) を使うようにしたり、
[この辺](https://github.com/neologd/mecab-ipadic-neologd/wiki/Regexp)を参考にテキストを事前に正規化したりすると良いと思う。

一つ注意点として、この `CountVector` のインスタンスはそのままではシリアライズできない。
いろいろ調べたところ、`WordDividor` のインスタンスメソッドを保持しているのが問題らしい。
`pickle` だけじゃなくて、`dill` みたいなライブラリを使っても同様だった。

なんでわざわざクラスを作っているかというと、MeCab の `Tagger` の初期化を一度だけにしたいから。
トップレベルのメソッド内で毎回 `Tagger` を作るようにすれば上記の問題はなくなるけど、
そうすると今度は `analyzer` の呼び出しのたびに `Tagger` の初期化が走ってとても時間がかかる。
この辺、なんかうまい解決策があれば良いのだけど。

いちおう対策として、`CountVector` のインスタンスをシリアライズする代わりに、`vocabulary_` アトリビュートをシリアライズして保存しておく方法がある。
デシリアライズ時は `CountVector` のコンストラクタの `vocabulary` 引数にデシリアライズした `vocabulary_` を、
`analyzer` 引数にはシリアライズ時と同じセットアップした `WordDividor` インスタンスの `extract_words` を渡すようにすれば、
同じ状態の `CountVector` が作れる。

### テキスト以外の特徴量も使うにはどうすればよいのか？

例えば tweet にはテキスト以外にも投稿者、投稿日時、位置情報などの情報がある。こういったテキスト情報以外も特徴量として使いたいよねー、ってことで方法を探したところ、
[FeatureUnion](http://scikit-learn.org/stable/modules/generated/sklearn.pipeline.FeatureUnion.html) を使えば良いことが分かった。

`FeatureUnion` は、簡単に言うと `CountVectorizer` などの transformer の出力データをまとめるための機能。
詳しい使い方は[この辺の例](http://scikit-learn.org/stable/modules/generated/sklearn.pipeline.FeatureUnion.html#examples-using-sklearn-pipeline-featureunion)を見るのが良いと思う。

具体的に、入力データが [text, float, float] というフォーマットの場合を考えてみる。text は `CountVectorizer` -> `TfidfTransformer` を適用して tf-idf に変換したい、
残りのデータはそのまま使いたい、とする。FeatureUnion を使えば、以下の様な事ができる。

1. text のみを取り出す transformer と、それ以外を取り出す transformer を作る
1. text を取り出す transformer の出力は、さらに `CountVectorizer` ->  `TfidfTransformer` で処理して tf-idf に変換する
1. text 以外を取り出す transformer の出力と、`TfidfTransformer` の出力を FeatureUnion でまとめる
1. FeatureUnion の出力を SDGClassifier などに渡して学習・分類する

これをコードにしたのが以下。

```
import MeCab
import numpy as np
from sklearn.base import BaseEstimator, TransformerMixin
from sklearn.feature_extraction.text import CountVectorizer, TfidfTransformer
from sklearn.linear_model import SGDClassifier
from sklearn.pipeline import FeatureUnion, Pipeline

class WordDividor:
    ...

class TextExtractor(BaseEstimator, TransformerMixin):
    def fit(self, x, y=None):
        return self

    def transform(self, rows):
        return rows[:, 0]

class OtherFeaturesExtractor(BaseEstimator, TransformerMixin):
    def fit(self, x, y=None):
        return self

    def transform(self, rows):
        return rows[:, 1:].astype('float')

if __name__ == '__main__':
    train_data = np.array([
        ['蛙の子は蛙', 0.5, 0.5],
        ['親の心子知らず', 0.2, 0.7]
    ])

    train_labels = [1, 2]

    test_data = np.array([
        ['鬼の居ぬ間に洗濯', 0.1, 0.8],
        ['取らぬ狸の皮算用', 0.4, 0.3]
    ])

    wd = WordDividor()

    clf1 = Pipeline([
        ('count_vector', CountVectorizer(analyzer=wd.extract_words)),
        ('tfidf', TfidfTransformer()),
        ('classifier', SGDClassifier(loss='hinge', random_state=42))
    ])

    clf1.fit(train_data[:, 0], train_labels)
    print("text only:")
    print(clf1.predict(test_data[:, 0]))

    clf2 = Pipeline([
        ('features', FeatureUnion([
            ('text', Pipeline([
                ('content', TextExtractor()),
                ('count_vector', CountVectorizer(analyzer=wd.extract_words)),
                ('tfidf', TfidfTransformer())
            ])),
            ('other_features', OtherFeaturesExtractor()),
        ])),
        ('classifier', SGDClassifier(loss='hinge', random_state=42))
    ])

    clf2.fit(train_data, train_labels)
    print("feature union:")
    print(clf2.predict(test_data))
```

`TextExractor` の `transform` メソッドでは、[text, float, float] というデータの先頭のカラム、つまりテキストだけ取り出して配列として返している。
一方 `OtherFeaturesExtractor` の `transform` メソッドでは、先頭の text カラムを除外した配列を返している。
`FeatureUnion` では `OtherFeatureExtractor` の出力と、`TexExtractor` の出力を `CountVectorizer`、`TfidfTransformer` と通したものを結合している。
最後に `FeatureUnion` の出力が `SDGClassifier` に渡っている。

上記を動かした結果はこんなかんじ。テキストだけの時と結果が変わっているので、テキスト以外のデータもちゃんと使われていると思われる。

```
text only:
[1 1]
feature union:
[2 1]
```

なお、`Pipeline` は[そのままシリアライズできる](http://scikit-learn.org/stable/modules/model_persistence.html)んだけど、
上記の例の `CountVectorizer` のようにシリアライズできないものが含まれていると、当然 `Pipeline` 自体のシリアライズにも失敗するので注意。

----

上記の例そのままだとシリアライズ周りで不都合があったり、ndarray のデータタイプを無理やり変換してたりと問題はあるけど、
似たようなことをするときには参考になるんじゃないかなと思う。

コードは https://github.com/tyamagu2/ja_text_classification_sample にアップロードしてある。

---
title: "Rector で始める自動リファクタリング入門"
emoji: "🍖"
type: "tech"
topics: ["php", "rector"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false
---

# 書くこと

- 通常の使い方
- PHP7.4 未満のクラスにタイプドプロパティを導入するデモ
- カスタムルールの作り方
- test から始まるテストメソッドをアノテーション方式に変えるデモ
- カスタムルールのテストの仕方
- 用意されているヘルパーの紹介
- 自分でカスタムルールを作るときのコツ

# はじめに

みなさん、コードの改善はしていますでしょうか。
コードの改善と一口に言っても様々な性格のものが存在しますが、その中でも**ある一定のルールに基づく機械的な作業**が要求されるものがあるかと思います。

例えば、

- 全ての関数 A を関数 B に変えたい
- クラス A を継承している全てのクラスの親クラスをクラス B に変えたい
- コンストラクタ内でインスタンス化している処理を DI パターンに変えたい

などなど……。

小規模なものであれば手作業でおこなっても問題のないことが多いですが、大規模になると**うっかりミスが発生しがち**だったり単純作業ゆえの**精神的な苦痛**が伴うこともあったりなど、あまり好ましくありません。

これから紹介する Rector は、そういった**機械的な作業からエンジニアを開放する**とてもすごいツールです。

是非この記事で Rector の基本的な使い方を覚え、日々の業務を楽にしてみてください。

# サンプルプロジェクトの構築

この記事では予め用意されたサンプルコードに対して Rector を実際に実行することで理解を深めていきたいと思います。

## 推奨環境

- PHP >=7.4

## git clone

作業ディレクトリから以下のコマンドを実行しサンプルプロジェクトをローカル環境に構築します。

```bash
$ git clone git@github.com:YAhiru/rector-tutorial.git
$ cd rector-tutorial
$ composer i
```

## サンプルプロジェクトの確認

サンプルプロジェクトは以下の構成になっています。

```
.
├── rector    # Rector 用のディレクトリ
│   ├── src   # 記事の終盤で作成するカスタムルール用のディレクトリ
│   └── tests # カスタムルールのテストディレクトリ
├── src       # サンプルのアプリケーションコードディレクトリ
└── tests     # サンプルのアプリケーションコードのテストディレクトリ
```

理由は後述しますが Rector 用のディレクトリとサンプルのアプリケーションコードを分けています。

ここでいうアプリケーションコードとは Web アプリケーションであればコントローラーやエンティティなどに当たるもので、リファクタリングの対象となるコードです。
今回は src ディレクトリに単純なクラスを 1 つだけ用意しています。

rector ディレクトリはこの記事の後半で実装するカスタムルールのためのディレクトリとなっていますので、最低限の設定のみで残りはほぼ空の状態となっています。

# インストール

Rector は普通に `composer req --dev rector/rector` を実行して利用する他に [phar](https://github.com/rectorphp/rector-prefixed) や [docker](https://hub.docker.com/r/rector/rector) などの形式が用意されています。
[Rector の依存関係](https://github.com/rectorphp/rector/blob/master/composer.json#L27) を確認するとわかる通り依存するライブラリが非常に多いため、 **カスタムルールを作りたいなどのニーズがない場合** は phar あるいは docker 形式での利用をおすすめします。

今回は最終的にカスタムルールを作りたいため rector/rector を composer 経由でダウンロードするわけですが、前述の通り**依存関係が多いため既存のプロダクトにインストールすることができない**といった問題が予想されます。
そういった場合に有効なのが Rector 用のサブディレクトリを作成し、その中で Rector 関連のコードを完結させることです。([アドバイス](https://twitter.com/tadsan/status/1315854532191023104) ありがとうございます!!)
サンプルプロジェクトでもそのアプローチを採用しています。

では、以下のコマンドで Rector のインストールをします。

```bash
$ cd rector
$ composer req rector/rector
```

# 予め用意されているルールを適用してみる

それでは Rector を試していきたいと思います。
今回自動リファクタリングをしてみるファイルは `src/User.php` です。

```php:src/User.php
final class User
{
    /** @var string */
    private $screenName;
    /** @var int */
    private $age;

    public function __construct(string $screenName, int $age)
    {
        $this->screenName = $screenName;
        $this->age = $age;
    }

    // ...
}
```

User クラスのプロパティである `$screenName` や `$age` を見てみると [Typed Property](https://wiki.php.net/rfc/typed_properties_v2) が適用されていないことがわかります。
Typed Property が指定されていない場合 PHP Doc に記述された型以外が代入された場合でもエラーが発生せず、思わぬバグを引き起こす可能性を秘めています。
あなたの所属するチーム内でこのことが問題視されたという設定で、Rector を使ってこのクラスを自動リファクタリングしてみましょう。

## Rector を実行する

Rector の実行は簡単で `vendor/bin/rector process {{ディレクトリ}} --set {{ルールセット}}` のように、 process コマンドに実行したいディレクトリや適用したい **ルールセット** や **ルール** を渡せば良いだけです。

唐突にルールやルールセットという単語を使いましたが、
ルールとはリファクタリングの内容が定義された PHP のクラスのことで、
ルールセットとは関心ごとが近い複数のルールがまとめられたもののことです。

今回は Typed Property だけを適用してくれればよいので、 **php74** というルールセットの中にある **Rector\Php74\Rector\Property\TypedPropertyRector** というルールのみを採用したいと思います。

**rector ディレクトリ内で** 以下のコマンドを実行してみましょう。

```bash
$ vendor/bin/rector process ../src --set php74 --only Rector\Php74\Rector\Property\TypedPropertyRector
```

## 修正されたファイルを確認する

コマンドを実行するとそれっぽさのある出力が表示されたかと思いますが、実際の User クラスを確認してみると以下のように `$screenName` と `$age` に Typed Property が適用されていることがわかります。

```php:src/User.php
final class User
{
    private string $screenName;
    private int $age;

    public function __construct(string $screenName, int $age)
    {
        $this->screenName = $screenName;
        $this->age = $age;
    }

    // ...
}
```

**どうですか！！！！！！！！すごくないですか！？！？！？！？！？！？！？！？**（唐突な興奮）

## その他のルールの紹介

Rector には 600 個を超える様々なルールが用意されているので、使えそうなものがないか探してみてください。
中には CakePHP の `App::uses()` を use 文に変えてくれるルールなどもあり、軽い感動を覚えます。

- [ルールセットの一覧]()
- [ルール一覧](https://github.com/rectorphp/rector/blob/master/docs/rector_rules_overview.md)

# カスタムルールを作る

- テストの作り方を紹介する
- test から始まるテストメソッドをアノテーション形式に変えるカスタムルールを実際に作る。
- テストが通ることを確認する

# カスタムルールを実行する

- config の書き方を紹介する
- config を元に rector を実行する

# カスタムルールを作る際のコツ

- やりたい内容に近いことを行っている既存のルールのコードを読むことが手取り早い

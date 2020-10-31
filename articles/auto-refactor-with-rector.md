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
  - カスタムルールって言い方を変えたい(公式な呼び方じゃない)
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

これから紹介する [Rector](https://github.com/rectorphp/rector) は、そういった**機械的な作業からエンジニアを開放する**とてもすごいツールです。

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

rector ディレクトリはこの記事の後半で実装するカスタムルールのためのディレクトリとなっていますので、最低限の設定のみでほぼ空の状態となっています。

# インストール

Rector は `composer req --dev rector/rector` を実行して利用する他に [phar](https://github.com/rectorphp/rector-prefixed) や [docker](https://hub.docker.com/r/rector/rector) などの形式が用意されています。
[Rector の依存関係](https://github.com/rectorphp/rector/blob/master/composer.json#L27) を確認するとわかる通り依存するライブラリが非常に多いため、 **カスタムルールを作りたいなどのニーズがない場合** は phar あるいは docker 形式での利用をおすすめします。

今回は最終的にカスタムルールを作りたいため [rector/rector](https://github.com/rectorphp/rector) を composer 経由でダウンロードするわけですが、前述の通り依存関係が多いため**既存のプロダクトにインストールすることができない**といった問題が予想されます。
そういった場合に有効なのが **Rector 用のサブディレクトリを作成しその中で Rector 関連のコードを完結させること**です。^[[アドバイス](https://twitter.com/tadsan/status/1315854532191023104) ありがとうございます!!]
今回のサンプルプロジェクトでもそのアプローチを採用しています。

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
ルールとは**リファクタリングの内容が定義された PHP のクラス**のことで、
ルールセットとは**関心ごとが近い複数のルールがまとめられたもの**のことです。

今回は Typed Property だけを適用してくれればよいので、 **php74** というルールセットの中にある **Rector\Php74\Rector\Property\TypedPropertyRector** というルールのみを採用したいと思います。

**rector ディレクトリ内で** 以下のコマンドを実行してみましょう。

```bash:rector/
$ vendor/bin/rector process ../src --set php74 --only Rector\Php74\Rector\Property\TypedPropertyRector
```

## 修正されたファイルを確認する

コマンドを実行するとそれっぽさのある出力がなされたかと思いますが、実際の User クラスを確認してみると以下のように `$screenName` と `$age` に Typed Property が適用されていることがわかります。

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

Rector を使ってリファクタリングを自動化することで工数を大幅に削減することが出来るため、あなたはチームメンバーに一目を置かれる存在となることができます。

## その他のルールの紹介

今回使ったルールの他に、Rector には 600 個を超える様々なルールが用意されています。あなたのプロダクトで使えそうなルールがないか探してみてください。

- [ルールセットの一覧](https://github.com/rectorphp/rector/tree/master/config/set)
- [ルール一覧](https://github.com/rectorphp/rector/blob/master/docs/rector_rules_overview.md)

# カスタムルール

リファクタリングの要件はプロダクトによって様々なので Rector が用意してくれているルールだけでは**要件を満たせない**ことがあります。
そういった場合に有効なのがカスタムルールです。

カスタムルールとはつまり **Rector が用意してくれているようなルールを自分で作ってしまおう**という話です。
カスタムルールを作ることが出来れば**プロダクト固有の要件であってもリファクタリングを自動化**することができます。

今回はあなたの所属するチーム内で「PHPUnit のテストメソッドは、メソッド名に `test` という prefix を付ける形式ではなく `@test` アノテーションを使う形式にしたい」という要望が出たという設定で進めていきましょう。^[あくま記事の都合上の話であり @test アノテーションを推奨しているわけではありません。]

## カスタムルールのテストを書く

早速カスタムルールを作っていきたいのですが、まずは**どのようなコードをどのようにリファクタリングしたいのか**ということを明確にする必要があります。
Rector はテスト基盤も整っていますので、テストを書くことでルールのイメージを固めていきましょう。

ルールのテストでは必要なものが 2 つあります。

1. テストクラス
1. Fixture

### テストクラス

ここでいうテストクラスとは、 `PHPUnit\Framework\TestCase` を継承した任意のクラスのことです。
Rector は `PHPUnit\Framework\TestCase` を拡張した `Rector\Core\Testing\PHPUnit\AbstractRectorTestCase` ^[継承関係には `Symplify\PackageBuilder\Tests\AbstractKernelTestCase` なども含まれていますが本筋から逸れるためスキップしています。] が用意されており、これによって**ルールのテストがとても簡単になる**ため今回はこれを利用します。

**rector ディレクトリ内**で以下のコマンドを実行してテストクラス用のファイルを作成します。

```bash:rector/
$ mkdir -p tests/AddTestAnnotationRector
$ touch tests/AddTestAnnotationRector/AddTestAnnotationRectorTest.php
```

ファイルが作成されたら、ファイルに以下の内容を書き込みます。

```php:rector/tests/AddTestAnnotationRector/AddTestAnnotationRectorTest.php
<?php
declare(strict_types=1);
namespace Yahiru\RectorTutorialRector\AddTestAnnotationRector;

use Iterator;
use Rector\Core\Testing\PHPUnit\AbstractRectorTestCase;
use Symplify\SmartFileSystem\SmartFileInfo;
use Yahiru\RectorTutorialRector\AddTestAnnotationRector;

final class AddTestAnnotationRectorTest extends AbstractRectorTestCase
{
    /**
     * @dataProvider provideData()
     */
    public function test(SmartFileInfo $fileInfo) : void
    {
        $this->doTestFileInfo($fileInfo);
    }

    /**
     * @return Iterator<SmartFileInfo>
     */
    public function provideData() : Iterator
    {
        return $this->yieldFilesFromDirectory(__DIR__ . '/Fixture');
    }

    protected function getRectorClass() : string
    {
        return AddTestAnnotationRector::class;
    }
}
```

ルールのテストクラスは `Rector\Core\Testing\PHPUnit\AbstractRectorTestCase` を使うことで多くの場合に上記とほぼ同じ内容で済みます。

`getRectorClass()` はテスト対象となるルールのクラス名を返却します。

```php:rector/tests/AddTestAnnotationRector/AddTestAnnotationRectorTest.php
    protected function getRectorClass(): string
    {
        return AddTestAnnotationRector::class;
    }
```

ここで返却したルールを `Rector\Core\Testing\PHPUnit\AbstractRectorTestCase` 内でよしなにテストしてくれます。

また `AddTestAnnotationRector` クラスはまだ存在していませんが、テストを実装した後に作成する予定です。

`provideData()` は、この後作成する Fixture ファイルを読み込み、 `test()` は読み込まれた各 Fixture に対してテストを実行します。

```php:rector/tests/AddTestAnnotationRector/AddTestAnnotationRectorTest.php
    /**
     * @dataProvider provideData()
     */
    public function test(SmartFileInfo $fileInfo): void
    {
        $this->doTestFileInfo($fileInfo);
    }

    /**
     * @return Iterator<SmartFileInfo>
     */
    public function provideData(): Iterator
    {
        return $this->yieldFilesFromDirectory(__DIR__ . '/Fixture');
    }
```

### Fixture

Fixture とは、**リファクタリング前のコード** と **リファクタリング後のコード** が以下のフォーマットで記述されたファイルのことです。

```
<?php
リファクタリング前のコード
?>
-----
<?php
リファクタリング後のコード
?>
```

実際の Fixture を見た方が理解が早いので、早速 Fixture を作ってみましょう。
**rector ディレクトリ内**で以下のコマンドを実行して Fixture を作成します。

```bash:rector/
$ mkdir -p tests/AddTestAnnotationRector/Fixture
$ touch tests/AddTestAnnotationRector/Fixture/fixture.php.inc
```

今回は `test` という prefix がついているテストメソッドを `@test` アノテーション形式に変換するというのが要件ですので、作成したファイルに以下の内容を書き込みます。

```php:rector/tests/AddTestAnnotationRector/Fixture/fixture.php.inc
<?php
namespace Yahiru\RectorTutorialRector\AddTestAnnotationRector\Fixture;

class SomeTest extends \PHPUnit\Framework\TestCase
{
    public function testSome() : void
    {
        // do test
    }
}

?>
-----
<?php
namespace Yahiru\RectorTutorialRector\AddTestAnnotationRector\Fixture;

class SomeTest extends \PHPUnit\Framework\TestCase
{
    /**
     * @test
     */
    public function some() : void
    {
        // do test
    }
}

?>
```

今作成した Fixture の差分は以下の通りです。
`test` から始まるテストメソッドが `@test` アノテーション形式に変わっています。

```diff
+   /**
+    * @test
+    */
+   public function some() : void
-   public function testSome() : void
    {
        // do test
    }
```

#### 注意点

Fixture を作成する上での重要な注意点ですが、 Rector は**リファクタリング対象のコードを動的に読み込みます**。

つまり今作成した Fixture に書かれているクラスも読み込まれるということなので、 Fixture にうっかり namespace を書き忘れてしまうと、他のテストの Fixture や既存クラスと**クラス名が衝突する可能性が増します**。

なので絶対に Fixture には **namespace を忘れないように**しましょう。

#### Fixture を複数用意する

テストの準備自体はこれで完了ですが、テスト項目が足りていないのでもう少し Fixture を増やしたいと思います。

今回のルールにおいて、メソッドがリファクタリング対象であると判断する基準は以下とします。

1. クラスが `PHPUnit\Framework\TestCase` を継承していること
1. メソッド名が `test` で始まっていること
1. メソッドの可視性が public であること

正常系は先ほどの Fixture で満たしているので、次は上記の条件以外ではリファクタリングされないことを保証する Fixture を追加します。
テストしたい内容を明確にするために、**適切な粒度で Fixture を分ける**と良いでしょう。

**rector ディレクトリ内**で以下のコマンドを実行してファイルを作成してください。

```bash:rector/
$ touch tests/AddTestAnnotationRector/Fixture/{no-test-prefix.php.inc,not-public-method.php.inc,not-test-class.php.inc}
```

コードの変更がない場合の Fixture は**リファクタリング後のコードを省くことが可能**ですので、それぞれのファイルに以下の内容を書き込みんでください。

```php:rector/tests/AddTestAnnotationRector/Fixture/no-test-prefix.php.inc
<?php
namespace Yahiru\RectorTutorialRector\AddTestAnnotationRector\Fixture;

class NotTestPrefix extends \PHPUnit\Framework\TestCase
{
    public function noPrefix() : void
    {
    }
}
```

```php:rector/tests/AddTestAnnotationRector/Fixture/not-public-method.php.inc
<?php
namespace Yahiru\RectorTutorialRector\AddTestAnnotationRector\Fixture;

class NotPublicMethodTest extends \PHPUnit\Framework\TestCase
{
    protected function testProtected() : void
    {
    }

    private function testPrivate() : void
    {
    }
}
```

```php:rector/tests/AddTestAnnotationRector/Fixture/not-test-class.php.inc
<?php
namespace Yahiru\RectorTutorialRector\AddTestAnnotationRector\Fixture;

class NotTest
{
    public function testSome() : void
    {
    }
}
```

Fixture の準備はこれで完了です。

## カスタムルールを作る

次はルールを作成します。
ルールは基本的に `Rector\Core\Rector\AbstractRector` を継承して作成します。

**rector ディレクトリ内**で以下のコマンドを実行してファイルを作成してください。

```bash:rector/
$ touch src/AddTestAnnotationRector.php
```

作成したファイルに以下の内容を書き込んでください。

```php:rector/src/AddTestAnnotationRector.php
<?php
declare(strict_types=1);
namespace Yahiru\RectorTutorialRector;

use PhpParser\Node;
use PhpParser\Node\Stmt\ClassMethod;
use PHPUnit\Framework\TestCase;
use Rector\Core\Rector\AbstractRector;
use Rector\Core\RectorDefinition\CodeSample;
use Rector\Core\RectorDefinition\RectorDefinition;
use Rector\NodeTypeResolver\Node\AttributeKey;

final class AddTestAnnotationRector extends AbstractRector
{
    public function getDefinition() : RectorDefinition
    {
        return new RectorDefinition('Add @test annotation', [
            new CodeSample(
                <<<'CODE_SAMPLE'
                class SomeTest extends \PHPUnit\Framework\TestCase
                {
                    public function testSome() : void
                    {
                        // do test
                    }
                }
                CODE_SAMPLE
                ,
                <<<'CODE_SAMPLE'
                class SomeTest extends \PHPUnit\Framework\TestCase
                {
                    /**
                     * @test
                     */
                    public function some() : void
                    {
                        // do test
                    }
                }
                CODE_SAMPLE
            ),
        ]);
    }

    /**
     * @phpstan-return array<class-string<Node>>
     *
     * @return array<string>
     */
    public function getNodeTypes() : array
    {
        return [ClassMethod::class];
    }

    /**
     * @param ClassMethod $node
     */
    public function refactor(Node $node) : ?Node
    {
        if (! $this->shouldRefactor($node)) {
            return null;
        }

        $phpDocInfo = $node->getAttribute(AttributeKey::PHP_DOC_INFO);
        if ($phpDocInfo === null) {
            $phpDocInfo = $this->phpDocInfoFactory->createEmpty($node);
        }

        $phpDocInfo->addBareTag('@test');

        $testName = \lcfirst(
            (string) \preg_replace('/\Atest/', '', (string) $this->getName($node))
        );
        $node->name = new Node\Identifier($testName);

        return $node;
    }

    private function shouldRefactor(ClassMethod $node) : bool
    {
        $class = $node->getAttribute(AttributeKey::CLASS_NODE);

        return $node->isPublic()
            && $this->isName($node, 'test*')
            && $this->isObjectType($class, TestCase::class);
    }
}
```

#### getDefinition

`getDefinition()` はリファクタリング前後のイメージを返します。

```php:rector/src/AddTestAnnotationRector.php
    public function getDefinition() : RectorDefinition
    {
        return new RectorDefinition('Add @test annotation', [
            new CodeSample(
                <<<'CODE_SAMPLE'
                class SomeTest extends \PHPUnit\Framework\TestCase
                {
                    public function testSome() : void
                    {
                        // do test
                    }
                }
                CODE_SAMPLE
                ,
                <<<'CODE_SAMPLE'
                class SomeTest extends \PHPUnit\Framework\TestCase
                {
                    /**
                     * @test
                     */
                    public function some() : void
                    {
                        // do test
                    }
                }
                CODE_SAMPLE
            ),
        ]);
    }
```

カスタムルールにおいてこれは必須ではないですが、ルールの利用者がルールの性質を理解する手助けとなるので書いておいて損はないでしょう。

#### getNodeTypes

`getNodeTypes()` はリファクタリング対象の Node クラスの配列を返却します。

```php:rector/src/AddTestAnnotationRector.php
    public function getNodeTypes() : array
    {
        return [ClassMethod::class];
    }
```

今回はクラスメソッド以外には興味がないので `PhpParser\Node\Stmt\ClassMethod` だけを指定しています。
Node の一覧は[こちら](https://github.com/rectorphp/rector/blob/master/docs/nodes_overview.md)から確認できます。

#### refactor

`refactor()` はルールの核となるメソッドです。
このメソッドで `$node` を書き換え返却するとその内容が反映され、 `null` を返すとスキップされます。

```php:rector/src/AddTestAnnotationRector.php
    /**
     * @param ClassMethod $node
     */
    public function refactor(Node $node) : ?Node
    {
        if (! $this->shouldRefactor($node)) {
            return null;
        }

        // ...

        return $node;
    }
```

今回は `getDefinition()` で `PhpParser\Node\Stmt\ClassMethod` のみを指定しているため `refactor()` には `PhpParser\Node\Stmt\ClassMethod` のインスタンスのみ渡されることが Rector によって保証されています。

なので PHPDoc は`@param PhpParser\Node\Stmt\ClassMethod $node`とし、 `$node` は `PhpParser\Node\Stmt\ClassMethod` のインスタンスであることを前提にロジックを組み立てて構いません。

```php:rector/src/AddTestAnnotationRector.php
    public function refactor(Node $node) : ?Node
    {
        // こういうことはしなくて良い
        if (! $node instanceof ClassMethod) {
            return null;
        }
    }
```

クラスメソッドであれば何でもリファクタリングしていいというわけではないので、 `refactor()` の最初にブロック文を実装しています。
`shouldRefactor()` はリファクタリングをすべきかどうかを判断するために実装したメソッドで、 `AddTestAnnotationRector` クラス固有のものです。

```php:rector/src/AddTestAnnotationRector.php
    /**
     * @param ClassMethod $node
     */
    public function refactor(Node $node) : ?Node
    {
        if (! $this->shouldRefactor($node)) {
            return null;
        }

        // ...
    }

    private function shouldRefactor(ClassMethod $node) : bool
    {
        // クラスメソッドが実装されているクラス情報を取得
        $class = $node->getAttribute(AttributeKey::CLASS_NODE);

        return
            // メソッドの可視性が public かどうか
            $node->isPublic()
            // メソッド名が `test` から始まっているかどうか
            && $this->isName($node, 'test*')
            // メソッドが実装されているクラスが `PHPUnit\Framework\TestCase` を継承しているかどうか
            && $this->isObjectType($class, TestCase::class);
    }
```

無事ブロック文を抜けたら、実際にリファクタリングをします。

```php:rector/src/AddTestAnnotationRector.php
    public function refactor(Node $node) : ?Node
    {
        // ...

        $phpDocInfo = $node->getAttribute(AttributeKey::PHP_DOC_INFO);
        if ($phpDocInfo === null) {
            $phpDocInfo = $this->phpDocInfoFactory->createEmpty($node);
        }

        $phpDocInfo->addBareTag('@test');

        $testName = \lcfirst(
            (string) \preg_replace('/\Atest/', '', (string) $this->getName($node))
        );
        $node->name = new Node\Identifier($testName);

        return $node;
    }
```

まず最初に PHPDoc に `@test` アノテーションを追加する処理をしています。

```php:rector/src/AddTestAnnotationRector.php
// `$node` から PHP Doc の情報を取得
$phpDocInfo = $node->getAttribute(AttributeKey::PHP_DOC_INFO);
if ($phpDocInfo === null) { // PHP Doc がない場合は空の PHP Doc を作成する
    $phpDocInfo = $this->phpDocInfoFactory->createEmpty($node);
}

$phpDocInfo->addBareTag('@test'); // PHP Doc に `@test` タグを追加する
```

次に `$this->getName($node)` で取得したメソッド名を元に新しいメソッド名を設定し直しています。

```php:rector/src/AddTestAnnotationRector.php
// 元のメソッド名から `test` prefix を削除し、最初の1文字を小文字に変更した文字列
$testName = \lcfirst(
    (string) \preg_replace('/\Atest/', '', (string) $this->getName($node))
);
// `$testName` を新しいメソッド名として設定しなおす
$node->name = new Node\Identifier($testName);
```

最後に `$node` を返却して、変更があったことを Rector に知らせて完了です。

```php:rector/src/AddTestAnnotationRector.php
return $node;
```

## 作成したルールをテストする

それではルールが問題なく実装できているか確認しましょう。
**rector ディレクトリ内**で以下のコマンドを実行してテストを走らせてください。

```bash:rector/
$ composer test
```

エラーが発生していなければ OK です。

# カスタムルールを実行する

- config の書き方を紹介する
- config を元に rector を実行する

# カスタムルールを作る際のコツ

- やりたい内容に近いことを行っている既存のルールのコードを読むことが手取り早い

```

```

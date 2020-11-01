---
title: "Rector で始める自動リファクタリング入門"
emoji: "🍖"
type: "tech"
topics: ["php", "rector"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false
---

# はじめに

みなさん、コードの改善はしていますでしょうか。
コードの改善と一口に言っても様々な性格のものが存在しますが、その中でも**ある一定のルールに基づく機械的な作業**が要求されるものがあるかと思います。

例えば、

- 全ての関数 A を関数 B に変えたい
- クラス A を継承している全てのクラスの親クラスをクラス B に変えたい
- コンストラクタ内でインスタンス化している処理を DI パターンに変えたい

などなど……。

小規模なものであれば手作業でおこなっても問題のないことが多いですが、大規模になると**うっかりミスが発生しがち**だったり単純作業ゆえの**精神的な苦痛**が伴うこともあったりなど、あまり好ましくありません。

本記事で紹介する [Rector](https://github.com/rectorphp/rector) はそういった**機械的な作業からエンジニアを開放する**とてもすごいツールです。

是非この記事で Rector の基本的な使い方を覚え、日々の業務を楽にしてみてください。

# サンプルプロジェクトの構築

この記事では予め用意された[サンプルコード](https://github.com/YAhiru/rector-tutorial)に対して Rector を実際に実行することで理解を深めていきたいと思います。

## 推奨環境
:::message
- PHP >=7.4
:::

## git clone

作業ディレクトリから以下のコマンドを実行しサンプルプロジェクトをローカル環境に構築します。

```bash
$ git clone git@github.com:YAhiru/rector-tutorial.git
$ cd rector-tutorial
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

理由は後述しますが **Rector 用のディレクトリとサンプルのアプリケーションコードを分けています**。

ここでいうアプリケーションコードとは Web アプリケーションであればコントローラーやエンティティなどに当たるもので、リファクタリングの対象となるコードです。
今回は src ディレクトリに単純なクラスを 1 つだけ用意しています。

rector ディレクトリはこの記事の後半で実装するカスタムルールのためのディレクトリとなっていますので、最低限の設定のみでほぼ空の状態となっています。

# インストール

Rector は `composer req --dev rector/rector` を実行して利用する他に [phar](https://github.com/rectorphp/rector-prefixed) や [docker](https://hub.docker.com/r/rector/rector) などの形式が用意されています。
[Rector の依存関係](https://github.com/rectorphp/rector/blob/master/composer.json#L27) を確認するとわかる通り依存するライブラリが非常に多いため、 **カスタムルールを作りたいなどのニーズがない場合は phar あるいは docker 形式での利用をおすすめします**。

今回は最終的にカスタムルールを作りたいため [rector/rector](https://github.com/rectorphp/rector) を composer 経由でダウンロードするわけですが、前述の通り依存関係が多いため**既存のプロダクトにインストールすることができない**といった問題の発生が予想されます。
そういった場合に有効なのが **Rector 用のサブディレクトリを作成しその中で Rector 関連のコードを完結させること**です。^[[アドバイス](https://twitter.com/tadsan/status/1315854532191023104) ありがとうございます!!]
今回のサンプルプロジェクトでもそのアプローチを採用しています。

では、以下のコマンドで Rector のインストールをします。

```bash
$ cd rector
$ composer req rector/rector
```

# ルールを適用してみる

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

User クラスのプロパティである `$screenName` や `$age` を見てみると [Typed Property](https://www.php.net/manual/ja/language.oop5.properties.php#language.oop5.properties.typed-properties) が適用されていないことがわかります。
Typed Property が適用されていない場合 PHP Doc に記述された型以外が代入された場合でもエラーが発生せず、思わぬバグを引き起こす可能性を秘めています。
あなたの所属するチーム内でこのことが問題視されたという設定で、Rector を使ってこのクラスを自動リファクタリングしてみましょう。

## Rector を実行する

Rector の実行は簡単で `vendor/bin/rector process {{ディレクトリ}} --set {{ルールセット}}` のように、 process コマンドに対してターゲットとなるディレクトリや適用したい **ルールセット** や **ルール** を渡せば良いだけです。

|名前|説明|
|---|---|
|ルール|リファクタリングの内容が定義された PHP のクラス|
|ルールセット|関心ごとが近い複数のルールがまとめられたもの|

今回は Typed Property だけを適用してくれればよいので、 **php74** というルールセットの中にある **Rector\Php74\Rector\Property\TypedPropertyRector** というルールのみを適用したいと思います。

**rector ディレクトリ内で**以下のコマンドを実行してみましょう。

```bash:rector/
$ vendor/bin/rector process ../src \
    --set php74 \
    --only 'Rector\Php74\Rector\Property\TypedPropertyRector'
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

Rector を使ってリファクタリングを自動化することで**修正規模によっては工数を大幅に削減することが出来る**ため、あなたの上司もニッコリです。

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

ここでいうテストクラスとは `PHPUnit\Framework\TestCase` を継承した任意のクラスのことです。
Rector には `PHPUnit\Framework\TestCase` を拡張した `Rector\Core\Testing\PHPUnit\AbstractRectorTestCase` ^[継承関係には `Symplify\PackageBuilder\Tests\AbstractKernelTestCase` なども含まれていますが本筋から逸れるためスキップしています。] が用意されており、これによって**ルールのテストがとても簡単になる**ため今回はこれを利用します。

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

`getRectorClass()` は**テスト対象となるルールのクラス名を返却**します。

```php:rector/tests/AddTestAnnotationRector/AddTestAnnotationRectorTest.php
    protected function getRectorClass(): string
    {
        return AddTestAnnotationRector::class;
    }
```

ここで返却したルールを `Rector\Core\Testing\PHPUnit\AbstractRectorTestCase` 内でよしなに扱ってくれます。
`AddTestAnnotationRector` クラスはまだ存在していませんが、テストを実装した後に作成する予定です。

`provideData()` はこの後作成する予定の Fixture ファイルを読み込み、 `test()` は読み込まれた Fixture を利用してテストを実行します。

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

Fixture とは、以下のフォーマットで **リファクタリング前のコード** と **リファクタリング後のコード** が記述されたファイルのことです。

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
**rector ディレクトリ内**で以下のコマンドを実行してファイルを作成します。

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

書き込んだ内容が前述のフォーマット通りになっていることを確認してください。
また、使用しているエディターによっては上記のコードをコピペした際にリファクタリング前と後の区切りを表す `-----` の直前にスペースが入り込むことがあるので注意してください。

また、作成した Fixture の差分は以下の通りです。
テストメソッドが `@test` アノテーション形式に変わっています。

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

つまり今作成した **Fixture に定義されているクラスも読み込まれる**ということなので、 Fixture にうっかり namespace を書き忘れてしまうと、他のテストの Fixture や既存クラスと**クラス名が衝突する可能性が増します**。

なので絶対に **Fixture には namespace を書くように**しましょう。

#### Fixture を複数用意する

テストの準備自体はこれで完了ですが、テスト項目が足りていないのでもう少し Fixture を増やしたいと思います。

今回のルールにおいて、メソッドがリファクタリング対象であると判断する基準は以下とします。

1. クラスが `PHPUnit\Framework\TestCase` を継承していること
1. メソッド名が `test` で始まっていること
1. メソッドの可視性が public であること

正常系は先ほどの Fixture でテストできているので、次は上記の条件以外ではリファクタリングされないことを保証する Fixture を追加します。

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
蛇足ですが、テストしたい内容を明確にするために **Fixture は適切な粒度で分ける**ことを意識すると良いでしょう。

## カスタムルールを作る

次はカスタムルールを作成します。
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

`getDefinition()` は**リファクタリング前後のイメージ**を返します。

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

カスタムルールにおいて `getDefinition` に正確な内容を記述することは必須ではないですが、ルールの利用者が**ルールの性質を理解する手助けとなる**ので書いておいて損はないでしょう。

#### getNodeTypes

`getNodeTypes()` は**リファクタリング対象の Node クラスの配列**を返却します。

```php:rector/src/AddTestAnnotationRector.php
    public function getNodeTypes() : array
    {
        return [ClassMethod::class];
    }
```

Node とは Rector 内部で使用されている [nikic/php-parser](https://github.com/nikic/PHP-Parser) によって AST (抽象構文木) にパースされた結果の各要素のことです。

今回はクラスメソッド以外には興味がないので `PhpParser\Node\Stmt\ClassMethod` だけを指定しています。
Node の一覧は[こちら](https://github.com/rectorphp/rector/blob/master/docs/nodes_overview.md)から確認できます。

#### refactor

`refactor()` はルールの核となるメソッドです。
このメソッドに渡された **`$node` を書き換えることでその内容がファイルに反映されます**。
また、**スキップする場合は null を返却**します。

```php:rector/src/AddTestAnnotationRector.php
    /**
     * @param ClassMethod $node
     */
    public function refactor(Node $node) : ?Node
    {
        if (! $this->shouldRefactor($node)) {
            return null;
        }

        // do refactor

        return $node;
    }
```

今回は `getDefinition()` で `PhpParser\Node\Stmt\ClassMethod` のみを指定しているため、 Rector によって `refactor()` には **`PhpParser\Node\Stmt\ClassMethod` のインスタンスのみが渡される**ことが保証されています。

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

```php:rector/src/AddTestAnnotationRector.php
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
`shouldRefactor()` はリファクタリングをすべきかどうかを判断するために実装したメソッドで、 **`AddTestAnnotationRector` クラス固有のもの**です。

ブロック文を抜けたらコードの修正に相当するロジックを実行し `$node` に変更を加えます。

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

まず最初に PHP Doc に `@test` アノテーションを追加する処理をしています。

```php:rector/src/AddTestAnnotationRector.php
// メソッドの PHP Doc を取得
$phpDocInfo = $node->getAttribute(AttributeKey::PHP_DOC_INFO);
if ($phpDocInfo === null) {
    // PHP Doc がない場合は空の PHP Doc を作成する
    $phpDocInfo = $this->phpDocInfoFactory->createEmpty($node);
}

$phpDocInfo->addBareTag('@test'); // PHP Doc に `@test` タグを追加する
```

次に `$this->getName($node)` で取得した**メソッド名から新しいメソッド名を生成**し、設定し直しています。

```php:rector/src/AddTestAnnotationRector.php
// 元のメソッド名から `test` prefix を削除し、1文字目を小文字に変更した文字列
$testName = \lcfirst(
    (string) \preg_replace('/\Atest/', '', (string) $this->getName($node))
);
// `$testName` を新しいメソッド名として設定しなおす
$node->name = new Node\Identifier($testName);
```

最後に `$node` を返却することで**変更を Rector に知らせて**完了です。

```php:rector/src/AddTestAnnotationRector.php
return $node;
```

`refactor()` の挙動をより理解したい場合は `nikic/php-parser` の [Walking the AST](https://github.com/nikic/PHP-Parser/blob/master/doc/component/Walking_the_AST.markdown) を読むことをオススメします。

## 作成したルールをテストする

それではルールが問題なく実装できているか確認しましょう。
**rector ディレクトリ内**で以下のコマンドを実行して先ほど作成したテストを走らせてください。

```bash:rector/
$ composer test
```

エラーが発生していなければ OK です。
テストが落ちた場合は Fixture の内容が記事の内容と一致しているか改めて確認してください。

# カスタムルールを実行する

自作したルールを実行するためには、コンフィグから **Rector にルールを登録する**必要があります。

**rector ディレクトリ内**で以下のコマンドを実行してコンフィグファイルを作成してください。

```bash:rector/
$ vendor/bin/rector init
```

`rector.php` というファイルが rector ディレクトリに作成されたかと思うので、作成されたファイルを以下の内容で書き換えてください。

```php:rector/rector.php
<?php
declare(strict_types=1);

use Symfony\Component\DependencyInjection\Loader\Configurator\ContainerConfigurator;

return static function (ContainerConfigurator $containerConfigurator) : void {
    $services = $containerConfigurator->services();

    $services->set(\Yahiru\RectorTutorialRector\AddTestAnnotationRector::class);
};
```

`ContainerConfigurator` についての詳しい使い方は [Symfony のドキュメント](https://symfony.com/doc/current/service_container.html) を参照していただければと思いますが、 `$services->set()` の部分でルールを Rector に登録しています。

config の作成が完了したら以下のコマンドで Rector を実行します。
**config ファイルで登録されたルールはオプションで渡さなくても有効になる**ので、引数はディレクトリの指定のみで OK です。

```bash:rector/
$ vendor/bin/rector process ../tests
```

それっぽさのある出力がなされたかと思いますが、実際に変更されたファイルを見てみましょう。
`tests/UserTest.php` に以下のような差分が発生していれば成功です!

```diff:tests/UserTest.php
+   /**
+    * @test
+    */
+   public function getScreenName() : void
-   public function testGetScreenName() : void
    {
        $this->assertSame('test user', $this->user->getScreenName());
    }

+   /**
+    * @test
+    */
+   public function getAge() : void
-   public function testGetAge() : void
    {
        $this->assertSame(20, $this->user->getAge());
    }
}
```

# Next Step

駆け足でいろいろと解説しましたが、**意外と簡単そうだな**という感想を抱いた方も多いのではないでしょうか。
きっとより理解を深めることが出来ると思います!!

...とはいえ、そんなに都合よく作りたいカスタムルールがあるとは限りませんよね。
その場合は今回作成したカスタムルールをさらに改善してみてください。

実はこの記事で作成したカスタムルールは、現状の実装のままでは**あまり好ましくない挙動をするケース**があります。
たとえば以下のようなテストメソッドの場合です。
```php
/**
 * @test
 */
public function testFoo() : void
{
    // do test
}
```

なので、既に `@test` アノテーションがある場合は PHP Doc に変更を加えないような修正をしてみましょう。

## カスタムルールを作る際のコツ

この記事では最低限の説明になってしまっているので、実際に自分でルールを作るとなると戸惑ってしまうことも多いかと思います。
しかも Rector のドキュメントは**カスタムルールを作るための説明が少なめ**なためドキュメントに頼ることもできません。^[[これ](https://github.com/rectorphp/rector/blob/master/docs/create_own_rule.md)しかない]

そこで、カスタムルールを作るときに知っていると便利なことをいくつか共有したいと思います。

### やりたい内容に近いことを行っているルールを探す

ドキュメントがあまりないので**最良の教材は既存ルールのコード**ということになります。
幸い Rector には**600を超えるルールが既に実装されている**ので、**ドキュメントがなくとも何とかなります**。

探し方としては、[ルール一覧](https://github.com/rectorphp/rector/blob/master/docs/rector_rules_overview.md) からページ内検索をするのが早いと思います。
ルール一覧には**リファクタリング前後の差分が PHP コードとして掲載されている**ので、それを手掛かりにすると簡単に見つけることができます。

たとえば PHP Doc 関連のリファクタリングがしたい場合は、上記のページから**ページ内検索**で `/**` のような PHP Doc っぽい文字列で検索してみると**案外簡単に見つかります**。
あまりにも大量にヒットしてしまう場合は差分の表現である `+` や `-` とセットで `+␣␣␣␣/**` などと検索をすることで検索結果を絞ることができます。(`␣` は半角スペースに読み替えてください)

お目当てのルールが見つかったあとは**そのルールのコードを読むことで大体それっぽいことが出来るようになる**と思います。

### よく使うメソッド・クラス
よく使うメソッドやクラスをパッと思いついた範囲で共有します。
この他には**既存のルールのコードを読むことで便利なメソッドをいろいろと発見できる**かと思われます。

| クラス | メソッド | 説明 |
| --- | --- | --- |
|Rector\Core\Rector\AbstractRector|isName|第一引数の Node の名前が 第二引数の文字列と一致するか判定してくれる。第二引数には正規表現も使える|
|Rector\Core\Rector\AbstractRector|getName|第一引数に渡した Node の名前を返してくれる|
|Rector\Core\Rector\AbstractRector|getShortName|FQN を渡すとショートネームを返してくれる|
|Rector\Core\Rector\AbstractRector|isObjectType|第一引数に渡した Node が第二引数のクラスとマッチするか判定してくれる(Class\_ など型情報を持つ Node に使う)|
|PhpParser\NodeAbstract|getAttribute|Rector\NodeTypeResolver\Node\AttributeKey の定数を使うことで様々な情報を Node から取得できる|
|Rector\Core\PhpParser\Node\BetterNodeFinder|find\*|Node の配列と検索条件を渡すと、検索条件にマッチする Node だけ返してくれる|
|Rector\Core\Rector\AbstractPHPUnitRector| - |PhpUnit 関連のルールを作るときに便利。今回作ったカスタムルールのテストメソッド判定ルールも、実はこのクラスで既に実装されてる。|
|Rector\BetterPhpDocParser\PhpDocInfo\PhpDocInfo|hasByName|引数に渡した文字列にマッチするタグが PhpDoc に含まれているかどうか判定してくれる|
|Rector\BetterPhpDocParser\PhpDocInfo\PhpDocInfo|getTagsByName|引数に渡した文字列にマッチするタグを取得する|

### 参考になるリンク一覧

- [ライフサイクル](https://github.com/rectorphp/rector/blob/master/docs/how_it_works.md)
- [Walking the AST - nikic/php-parser](https://github.com/nikic/PHP-Parser/blob/master/doc/component/Walking_the_AST.markdown)
- [ルールセット一覧](https://github.com/rectorphp/rector/tree/master/config/set)
- [ルール一覧](https://github.com/rectorphp/rector/blob/master/docs/rector_rules_overview.md)
- [config のオプション一覧](https://github.com/rectorphp/rector#full-config-configuration)
- [ルールの設定変更](https://github.com/rectorphp/rector/blob/master/docs/how_to_configure_rules.md)
- [Service Container](https://symfony.com/doc/current/service_container.html)
- [自作ルールの作り方](https://github.com/rectorphp/rector/blob/master/docs/create_own_rule.md)
- [generate コマンド](https://github.com/rectorphp/rector/blob/master/docs/rector_recipe.md)
- [Node 一覧](https://github.com/rectorphp/rector/blob/master/docs/nodes_overview.md)
- [テストについて](https://github.com/rectorphp/rector/blob/master/docs/how_to_add_test_for_rector_rule.md)


# おわりに

Rector はすごい！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！！
Rector をフル活用してコード改善を楽にしていきましょう！！！！

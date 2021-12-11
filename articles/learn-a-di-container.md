---
title: "作りながら学ぶDIコンテナ"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["PHP", "DIContainer"]
published: false
---

この記事は[スターフェスティバル Advent Calendar 2021](https://qiita.com/advent-calendar/2021/stafes)の 12 日目です。

---

# はじめに

DI コンテナって便利ですよね。中身を知らないと魔法のようにも思える便利さですが、機能を絞れば実は割と簡単に実装できるんです。
この記事は超単純な実装から始め、素朴な Autowiring を実装するところまで解説し、DI コンテナについていわゆる[完全に理解したという状態](https://togetter.com/li/1268851)[^1]になってもらおうという趣旨です。

[^1]: 基本的な仕組みがわかった程度の理解を想定しています

なるべく誰でも理解できるように冗長なくらい説明していくスタンスでやっていきたいと思います。

# 超簡単な実装

では早速実装していきましょう。
適当なファイルに以下のコードをコピペし、実行してみてください

```php
<?php

class Container
{
    /** @var array<string, string> */
    private $definitions = [];

    public function define(string $id, string $className): void
    {
        $this->definitions[$id] = $className;
    }

    public function get(string $id)
    {
        if (isset($this->definitions[$id])) {
            $className = $this->definitions[$id];
            return new $className();
        }

        return null;
    }
}

class Foo {}

$container = new Container();

$container->define('foo', Foo::class);

var_dump($container->get('foo'));
```

```bash
$ php src/Container.php
object(Foo)#2 (0) {
}
```

おめでとうございます！DI コンテナを作ることができました！

## 実装の解説

簡単に実装の解説をしたいと思います。

### 振る舞い

まずは振る舞いの確認です。
この DI コンテナは、define メソッドを実行することで「この ID(`$id`)が呼ばれた時はこのクラス(`$className`)を返してね」と定義(define)することができます。

```php
class Container
{
    public function define(string $id, string $className): void
    {
        // ...
    }
}
```

そして get メソッドは ID(`$id`)を渡すとその ID に紐づいたクラスを返してくれます。

```php
class Container
{
    public function get(string $id)
    {
        // ...
    }
}
```

このような振る舞いを持つことで以下のようなコードが書けるようになりますね。

```php
$container = new Container();

$container->define('foo', Foo::class);

var_dump($container->get('foo'));
```

### データ構造

次にデータ構造を見てみます。
この DI コンテナには `$definitions` という[インスタンスプロパティ](https://www.php.net/manual/ja/language.oop5.properties.php) が定義されています。

```php
class Container
{
    /** @var array<string, string> */
    private $definitions = [];
}
```

`$definitions` はその名の通り `define` メソッドを通じて定義された定義たちが入っています。
格好良く定義と言っても実際には ID がキーでクラス名が値の単純な[連想配列](https://www.php.net/manual/ja/language.types.array.php)です。

```php
class Container
{
    public function define(string $id, string $className): void
    {
        // IDがキーでクラス名が値の連想配列の要素を追加している
        $this->definitions[$id] = $className;
    }
}
```

なので以下のコードを実行した場合の `$definitions` の値は `['foo' => 'Foo']` となります。

```php
$container->define('foo', Foo::class);
```

(`Foo::class` という構文の挙動がわからない人は[こちらのページ](https://www.php.net/manual/ja/language.oop5.basic.php#language.oop5.basic.class.class)を参考にしてください。)

というわけで、define メソッドを実行することで `$definitions` に ID とクラス名のマップ、つまり定義が追加されていくので、 get メソッドでは `$definitions` の中から ID に紐づいたクラス名を探し出し[new キーワード](https://www.php.net/manual/ja/language.oop5.basic.php#language.oop5.basic.new)に渡すことで任意のクラスを動的にインスタンス化させることが出来るという仕組みです。

```php
class Container
{
    public function get(string $id)
    {
        if (isset($this->definitions[$id])) { // $definitionsに$idの定義が存在するか確認している
            $className = $this->definitions[$id];
            // 定義があればそのクラスをインスタンス化。
            // $className が 'Foo' という文字列であれば new Foo() と同じ意味になる
            return new $className();
        }

        return null; // なければ何もできないので null を返却
    }
}
```

本当に最小限の機能だけであればこの程度の実装でもギリギリ DI コンテナといえるのではないでしょうか。（本当か？）

# Autowiring

このままで終わってしまうといわゆる完全に理解した状態には到達できないと思われるので、次は Autowiring を実装してみたいと思います。

Autowiring というのは、依存の依存、そのまた依存のようにネストした依存関係を自動で解決してくれる機能で、多くの DI コンテナに実装されています。
![](https://storage.googleapis.com/zenn-user-upload/30dbe9e05230-20211211.png)

例えば現状の get メソッドは、コンストラクタに何も渡さなくてもインスタンス化できるクラスにしか対応しておらず、もし以下のように Foo クラスが Bar クラスに依存していたら、この get メソッドはエラーを吐いて止まってしまいます。

```php
class Bar {}

class Foo
{
    public $bar;

    public function __construct(Bar $bar)
    {
        $this->bar = $bar;
    }
}
```

```bash
$ php src/Container.php

Fatal error: Uncaught ArgumentCountError: Too few arguments to function Foo::__construct(), 0 passed
...
```

Autowiring を実装することで、ちょっと複雑になった Foo クラスをインスタンス化できるようにしましょう。

## Autowiring の実装

では、Autowiring というのはどのように実現できるのか見ていきましょう。

今回は[Reflection API](https://www.php.net/manual/ja/book.reflection.php)を使って Autowiring を実現したいと思います。
Reflection API というのは、メソッドにどんな引数が定義されているのかといった情報やクラスに定義されているプロパティの情報、PHPDoc の内容 など、様々な情報にアクセスできるすごい奴です。

例えば [ReflectionClass::getConstructor()](https://www.php.net/manual/ja/reflectionclass.getconstructor.php) を使うと、任意のクラスのコンストラクタの引数などを調べることができます。

コンストラクタの引数を調べることが出来るのであれば、どうにかがんばれば必要な引数を自動で用意することが出来そうな気がしますね？

ということでかなり雑ですが実装してみると以下のような感じになります。

```php
<?php

class Container
{
    /** @var array<string, string> */
    private $definitions = [];

    public function define(string $id, string $className): void
    {
        $this->definitions[$id] = $className;
    }

    public function get(string $id)
    {
        $className = $this->definitions[$id] ?? $id;

        $ref = new ReflectionClass($className);
        $constructor = $ref->getConstructor();
        if ($constructor === null) {
            return new $className();
        }

        $args = [];
        foreach ($constructor->getParameters() as $parameter) {
            $type = $parameter->getType();
            $args[] = $this->get($type->getName());
        }

        return new $className(...$args);
    }
}
class Bar {}

class Foo {
    public $bar;

    public function __construct(Bar $bar)
    {
        $this->bar = $bar;
    }
}

$container = new Container();

$container->define('foo', Foo::class);

var_dump($container->get('foo'));
```

試しに実行してみると、ちゃんと依存の増えた Foo クラスをインスタンス化できているのが確認できると思います。

```bash
$ php src/Container.php
object(Foo)#6 (1) {
  ["bar"]=>
  object(Bar)#7 (0) {
  }
}
```

## 実装の確認

さて、Autowiring に対応したところ、get メソッドが少し複雑になってしまったので一つ一つ確認してみましょう。

### ReflectionClass インスタンスの生成

```php
class Container
{
    public function get(string $id)
    {
        $className = $this->definitions[$id] ?? $id;

        $ref = new ReflectionClass($className);

        // ...
    }
}
```

まず最初に、定義されている ID であればそれに紐づいたクラス名を、そうでない場合は `$id` 自体をクラス名として扱い、そのクラス名を元に ReflectionClass クラスのインスタンスを生成しています。
(`??` の挙動がわからない方は[Null 合体演算子](https://www.php.net/manual/ja/language.operators.comparison.php#language.operators.comparison.coalesce) を調べてみてください。)

### Constructor の取得

```php
class Container
{
    public function get(string $id)
    {
        // ...

        $ref = new ReflectionClass($className);
        $constructor = $ref->getConstructor();
        if ($constructor === null) {
            return new $className();
        }

        // ...
    }
}
```

`ReflectionClass` をインスタンス化した後は、`getConstructor` メソッドを実行することでそのクラスのコンストラクタを取得します。
もしコンストラクタが存在しない場合は、そのクラスは引数なしでインスタンス化できるということなので、`new $className()` とすることでインスタンスを返却しています。

### コンストラクタの引数の取得

```php
class Container
{
    public function get(string $id)
    {
        // ...

        $args = [];
        foreach ($constructor->getParameters() as $parameter) {
            $type = $parameter->getType();
            $args[] = $this->get($type->getName());
        }

        // ...
    }
}
```

コンストラクタが存在する場合は、 `ReflectionMethod::getParameters()` を実行することでコンストラクタの引数の情報(`ReflectionParameter`)を全て取得し、一つ一つ foreach で回し、 `$args` 変数にコンストラクタへ渡すための値を追加していきます。

`ReflectionMethod::getParameters()` で取得した`ReflectionParameter`クラスのインスタンスたちには `getType()` というメソッドがあり、これを実行することでその引数の型情報(`ReflectionNamedType`)を取得することができ、取得した `ReflectionNamedType` の `getName()` メソッドを実行することで型の名前、つまりクラス名や `"array"` といった文字列を取得することができます。

今回の実装ではコンストラクタの引数の型がクラス以外だった場合のことは考えないこととするため、`ReflectionNamedType::getName()` つまりクラス名をそのまま `$this->get()` メソッドに渡します。
このように関数が自分自身を呼び出す関数を再帰関数と呼びます。慣れないと難しく感じるかもしれませんが、そのうち慣れるのでがんばりましょう。

### インスタンスの返却

```php
class Container
{
    public function get(string $id)
    {
        // ...

        return new $className(...$args);
    }
}
```

コンストラクタに渡す引数が揃ったら[... トークン](https://www.php.net/manual/ja/functions.arguments.php#functions.variable-arg-list) を使って `$args` の中身を展開してコンストラクタに渡し、インスタンスを返却して終わりです。

### Foo クラスの例

`Foo` クラスを例に、より具体的に流れを追ってみましょう。

`Foo` クラス はコンストラクタを持っているためコンストラクタの引数を用意する必要があります。
`Foo` クラスのコンストラクタは Bar クラスを受け取る引数が 1 つだけ定義されているため、`$constructor->getParameters()` を実行すると `ReflectionParameter` クラスのインスタンスが 1 つ入った配列を取得できます。
そうして取得した `ReflectionParameter` から型情報を抜き出し、 `$type->getName()` をすると、`Bar` クラスのクラス名、つまり `"Bar"` という文字列を取得できるので、その値で再度 `Container::get()` メソッドを呼び出します。

```php
class Container
{
    public function get(string $id)
    {
        // ...

        $args = [];
        foreach ($constructor->getParameters() as $parameter) {
            $type = $parameter->getType();
            $args[] = $this->get($type->getName()); // $this->get('Bar') のような呼び出しになる
        }

        // ...
    }
}
```

再帰呼び出しされたため`Container::get()` メソッドの先頭に戻ります。
`Bar` クラスはコンストラクタが存在しないため `$constructor === null` が `True` となり、 `return new $className();` で`Bar` クラスのインスタンスを返します。

```php
class Container
{
    public function get(string $id)
    {
        $className = $this->definitions[$id] ?? $id; // $className = 'Bar'

        $ref = new ReflectionClass($className);
        $constructor = $ref->getConstructor();
        if ($constructor === null) {
            return new $className(); // return new Bar();
        }

        // ...
    }
}
```

`Bar` クラスのインスタンス化が済んだので呼び出し元に戻り、`Bar` クラスのインスタンスを `$args` に追加し、 `$args` を `Foo` クラスのコンストラクタに渡し、無事 `Foo` クラスのインスタンスを返すことができるという具合です。

```php
class Container
{
    public function get(string $id)
    {
        // ...

        $args = [];
        foreach ($constructor->getParameters() as $parameter) {
            $type = $parameter->getType();
            $args[] = $this->get($type->getName()); // $args = [Bar]
        }

        return new $className(...$args);
    }
}
```

これで一通り解説できました。お疲れ様でした！

# おわりに

この記事では超簡単な実装から初めて、素朴な Autowiring ができるところまでやってみました。
DI コンテナが単なるブラックボックスではなくもう少し身近に感じることができたなら幸いです。

より本格的な DI コンテナの実装について知りたい人は [capsulephp/di](https://github.com/capsulephp/di) などを読んでみるのがオススメです。小さめかつコードがとても綺麗で読みやすいです。
多機能な DI コンテナについては [Ray.Di](https://github.com/ray-di/Ray.Di) や [PHP-DI](https://github.com/PHP-DI/PHP-DI) あたりがオススメです。

# 蛇足

## Laravel の Controller と DI コンテナ

Laravel の Controller は、メソッドの引数に何かしらのクラスを指定すると勝手にそのクラスのインスタンスが渡ってきますよね？
あれも今回やったことと同じようなもので、Reflection API を使ってメソッドの引数を読み取ってなんやかんやしているわけですね。

## Laravel の Facade

Facade は DI コンテナのおかげで成り立っている仕組みですね。
`Facade::getFacadeAccessor()` で返している値を元に DI コンテナからインスタンスを取得して、そのインスタンスに実際の処理を任せている感じです。

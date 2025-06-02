# Rust構文チュートリアル：PHP開発者向け

## はじめに

このチュートリアルでは、PHP開発者がRustの構文を理解しやすいように、両言語の類似点と相違点を中心に解説します。PHPとLaravelの知識をお持ちの方が、Rustの基本的な構文を効率的に学べるように構成しています。

## 1. 基本的な構文の違い

### 1.1 コンパイル vs インタープリタ

**PHP**:
```php
<?php
echo "Hello, World!";
// 実行するだけでOK
```

**Rust**:
```rust
fn main() {
    println!("Hello, World!");
}
// コンパイルが必要
// $ rustc main.rs
// $ ./main
```

PHPはインタープリタ言語であり、コードを書いてすぐに実行できます。一方、Rustはコンパイル言語であり、コードを実行する前にコンパイルする必要があります。

### 1.2 プロジェクト管理

**PHP (Composer)**:
```bash
composer init
composer require some/package
```

**Rust (Cargo)**:
```bash
cargo new my_project
cargo add some_crate
cargo run  # コンパイルと実行を一度に行う
```

PHPではComposerを使用してパッケージを管理しますが、RustではCargoというツールを使用します。Cargoは、パッケージ管理だけでなく、ビルド、テスト、ドキュメント生成なども行います。

## 2. 変数と型

### 2.1 変数宣言

**PHP**:
```php
$x = 5;  // 変数は$で始まる
$x = "文字列に変更可能";  // 型は動的に変更可能
```

**Rust**:
```rust
let x = 5;  // 型推論により、xはi32型（32ビット整数）と推論される
// x = "文字列に変更不可能";  // エラー：型を変更できない
let mut y = 5;  // mutキーワードで可変変数を宣言
y = 10;  // 可変変数は値を変更できる（ただし型は変更不可）
```

Rustでは、変数はデフォルトで不変（イミュータブル）です。値を変更するには`mut`キーワードを使用する必要があります。また、一度型が決まると、その変数の型を変更することはできません。

### 2.2 型アノテーション

**PHP**:
```php
// PHP 7以降のタイプヒント
function add(int $a, int $b): int {
    return $a + $b;
}
```

**Rust**:
```rust
fn add(a: i32, b: i32) -> i32 {
    a + b  // 最後のセミコロンを省略すると、その式が戻り値になる
}
```

Rustでは、変数や関数の戻り値の型を明示的に指定できます。型推論が可能な場合は省略できますが、明示的に指定することでコードの可読性が向上します。

### 2.3 基本的なデータ型

**PHP**:
```php
$integer = 42;
$float = 3.14;
$string = "Hello";
$boolean = true;
$array = [1, 2, 3];
$associative_array = ["name" => "John", "age" => 30];
$null = null;
```

**Rust**:
```rust
let integer: i32 = 42;  // 32ビット整数
let float: f64 = 3.14;  // 64ビット浮動小数点数
let string: String = String::from("Hello");  // 所有権のある文字列
let str_slice: &str = "Hello";  // 文字列スライス（参照）
let boolean: bool = true;  // ブール値
let array: [i32; 3] = [1, 2, 3];  // 固定長配列
let vector: Vec<i32> = vec![1, 2, 3];  // 可変長配列（ベクター）
let tuple: (i32, &str) = (42, "Hello");  // タプル
let option: Option<i32> = Some(42);  // Option型（値があるかもしれないし、ないかもしれない）
let result: Result<i32, &str> = Ok(42);  // Result型（成功または失敗）
```

Rustには、PHPよりも多くの基本データ型があります。特に、`Option`型と`Result`型は、Rustのエラー処理において重要な役割を果たします。

## 3. 制御構造

### 3.1 条件分岐

**PHP**:
```php
if ($x > 5) {
    echo "xは5より大きい";
} elseif ($x > 0) {
    echo "xは0より大きく5以下";
} else {
    echo "xは0以下";
}

// 三項演算子
$result = $x > 5 ? "大きい" : "小さい";
```

**Rust**:
```rust
if x > 5 {
    println!("xは5より大きい");
} else if x > 0 {
    println!("xは0より大きく5以下");
} else {
    println!("xは0以下");
}

// if式（三項演算子の代わり）
let result = if x > 5 { "大きい" } else { "小さい" };
```

Rustの条件分岐はPHPと似ていますが、条件式の周りに括弧が不要です。また、Rustでは`if`は式であり、値を返すことができます。

### 3.2 ループ

**PHP**:
```php
// for
for ($i = 0; $i < 10; $i++) {
    echo $i;
}

// while
$i = 0;
while ($i < 10) {
    echo $i;
    $i++;
}

// foreach
$array = [1, 2, 3];
foreach ($array as $value) {
    echo $value;
}
foreach ($array as $key => $value) {
    echo "$key: $value";
}
```

**Rust**:
```rust
// for（イテレータを使用）
for i in 0..10 {  // 0から9まで
    println!("{}", i);
}

// while
let mut i = 0;
while i < 10 {
    println!("{}", i);
    i += 1;
}

// loop（無限ループ）
let mut i = 0;
loop {
    println!("{}", i);
    i += 1;
    if i >= 10 {
        break;
    }
}

// イテレータを使用したfor
let array = [1, 2, 3];
for value in array.iter() {
    println!("{}", value);
}
for (index, value) in array.iter().enumerate() {
    println!("{}: {}", index, value);
}
```

Rustのループ構造はPHPと似ていますが、`for`ループはイテレータを使用します。また、Rustには無限ループを簡単に書ける`loop`キーワードがあります。

### 3.3 パターンマッチング

**PHP**:
```php
switch ($value) {
    case 1:
        echo "1です";
        break;
    case 2:
    case 3:
        echo "2または3です";
        break;
    default:
        echo "その他の値です";
}
```

**Rust**:
```rust
match value {
    1 => println!("1です"),
    2 | 3 => println!("2または3です"),
    4..=10 => println!("4から10までの値です"),
    _ => println!("その他の値です"),
}
```

Rustの`match`式はPHPの`switch`文よりも強力です。パターンマッチングを使用して、値の範囲やタプル、構造体などの複雑なデータ構造にも対応できます。また、すべての可能なケースを網羅する必要があります（`_`はワイルドカードパターン）。

## 4. 関数とクロージャ

### 4.1 関数定義

**PHP**:
```php
function add($a, $b) {
    return $a + $b;
}

// PHP 7以降のタイプヒント
function add_typed(int $a, int $b): int {
    return $a + $b;
}
```

**Rust**:
```rust
fn add(a: i32, b: i32) -> i32 {
    a + b  // 最後のセミコロンを省略すると、その式が戻り値になる
}

// 明示的なreturn
fn add_with_return(a: i32, b: i32) -> i32 {
    return a + b;  // 明示的なreturnも使用可能
}
```

Rustでは、関数の引数と戻り値の型を明示的に指定する必要があります。また、最後の式のセミコロンを省略することで、その式の値が関数の戻り値になります。

### 4.2 クロージャ（無名関数）

**PHP**:
```php
// 無名関数
$add = function($a, $b) {
    return $a + $b;
};
echo $add(1, 2);  // 3

// アロー関数（PHP 7.4以降）
$multiply = fn($a, $b) => $a * $b;
echo $multiply(2, 3);  // 6
```

**Rust**:
```rust
// クロージャ
let add = |a, b| {
    a + b
};
println!("{}", add(1, 2));  // 3

// 単一式のクロージャ
let multiply = |a, b| a * b;
println!("{}", multiply(2, 3));  // 6
```

Rustのクロージャは、PHPの無名関数やアロー関数に似ています。クロージャは、周囲の変数を捕捉（キャプチャ）することもできます。

## 5. 構造体とメソッド

### 5.1 クラスと構造体

**PHP**:
```php
class Person {
    public string $name;
    public int $age;
    
    public function __construct(string $name, int $age) {
        $this->name = $name;
        $this->age = $age;
    }
    
    public function greet(): string {
        return "こんにちは、{$this->name}さん！";
    }
}

$person = new Person("田中", 30);
echo $person->greet();  // こんにちは、田中さん！
```

**Rust**:
```rust
struct Person {
    name: String,
    age: u32,
}

impl Person {
    // 関連関数（静的メソッド）
    fn new(name: String, age: u32) -> Person {
        Person { name, age }
    }
    
    // メソッド（第一引数が&self）
    fn greet(&self) -> String {
        format!("こんにちは、{}さん！", self.name)
    }
}

fn main() {
    let person = Person::new(String::from("田中"), 30);
    println!("{}", person.greet());  // こんにちは、田中さん！
}
```

Rustには、PHPのようなクラスはありませんが、構造体（`struct`）とその実装（`impl`）を使用して、オブジェクト指向プログラミングに似た機能を実現できます。

### 5.2 トレイト（インターフェース）

**PHP**:
```php
interface Greeter {
    public function greet(): string;
}

class EnglishGreeter implements Greeter {
    public function greet(): string {
        return "Hello!";
    }
}

class JapaneseGreeter implements Greeter {
    public function greet(): string {
        return "こんにちは！";
    }
}
```

**Rust**:
```rust
trait Greeter {
    fn greet(&self) -> String;
}

struct EnglishGreeter;

impl Greeter for EnglishGreeter {
    fn greet(&self) -> String {
        String::from("Hello!")
    }
}

struct JapaneseGreeter;

impl Greeter for JapaneseGreeter {
    fn greet(&self) -> String {
        String::from("こんにちは！")
    }
}
```

Rustのトレイトは、PHPのインターフェースに似ています。トレイトは、型が実装すべきメソッドのセットを定義します。

## 6. エラー処理

### 6.1 例外処理

**PHP**:
```php
try {
    $result = some_function_that_might_throw();
    echo $result;
} catch (Exception $e) {
    echo "エラーが発生しました: " . $e->getMessage();
} finally {
    echo "常に実行されます";
}
```

**Rust**:
```rust
// Rustには例外がありません。代わりにResult型を使用します
fn some_function_that_might_fail() -> Result<String, String> {
    // 成功した場合
    Ok(String::from("成功しました"))
    // 失敗した場合
    // Err(String::from("エラーが発生しました"))
}

fn main() {
    match some_function_that_might_fail() {
        Ok(result) => println!("{}", result),
        Err(error) => println!("エラーが発生しました: {}", error),
    }
    
    // より簡潔な方法
    if let Ok(result) = some_function_that_might_fail() {
        println!("{}", result);
    }
    
    // ?演算子を使用した方法（関数内で使用）
    fn process() -> Result<(), String> {
        let result = some_function_that_might_fail()?;  // エラーの場合は早期リターン
        println!("{}", result);
        Ok(())
    }
}
```

Rustには例外がありません。代わりに、`Result<T, E>`型を使用してエラーを表現します。これにより、エラー処理が明示的になり、コンパイル時にエラーハンドリングの漏れを検出できます。

### 6.2 Nullの扱い

**PHP**:
```php
$value = null;
if ($value === null) {
    echo "値はnullです";
}

// PHP 7以降のNull合体演算子
$result = $value ?? "デフォルト値";
```

**Rust**:
```rust
// Rustにはnullがありません。代わりにOption型を使用します
let value: Option<i32> = None;
match value {
    Some(v) => println!("値は{}です", v),
    None => println!("値はありません"),
}

// より簡潔な方法
if let Some(v) = value {
    println!("値は{}です", v);
} else {
    println!("値はありません");
}

// unwrap_or()を使用した方法（PHPのNull合体演算子に相当）
let result = value.unwrap_or(42);  // valueがNoneの場合は42を使用
```

Rustには`null`がありません。代わりに、`Option<T>`型を使用して、値が存在するかどうかを表現します。これにより、「nullポインタ例外」のような問題を防ぐことができます。

## 7. モジュールとパッケージ

### 7.1 モジュール化

**PHP**:
```php
// math.php
<?php
namespace Math;

function add($a, $b) {
    return $a + $b;
}

// main.php
<?php
require_once 'math.php';

echo Math\add(1, 2);  // 3

// または
use Math\add;
echo add(1, 2);  // 3
```

**Rust**:
```rust
// src/math.rs
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

// src/lib.rs
pub mod math;  // mathモジュールを公開

// src/02.rs
use rust_examples::math;
fn main() {
    println!("{}", math::add(1, 2));  // 3
    
    // または
    use math::add;
    println!("{}", add(1, 2));  // 3
}

// または
use rust_examplets::math::add;
fn main() {
    println!("{}", add(1, 2));
}
```

Rustのモジュールシステムは、PHPの名前空間に似ています。`mod`キーワードを使用してモジュールを定義し、`use`キーワードを使用して特定の要素をインポートします。

### 7.2 パッケージ管理

**PHP (Composer)**:
```json
// composer.json
{
    "require": {
        "vendor/package": "^1.0"
    }
}
```

**Rust (Cargo)**:
```toml
# Cargo.toml
[dependencies]
some_crate = "1.0"
```

RustのCargoは、PHPのComposerに相当するパッケージマネージャです。`Cargo.toml`ファイルに依存関係を記述し、`cargo build`コマンドでパッケージをダウンロードしてビルドします。

## 8. 非同期プログラミング

### 8.1 非同期処理

**PHP**:
```php
// PHP 8.1以降のFiberを使用した例
$fiber = new Fiber(function() {
    echo "Fiber開始\n";
    Fiber::suspend("一時停止");
    echo "Fiber再開\n";
    return "完了";
});

echo "Fiber実行前\n";
$value = $fiber->start();
echo "Fiberから受け取った値: $value\n";
$value = $fiber->resume();
echo "Fiberから受け取った最終値: $value\n";
```

**Rust (async/await)**:
```rust
use tokio::time::{sleep, Duration};

async fn async_task() -> String {
    println!("非同期タスク開始");
    sleep(Duration::from_secs(1)).await;
    println!("非同期タスク完了");
    String::from("完了")
}

#[tokio::main]
async fn main() {
    println!("メイン関数開始");
    let result = async_task().await;
    println!("非同期タスクから受け取った値: {}", result);
}
```

Rustの非同期プログラミングは、`async`/`await`構文を使用します。PHPの非同期プログラミングは比較的新しい機能ですが、Rustでは非同期プログラミングが言語レベルでサポートされています。

## 9. PHPとRustの主な違いのまとめ

| 機能 | PHP | Rust |
|------|-----|------|
| 実行方法 | インタープリタ | コンパイラ |
| 型システム | 動的型付け（型ヒントあり） | 静的型付け |
| メモリ管理 | ガベージコレクション | 所有権システム |
| 変数 | 可変 | デフォルトで不変 |
| null | あり | なし（Option型を使用） |
| 例外処理 | try/catch | Result型 |
| クラス | あり | なし（構造体とトレイトを使用） |
| 並行処理 | 限定的 | 言語レベルでサポート |
| パッケージ管理 | Composer | Cargo |

## 10. 次のステップ

このチュートリアルでは、PHPとRustの構文の違いについて学びました。次のステップでは、実際のコード例を通じて、これらの概念をより深く理解していきましょう。

Rustの学習を続けるためのリソース：
- [The Rust Programming Language](https://doc.rust-lang.org/book/)（公式ドキュメント）
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)（例を通じて学ぶ）
- [Rustlings](https://github.com/rust-lang/rustlings)（小さな演習問題）

次のセクションでは、実践的な例題と演習を通じて、Rustの構文をさらに深く学んでいきます。

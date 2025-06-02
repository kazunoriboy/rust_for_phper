# Rustのメモリ管理：PHP開発者のための概念ガイド

## はじめに

PHPからRustへの移行において、最も大きな違いの一つがメモリ管理の方法です。PHPではガベージコレクションが自動的にメモリを管理してくれるため、開発者がメモリについて深く考える必要はありません。一方、Rustはガベージコレクションを使用せず、独自の「所有権（Ownership）」システムを採用しています。

このガイドでは、C/C++などのメモリを直接管理する言語の経験がない方向けに、Rustのメモリ管理の概念を詳しく解説します。

## 1. メモリ管理の基本概念

### 1.1 スタックとヒープ

メモリ管理を理解するには、まず「スタック」と「ヒープ」という2種類のメモリ領域について知る必要があります。

**スタック（Stack）**:
- 高速なメモリ領域
- サイズが固定されたデータを格納
- LIFO（Last In, First Out）方式で動作
- コンパイル時にサイズが決まるデータ（整数、浮動小数点数、ブール値など）

**ヒープ（Heap）**:
- より柔軟だが、スタックより遅いメモリ領域
- 実行時にサイズが変わる可能性のあるデータを格納
- メモリの確保と解放を明示的に行う必要がある
- 文字列、配列、オブジェクトなどの可変サイズのデータ

PHPでは、これらの違いを意識する必要はほとんどありませんでしたが、Rustではこの区別が重要になります。

### 1.2 PHPとRustのメモリ管理の違い

**PHP**:
```php
$str1 = "Hello";
$str2 = $str1;
$str2 .= ", World!";

echo $str1; // "Hello" - 元の変数は変更されない
echo $str2; // "Hello, World!" - 新しい変数のみ変更される
```

PHPでは、変数を別の変数に代入すると、値のコピーが作成されます（または参照が共有されますが、それは明示的に指定した場合のみ）。

**Rust**:
```rust
let str1 = String::from("Hello");
let str2 = str1;
// ここでstr1は使えなくなる（所有権がstr2に移動した）

println!("{}", str1); // エラー: str1の値はすでに移動されている
println!("{}", str2); // "Hello"
```

Rustでは、変数を別の変数に代入すると、所有権が移動し、元の変数は無効になります。これがRustの所有権システムの基本的な動作です。

## 2. Rustの所有権システム

### 2.1 所有権（Ownership）の基本ルール

Rustの所有権システムは、以下の3つの基本ルールに基づいています：

1. Rustの各値は、「所有者（owner）」と呼ばれる変数を持つ
2. 一度に所有者は1つだけ
3. 所有者がスコープから外れると、値は自動的に破棄される

これらのルールにより、メモリリークやダングリングポインタなどの問題を防ぎます。

### 2.2 所有権の移動（Move）

PHPでは、変数を別の変数に代入すると、値のコピーが作成されるか、参照が共有されます。Rustでは、デフォルトでは「移動（Move）」が発生します。

```rust
let s1 = String::from("hello");
let s2 = s1; // s1の所有権がs2に移動

// println!("{}", s1); // エラー: s1の値はすでに移動されている
println!("{}", s2); // "hello"
```

この例では、`s1`の所有権が`s2`に移動し、`s1`は無効になります。これにより、同じメモリ領域に対して複数の変数が変更を加えることによる問題を防ぎます。

### 2.3 クローン（Clone）

値のコピーが必要な場合は、明示的に`clone`メソッドを使用します：

```rust
let s1 = String::from("hello");
let s2 = s1.clone(); // s1の値のディープコピーを作成

println!("{}", s1); // "hello" - s1はまだ有効
println!("{}", s2); // "hello" - s2は別のメモリ領域にある同じ値
```

`clone`を使用すると、ヒープ上のデータも含めて完全にコピーされます。これはPHPの変数のコピーに近い動作ですが、明示的に行う必要があります。

### 2.4 スタックのみのデータのコピー

整数や浮動小数点数などのスタックにのみ存在するシンプルなデータ型は、`Copy`トレイトを実装しています。これらの型は、代入時に自動的にコピーされます：

```rust
let x = 5;
let y = x; // xの値がyにコピーされる

println!("{}", x); // 5 - xはまだ有効
println!("{}", y); // 5
```

## 3. 参照と借用（References and Borrowing）

### 3.1 参照の基本

所有権を移動せずに値を使用するには、「参照（Reference）」を使用します。これは「借用（Borrowing）」とも呼ばれます：

```rust
fn main() {
    let s1 = String::from("hello");
    
    // &s1で参照を作成し、所有権を移動せずに関数に渡す
    let len = calculate_length(&s1);
    
    println!("The length of '{}' is {}.", s1, len); // s1はまだ有効
}

fn calculate_length(s: &String) -> usize {
    s.len()
} // sはスコープ外になるが、参照しているだけなので何も起きない
```

PHPの例で考えると、これは以下のようなコードに相当します：

```php
function calculateLength($str) {
    return strlen($str);
}

$s1 = "hello";
$len = calculateLength($s1); // $s1の値は変更されない
```

### 3.2 可変参照（Mutable References）

デフォルトでは、参照は不変（イミュータブル）です。値を変更するには、可変参照（ミュータブルリファレンス）を使用します：

```rust
fn main() {
    let mut s = String::from("hello");
    
    change(&mut s); // 可変参照を渡す
    
    println!("{}", s); // "hello, world"
}

fn change(s: &mut String) {
    s.push_str(", world");
}
```

PHPでは、関数に渡した変数を変更するには、参照渡しを使用します：

```php
function change(&$str) {
    $str .= ", world";
}

$s = "hello";
change($s);
echo $s; // "hello, world"
```

### 3.3 参照のルール

Rustの参照には、以下の重要なルールがあります：

1. 任意の時点で、**1つの可変参照**または**任意の数の不変参照**を持つことができる（両方は不可）
2. 参照は常に有効でなければならない（ダングリングポインタは許可されない）

これらのルールにより、データ競合やその他のメモリ安全性の問題を防ぎます。

```rust
let mut s = String::from("hello");

let r1 = &s; // 不変参照
let r2 = &s; // 不変参照 - OK
// let r3 = &mut s; // エラー: 不変参照と可変参照を同時に持つことはできない

println!("{} and {}", r1, r2);
// r1とr2はここで最後に使用される

let r3 = &mut s; // OK - r1とr2はもう使用されない
println!("{}", r3);
```

## 4. スライス（Slice）

スライスは、コレクションの一部への参照です。所有権を持たず、コレクションの一部を参照するだけです。

```rust
let s = String::from("hello world");

let hello = &s[0..5]; // "hello"への参照
let world = &s[6..11]; // "world"への参照

println!("{} {}", hello, world); // "hello world"
```

文字列スライスは`&str`型で表されます。これは、文字列リテラルの型でもあります：

```rust
let s: &str = "Hello, world!"; // 文字列リテラル
```

PHPでは、部分文字列を取得するには`substr`関数を使用します：

```php
$s = "hello world";
$hello = substr($s, 0, 5); // "hello"
$world = substr($s, 6, 5); // "world"
```

ただし、PHPの`substr`は新しい文字列を作成しますが、Rustのスライスは元の文字列への参照です。

## 5. ライフタイム（Lifetime）

ライフタイムは、参照が有効である期間を指定するRustの機能です。多くの場合、ライフタイムは暗黙的ですが、複雑な状況では明示的に指定する必要があります。

```rust
// 'a はライフタイムパラメータ
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

この関数は、2つの文字列スライスを受け取り、長い方を返します。ライフタイム`'a`は、返される参照が少なくとも入力パラメータと同じ長さのライフタイムを持つことを保証します。

PHPには同等の概念がないため、これはRustを学ぶ際の新しい概念です。

## 6. 実践的な例：PHPとRustの比較

### 例1: 文字列の連結

**PHP**:
```php
function appendWorld($str) {
    return $str . " World";
}

$greeting = "Hello";
$result = appendWorld($greeting);

echo $greeting; // "Hello" - 元の変数は変更されない
echo $result;   // "Hello World"
```

**Rust**:
```rust
fn append_world(s: String) -> String {
    // sの所有権を受け取り、新しい文字列を返す
    s + " World"
}

fn main() {
    let greeting = String::from("Hello");
    let result = append_world(greeting);
    
    // println!("{}", greeting); // エラー: greetingの所有権はappend_world関数に移動した
    println!("{}", result); // "Hello World"
}
```

より良い方法（参照を使用）：

```rust
fn append_world(s: &str) -> String {
    // sの参照を受け取り、新しい文字列を返す
    format!("{} World", s)
}

fn main() {
    let greeting = String::from("Hello");
    let result = append_world(&greeting);
    
    println!("{}", greeting); // "Hello" - greetingはまだ有効
    println!("{}", result);   // "Hello World"
}
```

### 例2: 配列の操作

**PHP**:
```php
function addElement($arr) {
    $arr[] = 4;
    return $arr;
}

$numbers = [1, 2, 3];
$newNumbers = addElement($numbers);

print_r($numbers);    // [1, 2, 3] - 元の配列は変更されない
print_r($newNumbers); // [1, 2, 3, 4]
```

**Rust**:
```rust
fn add_element(mut v: Vec<i32>) -> Vec<i32> {
    v.push(4);
    v
}

fn main() {
    let numbers = vec![1, 2, 3];
    let new_numbers = add_element(numbers);
    
    // println!("{:?}", numbers); // エラー: numbersの所有権はadd_element関数に移動した
    println!("{:?}", new_numbers); // [1, 2, 3, 4]
}
```

参照を使用した方法：

```rust
fn add_element(v: &[i32]) -> Vec<i32> {
    let mut new_v = v.to_vec(); // vのコピーを作成
    new_v.push(4);
    new_v
}

fn main() {
    let numbers = vec![1, 2, 3];
    let new_numbers = add_element(&numbers);
    
    println!("{:?}", numbers);    // [1, 2, 3] - numbersはまだ有効
    println!("{:?}", new_numbers); // [1, 2, 3, 4]
}
```

## 7. メモリ管理のベストプラクティス

### 7.1 所有権を意識したコーディング

- 関数が値を「消費」する必要がない場合は、参照を使用する
- 関数が値を変更する必要がある場合は、可変参照を使用する
- 関数が値の所有権を取得する必要がある場合のみ、値を直接渡す

### 7.2 `Copy`と`Clone`の適切な使用

- 小さな値（整数、浮動小数点数など）は`Copy`トレイトを実装しているため、自動的にコピーされる
- 大きな値（文字列、ベクターなど）は`Clone`メソッドを使用して明示的にコピーする
- 不必要なクローンは避け、可能な限り参照を使用する

### 7.3 スコープを小さく保つ

- 変数のスコープを可能な限り小さく保つことで、メモリの使用を最適化する
- 変数が不要になったらすぐにスコープから外れるようにする

```rust
{
    let temp = String::from("一時的な文字列");
    // tempを使用する処理
} // tempはここでスコープから外れ、メモリが解放される

// 以降のコードでtempは使用できない
```

## 8. まとめ

Rustのメモリ管理システムは、PHPのようなガベージコレクション言語とは大きく異なります。所有権、借用、ライフタイムという概念を理解することで、メモリ安全性を保ちながら効率的なコードを書くことができます。

初めは複雑に感じるかもしれませんが、これらの概念に慣れると、メモリ管理に関する多くの問題を防ぐことができます。Rustのコンパイラは、これらのルールに違反するコードをコンパイル時に検出し、エラーメッセージを表示してくれます。

次のステップでは、これらの概念を実際のコードで実践し、Rustの構文や機能についてさらに学んでいきましょう。

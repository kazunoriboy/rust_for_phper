# Rust実践例題と演習：PHP開発者向け

## はじめに

このセクションでは、前のチュートリアルで学んだRustの概念を実践するための例題と演習を提供します。PHPからRustへの移行をスムーズにするために、PHPとRustの両方のコードを比較しながら進めていきます。

## 1. 基本的なデータ処理

### 例題1: 文字列操作

**PHP**:
```php
<?php
// 文字列の連結
$first_name = "田中";
$last_name = "太郎";
$full_name = $first_name . " " . $last_name;
echo $full_name . "\n";  // 田中 太郎

// 文字列の長さ
$length = mb_strlen($full_name);
echo "文字列の長さ: " . $length . "\n";  // 文字列の長さ: 5

// 部分文字列
$substring = mb_substr($full_name, 0, 2);
echo "部分文字列: " . $substring . "\n";  // 部分文字列: 田中
```

**Rust**:
```rust
fn main() {
    // 文字列の連結
    let first_name = "田中";
    let last_name = "太郎";
    let full_name = format!("{} {}", first_name, last_name);
    println!("{}", full_name);  // 田中 太郎

    // 文字列の長さ（Unicode文字数）
    let length = full_name.chars().count();
    println!("文字列の長さ: {}", length);  // 文字列の長さ: 5

    // 部分文字列（Unicode文字で処理）
    let substring: String = full_name.chars().take(2).collect();
    println!("部分文字列: {}", substring);  // 部分文字列: 田中
}
```

**演習1**: 以下の関数を実装してください。
1. 文字列を受け取り、その文字列を逆順にして返す関数
2. 文字列を受け取り、その文字列に含まれる特定の文字の出現回数を数える関数

### 例題2: 配列操作

**PHP**:
```php
<?php
// 配列の作成と操作
$numbers = [1, 2, 3, 4, 5];
$numbers[] = 6;  // 要素の追加
array_pop($numbers);  // 最後の要素を削除
print_r($numbers);  // [1, 2, 3, 4, 5]

// 配列の変換（各要素を2倍にする）
$doubled = array_map(function($n) {
    return $n * 2;
}, $numbers);
print_r($doubled);  // [2, 4, 6, 8, 10]

// 配列のフィルタリング（偶数のみ）
$evens = array_filter($numbers, function($n) {
    return $n % 2 == 0;
});
print_r($evens);  // [2 => 2, 4 => 4]

// 配列の集計（合計）
$sum = array_reduce($numbers, function($carry, $n) {
    return $carry + $n;
}, 0);
echo "合計: " . $sum . "\n";  // 合計: 15
```

**Rust**:
```rust
fn main() {
    // ベクターの作成と操作
    let mut numbers = vec![1, 2, 3, 4, 5];
    numbers.push(6);  // 要素の追加
    numbers.pop();  // 最後の要素を削除
    println!("{:?}", numbers);  // [1, 2, 3, 4, 5]

    // ベクターの変換（各要素を2倍にする）
    let doubled: Vec<i32> = numbers.iter().map(|&n| n * 2).collect();
    println!("{:?}", doubled);  // [2, 4, 6, 8, 10]

    // ベクターのフィルタリング（偶数のみ）
    let evens: Vec<i32> = numbers.iter().filter(|&&n| n % 2 == 0).cloned().collect();
    println!("{:?}", evens);  // [2, 4]

    // ベクターの集計（合計）
    let sum: i32 = numbers.iter().sum();
    println!("合計: {}", sum);  // 合計: 15

    // 別の方法（fold を使用）
    let sum_fold = numbers.iter().fold(0, |acc, &n| acc + n);
    println!("合計 (fold): {}", sum_fold);  // 合計 (fold): 15
}
```

**演習2**: 以下の関数を実装してください。
1. 整数のベクターを受け取り、その中の最大値と最小値を返す関数
2. 整数のベクターを受け取り、その中の要素を昇順にソートして返す関数
3. 2つの整数のベクターを受け取り、それらをマージして重複を除いた新しいベクターを返す関数

## 2. 構造体とトレイト

### 例題3: 商品管理システム

**PHP**:
```php
<?php
interface Priceable {
    public function getPrice(): float;
    public function getDiscountedPrice(float $discountRate): float;
}

class Product implements Priceable {
    private string $name;
    private float $price;
    
    public function __construct(string $name, float $price) {
        $this->name = $name;
        $this->price = $price;
    }
    
    public function getName(): string {
        return $this->name;
    }
    
    public function getPrice(): float {
        return $this->price;
    }
    
    public function getDiscountedPrice(float $discountRate): float {
        return $this->price * (1 - $discountRate);
    }
}

class DigitalProduct extends Product {
    private int $downloadLimit;
    
    public function __construct(string $name, float $price, int $downloadLimit) {
        parent::__construct($name, $price);
        $this->downloadLimit = $downloadLimit;
    }
    
    public function getDownloadLimit(): int {
        return $this->downloadLimit;
    }
}

// 使用例
$product = new Product("ノート", 100);
echo $product->getName() . ": " . $product->getPrice() . "円\n";
echo "割引後: " . $product->getDiscountedPrice(0.1) . "円\n";

$digitalProduct = new DigitalProduct("電子書籍", 1500, 3);
echo $digitalProduct->getName() . ": " . $digitalProduct->getPrice() . "円\n";
echo "割引後: " . $digitalProduct->getDiscountedPrice(0.2) . "円\n";
echo "ダウンロード制限: " . $digitalProduct->getDownloadLimit() . "回\n";
```

**Rust**:
```rust
trait Priceable {
    fn get_price(&self) -> f64;
    fn get_discounted_price(&self, discount_rate: f64) -> f64;
}

struct Product {
    name: String,
    price: f64,
}

impl Product {
    fn new(name: String, price: f64) -> Self {
        Product { name, price }
    }
    
    fn get_name(&self) -> &str {
        &self.name
    }
}

impl Priceable for Product {
    fn get_price(&self) -> f64 {
        self.price
    }
    
    fn get_discounted_price(&self, discount_rate: f64) -> f64 {
        self.price * (1.0 - discount_rate)
    }
}

struct DigitalProduct {
    product: Product,
    download_limit: u32,
}

impl DigitalProduct {
    fn new(name: String, price: f64, download_limit: u32) -> Self {
        DigitalProduct {
            product: Product::new(name, price),
            download_limit,
        }
    }
    
    fn get_name(&self) -> &str {
        self.product.get_name()
    }
    
    fn get_download_limit(&self) -> u32 {
        self.download_limit
    }
}

impl Priceable for DigitalProduct {
    fn get_price(&self) -> f64 {
        self.product.get_price()
    }
    
    fn get_discounted_price(&self, discount_rate: f64) -> f64 {
        self.product.get_discounted_price(discount_rate)
    }
}

fn main() {
    let product = Product::new(String::from("ノート"), 100.0);
    println!("{}: {}円", product.get_name(), product.get_price());
    println!("割引後: {}円", product.get_discounted_price(0.1));
    
    let digital_product = DigitalProduct::new(String::from("電子書籍"), 1500.0, 3);
    println!("{}: {}円", digital_product.get_name(), digital_product.get_price());
    println!("割引後: {}円", digital_product.get_discounted_price(0.2));
    println!("ダウンロード制限: {}回", digital_product.get_download_limit());
}
```

**演習3**: 以下の機能を追加してください。
1. `PhysicalProduct`構造体を作成し、重量（weight）と在庫数（stock）のフィールドを追加してください。
2. `Priceable`トレイトに`is_available`メソッドを追加し、商品が利用可能かどうかを返すようにしてください。
3. `ShoppingCart`構造体を作成し、複数の商品を追加して合計金額を計算できるようにしてください。

## 3. エラー処理

### 例題4: ファイル操作

**PHP**:
```php
<?php
function readFile(string $filename): string {
    try {
        if (!file_exists($filename)) {
            throw new Exception("ファイルが存在しません: " . $filename);
        }
        
        $content = file_get_contents($filename);
        if ($content === false) {
            throw new Exception("ファイルの読み込みに失敗しました: " . $filename);
        }
        
        return $content;
    } catch (Exception $e) {
        echo "エラー: " . $e->getMessage() . "\n";
        return "";
    }
}

// 使用例
$content = readFile("example.txt");
if ($content !== "") {
    echo "ファイルの内容:\n" . $content . "\n";
}
```

**Rust**:
```rust
use std::fs;
use std::io;
use std::path::Path;

fn read_file(filename: &str) -> Result<String, io::Error> {
    if !Path::new(filename).exists() {
        return Err(io::Error::new(
            io::ErrorKind::NotFound,
            format!("ファイルが存在しません: {}", filename),
        ));
    }
    
    fs::read_to_string(filename)
}

fn main() {
    match read_file("example.txt") {
        Ok(content) => {
            println!("ファイルの内容:\n{}", content);
        }
        Err(error) => {
            println!("エラー: {}", error);
        }
    }
    
    // 別の方法（?演算子を使用）
    fn process_file() -> Result<(), io::Error> {
        let content = read_file("example.txt")?;
        println!("ファイルの内容:\n{}", content);
        Ok(())
    }
    
    if let Err(error) = process_file() {
        println!("エラー: {}", error);
    }
}
```

**演習4**: 以下の関数を実装してください。
1. ファイルに文字列を書き込む関数（既存のファイルがある場合は上書き）
2. ファイルに文字列を追記する関数
3. ディレクトリ内のすべてのファイル名を取得する関数

## 4. 非同期プログラミング

### 例題5: APIリクエスト

**PHP**:
```php
<?php
function fetchData(string $url): string {
    $ch = curl_init();
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    $response = curl_exec($ch);
    
    if ($response === false) {
        throw new Exception("リクエストに失敗しました: " . curl_error($ch));
    }
    
    curl_close($ch);
    return $response;
}

try {
    $data = fetchData("https://jsonplaceholder.typicode.com/todos/1");
    echo "取得したデータ: " . $data . "\n";
} catch (Exception $e) {
    echo "エラー: " . $e->getMessage() . "\n";
}
```

**Rust**:
```rust
use reqwest;
use tokio;

async fn fetch_data(url: &str) -> Result<String, reqwest::Error> {
    let response = reqwest::get(url).await?;
    let body = response.text().await?;
    Ok(body)
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    match fetch_data("https://jsonplaceholder.typicode.com/todos/1").await {
        Ok(data) => {
            println!("取得したデータ: {}", data);
        }
        Err(error) => {
            println!("エラー: {}", error);
        }
    }
    
    Ok(())
}
```

**演習5**: 以下の機能を実装してください。
1. 複数のURLから並行してデータを取得する関数
2. 取得したJSONデータを構造体にデシリアライズする関数（serde_jsonクレートを使用）
3. 一定時間ごとに特定のURLからデータを取得し続ける関数

## 5. 実践的なミニプロジェクト

### ミニプロジェクト1: 簡易メモ帳アプリ

**要件**:
1. メモの作成、読み取り、更新、削除（CRUD操作）ができる
2. メモはファイルシステムに保存される
3. コマンドライン引数で操作を指定できる

**PHP実装の例**:
```php
<?php
// memo.php

function createMemo(string $title, string $content): bool {
    $filename = "memos/" . str_replace(" ", "_", $title) . ".txt";
    
    if (file_exists($filename)) {
        echo "エラー: 同じタイトルのメモが既に存在します。\n";
        return false;
    }
    
    if (!is_dir("memos")) {
        mkdir("memos");
    }
    
    $result = file_put_contents($filename, $content);
    
    if ($result === false) {
        echo "エラー: メモの作成に失敗しました。\n";
        return false;
    }
    
    echo "メモを作成しました: " . $title . "\n";
    return true;
}

function readMemo(string $title): ?string {
    $filename = "memos/" . str_replace(" ", "_", $title) . ".txt";
    
    if (!file_exists($filename)) {
        echo "エラー: 指定されたメモが見つかりません。\n";
        return null;
    }
    
    $content = file_get_contents($filename);
    
    if ($content === false) {
        echo "エラー: メモの読み込みに失敗しました。\n";
        return null;
    }
    
    return $content;
}

function updateMemo(string $title, string $content): bool {
    $filename = "memos/" . str_replace(" ", "_", $title) . ".txt";
    
    if (!file_exists($filename)) {
        echo "エラー: 指定されたメモが見つかりません。\n";
        return false;
    }
    
    $result = file_put_contents($filename, $content);
    
    if ($result === false) {
        echo "エラー: メモの更新に失敗しました。\n";
        return false;
    }
    
    echo "メモを更新しました: " . $title . "\n";
    return true;
}

function deleteMemo(string $title): bool {
    $filename = "memos/" . str_replace(" ", "_", $title) . ".txt";
    
    if (!file_exists($filename)) {
        echo "エラー: 指定されたメモが見つかりません。\n";
        return false;
    }
    
    $result = unlink($filename);
    
    if ($result === false) {
        echo "エラー: メモの削除に失敗しました。\n";
        return false;
    }
    
    echo "メモを削除しました: " . $title . "\n";
    return true;
}

function listMemos(): array {
    if (!is_dir("memos")) {
        mkdir("memos");
        return [];
    }
    
    $files = scandir("memos");
    $memos = [];
    
    foreach ($files as $file) {
        if ($file !== "." && $file !== ".." && pathinfo($file, PATHINFO_EXTENSION) === "txt") {
            $memos[] = str_replace("_", " ", pathinfo($file, PATHINFO_FILENAME));
        }
    }
    
    return $memos;
}

// コマンドライン引数の処理
$command = $argv[1] ?? "";

switch ($command) {
    case "create":
        $title = $argv[2] ?? "";
        $content = $argv[3] ?? "";
        
        if (empty($title) || empty($content)) {
            echo "使用法: php memo.php create <タイトル> <内容>\n";
            exit(1);
        }
        
        createMemo($title, $content);
        break;
        
    case "read":
        $title = $argv[2] ?? "";
        
        if (empty($title)) {
            echo "使用法: php memo.php read <タイトル>\n";
            exit(1);
        }
        
        $content = readMemo($title);
        
        if ($content !== null) {
            echo "タイトル: " . $title . "\n";
            echo "内容:\n" . $content . "\n";
        }
        break;
        
    case "update":
        $title = $argv[2] ?? "";
        $content = $argv[3] ?? "";
        
        if (empty($title) || empty($content)) {
            echo "使用法: php memo.php update <タイトル> <新しい内容>\n";
            exit(1);
        }
        
        updateMemo($title, $content);
        break;
        
    case "delete":
        $title = $argv[2] ?? "";
        
        if (empty($title)) {
            echo "使用法: php memo.php delete <タイトル>\n";
            exit(1);
        }
        
        deleteMemo($title);
        break;
        
    case "list":
        $memos = listMemos();
        
        if (empty($memos)) {
            echo "メモはありません。\n";
        } else {
            echo "メモ一覧:\n";
            foreach ($memos as $memo) {
                echo "- " . $memo . "\n";
            }
        }
        break;
        
    default:
        echo "使用法: php memo.php <コマンド> [引数...]\n";
        echo "コマンド:\n";
        echo "  create <タイトル> <内容>   - 新しいメモを作成\n";
        echo "  read <タイトル>           - メモを読み取り\n";
        echo "  update <タイトル> <内容>   - メモを更新\n";
        echo "  delete <タイトル>         - メモを削除\n";
        echo "  list                     - メモ一覧を表示\n";
        break;
}
```

**Rust実装の課題**:

上記のPHP実装と同等の機能を持つRustプログラムを作成してください。以下の要件を満たす必要があります：

1. `clap`クレートを使用してコマンドライン引数を処理する
2. ファイル操作には標準ライブラリの`std::fs`モジュールを使用する
3. エラー処理には`Result`型を使用する
4. メモのタイトルと内容を保持する`Memo`構造体を定義する

**ヒント**:
- Rustでのファイル操作は`std::fs`モジュールを使用します
- ディレクトリの作成は`std::fs::create_dir_all`関数を使用します
- ファイルの読み書きは`std::fs::read_to_string`と`std::fs::write`関数を使用します
- コマンドライン引数の処理には`clap`クレートを使用します

### ミニプロジェクト2: 簡易計算機

**要件**:
1. 基本的な算術演算（加算、減算、乗算、除算）をサポート
2. コマンドライン引数で式を指定できる
3. エラー処理（ゼロ除算、無効な式など）

**PHP実装の例**:
```php
<?php
// calculator.php

function calculate(string $expression): float {
    // 安全のため、許可された文字のみを含む式かチェック
    if (!preg_match('/^[0-9\+\-\*\/\(\)\.\s]+$/', $expression)) {
        throw new Exception("無効な式です: " . $expression);
    }
    
    // evalを使用して式を評価（実際のアプリケーションではセキュリティ上の理由から避けるべき）
    $result = @eval("return " . $expression . ";");
    
    if ($result === false) {
        throw new Exception("式の評価に失敗しました: " . $expression);
    }
    
    return $result;
}

try {
    $expression = $argv[1] ?? "";
    
    if (empty($expression)) {
        echo "使用法: php calculator.php <式>\n";
        echo "例: php calculator.php \"2 + 2 * 3\"\n";
        exit(1);
    }
    
    $result = calculate($expression);
    echo $expression . " = " . $result . "\n";
} catch (Exception $e) {
    echo "エラー: " . $e->getMessage() . "\n";
    exit(1);
}
```

**Rust実装の課題**:

上記のPHP実装と同等の機能を持つRustプログラムを作成してください。以下の要件を満たす必要があります：

1. 式の解析と評価を自分で実装する（外部クレートを使用しても可）
2. エラー処理には`Result`型を使用する
3. ユーザーが式を対話的に入力できるようにする（オプション）

**ヒント**:
- 式の解析には再帰下降構文解析などの手法を使用できます
- または、`eval`クレートなどの外部ライブラリを使用することもできます
- 対話的なインターフェースには`std::io`モジュールの`stdin`を使用します

## 6. 次のステップ

これらの例題と演習を通じて、Rustの基本的な構文と概念を実践的に学ぶことができました。次のステップでは、これらの知識を活かして、より複雑なアプリケーション（TODOアプリなど）を開発していきましょう。

Rustの学習を続けるためのリソース：
- [The Rust Programming Language](https://doc.rust-lang.org/book/)（公式ドキュメント）
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)（例を通じて学ぶ）
- [Rustlings](https://github.com/rust-lang/rustlings)（小さな演習問題）
- [Rust Cookbook](https://rust-lang-nursery.github.io/rust-cookbook/)（実用的なレシピ集）

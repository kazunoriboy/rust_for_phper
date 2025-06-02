# トピック3: マクロによるメタプログラミング入門

## 学習目標

このセクションを学ぶことで、以下のことができるようになります：

- Rustの宣言的マクロ（`macro_rules!`）の基本構文を理解し、実装する
- マクロのパターンマッチングと繰り返し構文を使いこなす
- 手続き的マクロ（カスタム`derive`など）の概要と使用シーンを理解する
- マクロを使用するメリットと注意点を把握し、適切な場面で活用する

## 主要な概念の説明

### マクロとは何か

マクロは、コードを生成するコードです。Rustのマクロシステムを使用すると、コードの繰り返しを減らし、ドメイン固有言語（DSL）を作成し、インターフェースをより使いやすくすることができます。

Rustには主に2種類のマクロがあります：

1. **宣言的マクロ（Declarative Macros）**: `macro_rules!`を使用して定義され、パターンマッチングに基づいています。
2. **手続き的マクロ（Procedural Macros）**: Rustのコードを入力として受け取り、コードを出力する関数です。さらに以下の3種類に分類されます：
   - カスタム`derive`マクロ
   - 属性マクロ
   - 関数風マクロ

### 宣言的マクロ（`macro_rules!`）

宣言的マクロは、パターンマッチングシステムを使用してコードを生成します。`println!`や`vec!`などの標準ライブラリのマクロはこのタイプです。

#### 基本構文

```rust
macro_rules! マクロ名 {
    (パターン1) => {
        // 展開されるコード1
    };
    (パターン2) => {
        // 展開されるコード2
    };
    // 他のパターン...
}
```

各パターンは、マクロが呼び出されたときに一致するかどうかがチェックされます。最初に一致したパターンのコードが展開されます。

#### 簡単な例：`say_hello!`マクロ

```rust
macro_rules! say_hello {
    () => {
        println!("Hello!");
    };
    ($name:expr) => {
        println!("Hello, {}!", $name);
    };
}

fn main() {
    say_hello!(); // "Hello!"を出力
    say_hello!("Rust"); // "Hello, Rust!"を出力
}
```

この例では、`say_hello!`マクロは2つのパターンを持っています：
1. 引数なし: `Hello!`を出力
2. 式（`$name:expr`）を1つ受け取る: `Hello, {name}!`を出力

#### マクロのパターンマッチング

マクロのパターンは、以下の要素で構成されます：

1. **メタ変数**: `$name`のような、`$`で始まる識別子
2. **フラグメント指定子**: メタ変数の後に`:type`の形式で指定される型
3. **繰り返し**: `$(...)sep rep`の形式で、パターンの繰り返しを表現

主なフラグメント指定子：

- `expr`: 式
- `ident`: 識別子
- `ty`: 型
- `path`: パス（`foo`、`::std::mem::replace`など）
- `stmt`: 文
- `block`: ブロック式
- `item`: アイテム（関数、構造体など）
- `meta`: メタアイテム（属性の内側）
- `tt`: トークンツリー（単一のトークンまたは括弧で囲まれたトークン）
- `pat`: パターン

#### 繰り返しパターン

繰り返しパターンは、可変長の引数を処理するのに役立ちます：

```rust
macro_rules! vector {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}

fn main() {
    let v = vector![1, 2, 3, 4];
    println!("{:?}", v); // [1, 2, 3, 4]を出力
}
```

この例では：
- `$( $x:expr ),*`は、カンマで区切られた式のリストにマッチします
- `$(temp_vec.push($x);)*`は、マッチした各式に対して`push`操作を生成します

繰り返しの構文：
- `$(...)*`: 0回以上の繰り返し
- `$(...)+`: 1回以上の繰り返し
- `$(...),*`: カンマで区切られた0回以上の繰り返し
- `$(...);*`: セミコロンで区切られた0回以上の繰り返し

#### 複数のパターンを持つマクロ

```rust
macro_rules! math {
    (add $a:expr, $b:expr) => {
        $a + $b
    };
    (sub $a:expr, $b:expr) => {
        $a - $b
    };
    (mul $a:expr, $b:expr) => {
        $a * $b
    };
    (div $a:expr, $b:expr) => {
        $a / $b
    };
}

fn main() {
    println!("加算: {}", math!(add 5, 3)); // 8
    println!("減算: {}", math!(sub 5, 3)); // 2
    println!("乗算: {}", math!(mul 5, 3)); // 15
    println!("除算: {}", math!(div 6, 3)); // 2
}
```

この例では、マクロは操作の種類に応じて異なるコードを生成します。

#### 再帰的なマクロ

マクロは自分自身を呼び出すことができ、これにより複雑なパターンを処理できます：

```rust
macro_rules! find_min {
    ($x:expr) => ($x);
    ($x:expr, $($y:expr),+) => {
        std::cmp::min($x, find_min!($($y),+))
    };
}

fn main() {
    println!("最小値: {}", find_min!(5, 2, 8, 1, 7)); // 1
}
```

この例では、`find_min!`マクロは再帰的に最小値を計算します：
1. 単一の式の場合はその値を返す
2. 複数の式の場合は、最初の値と残りの値の最小値を比較する

### 衛生的なマクロ（Hygiene）

Rustのマクロは「衛生的」です。これは、マクロが展開されるときに、マクロ内で定義された変数が外部のコードと衝突しないことを意味します。

```rust
macro_rules! create_function {
    ($func_name:ident) => {
        fn $func_name() {
            println!("関数 {} が呼び出されました", stringify!($func_name));
        }
    };
}

create_function!(hello);

fn main() {
    hello(); // "関数 hello が呼び出されました"を出力
    
    let hello = "Hello, world!";
    println!("{}", hello); // マクロで定義した関数とは衝突しない
}
```

`stringify!`は、トークンを文字列リテラルに変換する組み込みマクロです。

### マクロのスコープとインポート

マクロは、定義されたモジュール内とその子モジュールでのみ使用できます。他のモジュールでマクロを使用するには、`#[macro_export]`属性を使用してエクスポートする必要があります：

```rust
#[macro_export]
macro_rules! say_hello {
    () => {
        println!("Hello!");
    };
}
```

エクスポートされたマクロは、クレートのルートからインポートできます：

```rust
use my_crate::say_hello;

fn main() {
    say_hello!();
}
```

### 手続き的マクロ（Procedural Macros）

手続き的マクロは、Rustのコードを入力として受け取り、コードを出力する関数です。宣言的マクロよりも強力ですが、実装も複雑です。

手続き的マクロを作成するには、専用のクレートタイプが必要です：

```toml
[lib]
proc-macro = true
```

#### カスタム`derive`マクロ

カスタム`derive`マクロは、構造体や列挙型に対して自動的にコードを生成します：

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput};

#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    // 入力トークンをRustの構文ツリーにパース
    let input = parse_macro_input!(input as DeriveInput);
    
    // 構造体または列挙型の名前を取得
    let name = input.ident;
    
    // コードを生成
    let expanded = quote! {
        impl HelloMacro for #name {
            fn hello_macro() {
                println!("Hello, Macro! My name is {}", stringify!(#name));
            }
        }
    };
    
    // 生成されたコードをトークンストリームに変換して返す
    expanded.into()
}
```

使用例：

```rust
use hello_macro::HelloMacro;
use hello_macro_derive::HelloMacro;

trait HelloMacro {
    fn hello_macro();
}

#[derive(HelloMacro)]
struct Pancakes;

fn main() {
    Pancakes::hello_macro(); // "Hello, Macro! My name is Pancakes"を出力
}
```

#### 属性マクロ

属性マクロは、任意のアイテム（関数、構造体など）に適用できる新しい属性を定義します：

```rust
#[proc_macro_attribute]
pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream {
    // attr: 属性の引数（例: "GET, /"）
    // item: 属性が付けられたアイテム（例: 関数定義）
    
    // コードを生成して返す
    // ...
}
```

使用例：

```rust
#[route(GET, "/")]
fn index() {
    // ...
}
```

#### 関数風マクロ

関数風マクロは、関数呼び出しのように見えますが、コンパイル時に展開されるマクロです：

```rust
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {
    // SQLクエリを解析してRustコードを生成
    // ...
}
```

使用例：

```rust
let users = sql!(SELECT * FROM users WHERE age > 18);
```

### マクロを使用するメリットと注意点

#### メリット

1. **コードの繰り返しを減らす**: 似たようなコードパターンを抽象化できます
2. **ドメイン固有言語（DSL）の作成**: 特定のドメインに特化した構文を提供できます
3. **コンパイル時の検証**: 型チェックなどをコンパイル時に行えます
4. **条件付きコンパイル**: 異なる環境や設定に応じてコードを生成できます
5. **メタプログラミング**: コードを生成するコードを書くことで、抽象化レベルを上げられます

#### 注意点

1. **デバッグの難しさ**: マクロのデバッグは通常のコードよりも難しいです
2. **コードの読みにくさ**: 過度にマクロを使用すると、コードが読みにくくなります
3. **コンパイル時間の増加**: 複雑なマクロはコンパイル時間を増加させます
4. **学習曲線**: マクロの構文は通常のRustコードよりも学習が難しいです
5. **IDE対応の制限**: マクロはIDEのコード補完などの機能と相性が悪いことがあります

## 具体的なコード例

### 例1: デバッグマクロの作成

デバッグ情報を出力するカスタムマクロを作成します：

```rust
macro_rules! debug {
    ($($arg:tt)*) => {
        #[cfg(debug_assertions)]
        println!("[DEBUG] {}", format!($($arg)*));
    };
}

fn main() {
    let x = 42;
    let name = "Rust";
    
    debug!("x = {}", x); // デバッグビルドでのみ出力
    debug!("name = {}, x = {}", name, x); // デバッグビルドでのみ出力
    
    println!("通常のメッセージ"); // 常に出力
}
```

このマクロは、`debug_assertions`が有効な場合（通常はデバッグビルド時）にのみメッセージを出力します。`tt`（トークンツリー）フラグメント指定子を使用して、任意のトークンシーケンスをキャプチャしています。

### 例2: ビルダーパターンの自動実装

構造体に対してビルダーパターンを自動的に実装するマクロを作成します：

```rust
macro_rules! builder {
    (
        $name:ident {
            $($field:ident: $type:ty,)*
        }
    ) => {
        // 元の構造体
        pub struct $name {
            $($field: $type,)*
        }
        
        // ビルダー構造体
        pub struct $name Builder {
            $($field: Option<$type>,)*
        }
        
        impl $name {
            // ビルダーを作成するメソッド
            pub fn builder() -> $name Builder {
                $name Builder {
                    $($field: None,)*
                }
            }
        }
        
        impl $name Builder {
            // 各フィールドのセッターメソッド
            $(
                pub fn $field(mut self, value: $type) -> Self {
                    self.$field = Some(value);
                    self
                }
            )*
            
            // ビルドメソッド
            pub fn build(self) -> Result<$name, String> {
                Ok($name {
                    $(
                        $field: self.$field.ok_or(
                            format!("Field {} is required", stringify!($field))
                        )?,
                    )*
                })
            }
        }
    };
}

// マクロを使用して構造体とビルダーを定義
builder! {
    Person {
        name: String,
        age: u32,
        email: String,
    }
}

fn main() -> Result<(), String> {
    // ビルダーパターンを使用してPersonインスタンスを作成
    let person = Person::builder()
        .name("Alice".to_string())
        .age(30)
        .email("alice@example.com".to_string())
        .build()?;
    
    println!("Person: {} ({}, {})", person.name, person.age, person.email);
    
    // 必須フィールドが欠けている場合はエラー
    let result = Person::builder()
        .name("Bob".to_string())
        .age(25)
        // emailが欠けている
        .build();
    
    match result {
        Ok(_) => println!("成功（ここには到達しないはず）"),
        Err(e) => println!("エラー: {}", e), // "エラー: Field email is required"
    }
    
    Ok(())
}
```

このマクロは、構造体の定義からビルダーパターンを自動的に生成します。各フィールドに対するセッターメソッドと、すべての必須フィールドが設定されているかをチェックするビルドメソッドが生成されます。

### 例3: SQLクエリビルダー

簡単なSQLクエリビルダーマクロを作成します：

```rust
macro_rules! select {
    ($($column:ident),*; from $table:ident $(where $condition:expr)?) => {
        {
            let mut query = format!("SELECT {} FROM {}", 
                stringify!($($column),*).replace(" ", ""), 
                stringify!($table));
            
            $(
                query.push_str(&format!(" WHERE {}", stringify!($condition)));
            )?
            
            query
        }
    };
}

fn main() {
    // 基本的なSELECTクエリ
    let query1 = select!(id, name, email; from users);
    println!("クエリ1: {}", query1);
    
    // WHERE句付きのクエリ
    let query2 = select!(id, name; from users where id > 100);
    println!("クエリ2: {}", query2);
    
    // 実際のアプリケーションでは、このようにパラメータを使用することもできます
    let min_age = 18;
    let query3 = select!(id, name, age; from users where age >= min_age);
    println!("クエリ3: {}", query3);
}
```

このマクロは、SQLのSELECT文を生成します。カラム名、テーブル名、およびオプションのWHERE句を指定できます。

### 例4: カスタムderive実装（手続き的マクロ）

カスタム`derive`マクロを使用して、構造体に対して自動的にJSONシリアライズ機能を実装する例を示します。

まず、手続き的マクロクレートを作成します（`json_derive`という名前）：

```rust
// json_derive/src/lib.rs
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, Data, DeriveInput, Fields};

#[proc_macro_derive(ToJson)]
pub fn to_json_derive(input: TokenStream) -> TokenStream {
    // 入力をパース
    let input = parse_macro_input!(input as DeriveInput);
    let name = input.ident;
    
    // フィールドを取得
    let fields = match input.data {
        Data::Struct(data) => {
            match data.fields {
                Fields::Named(fields) => fields.named,
                _ => panic!("ToJson can only be derived for structs with named fields"),
            }
        },
        _ => panic!("ToJson can only be derived for structs"),
    };
    
    // フィールド名を取得
    let field_names = fields.iter().map(|field| {
        field.ident.as_ref().unwrap()
    });
    
    // フィールド名（文字列リテラル）を取得
    let field_name_strings = fields.iter().map(|field| {
        let name = field.ident.as_ref().unwrap().to_string();
        format!("\"{}\"", name)
    });
    
    // to_json実装を生成
    let expanded = quote! {
        impl ToJson for #name {
            fn to_json(&self) -> String {
                let mut json = String::from("{");
                
                #(
                    json.push_str(&format!("{}:\"{}\",", #field_name_strings, self.#field_names));
                )*
                
                // 最後のカンマを削除
                if json.ends_with(',') {
                    json.pop();
                }
                
                json.push_str("}");
                json
            }
        }
    };
    
    expanded.into()
}
```

次に、メインクレートでこのマクロを使用します：

```rust
// main.rs
use json_derive::ToJson;

trait ToJson {
    fn to_json(&self) -> String;
}

#[derive(ToJson)]
struct Person {
    name: String,
    age: u32,
    email: String,
}

fn main() {
    let person = Person {
        name: "Alice".to_string(),
        age: 30,
        email: "alice@example.com".to_string(),
    };
    
    println!("JSON: {}", person.to_json());
    // 出力: JSON: {"name":"Alice","age":"30","email":"alice@example.com"}
}
```

この例では、`ToJson`トレイトを自動的に実装するカスタム`derive`マクロを作成しています。このマクロは、構造体のフィールドを反復処理し、JSONシリアライズコードを生成します。

## PHP/Laravel開発者向けのポイント

### PHPのリフレクションとの比較

PHPでは、リフレクションAPIを使用して実行時にクラスやメソッドの情報を取得し、動的にコードを生成することができます：

```php
<?php
// PHPのリフレクション例
class Person {
    public string $name;
    public int $age;
    public string $email;
    
    public function __construct(string $name, int $age, string $email) {
        $this->name = $name;
        $this->age = $age;
        $this->email = $email;
    }
}

function toJson($object) {
    $reflection = new ReflectionClass($object);
    $properties = $reflection->getProperties(ReflectionProperty::IS_PUBLIC);
    
    $json = [];
    foreach ($properties as $property) {
        $name = $property->getName();
        $value = $property->getValue($object);
        $json[$name] = $value;
    }
    
    return json_encode($json);
}

$person = new Person("Alice", 30, "alice@example.com");
echo toJson($person); // {"name":"Alice","age":30,"email":"alice@example.com"}
```

Rustのマクロは、PHPのリフレクションとは異なり、**コンパイル時**に動作します。これにより、実行時のオーバーヘッドがなく、型安全性が保証されます。

### Laravelのファサードやマクロ機能との対比

Laravelでは、ファサードパターンを使用して静的インターフェースを提供し、マクロ機能を使用してクラスに動的にメソッドを追加できます：

```php
<?php
// Laravelのマクロ例
use Illuminate\Support\Str;

Str::macro('toTitleCase', function ($value) {
    return mb_convert_case($value, MB_CASE_TITLE, 'UTF-8');
});

$title = Str::toTitleCase('hello world'); // "Hello World"
```

Rustのマクロは、Laravelのマクロよりも強力で柔軟ですが、構文が複雑です。Laravelのマクロは実行時に動的にメソッドを追加しますが、Rustのマクロはコンパイル時にコードを生成します。

### PHPの属性（Attributes）との比較

PHP 8.0以降では、属性（Attributes）を使用してメタデータをクラスやメソッドに追加できます：

```php
<?php
// PHP 8の属性例
#[Route("/api/users", methods: ["GET"])]
function getUsers() {
    // ...
}
```

Rustの属性マクロは、PHPの属性と同様の目的で使用されますが、より強力です。Rustの属性マクロは、属性が付けられたアイテムを変更または拡張できます。

### メタプログラミングの移行ポイント

1. **実行時 vs コンパイル時**:
   - PHPのメタプログラミングは主に実行時に行われる
   - Rustのマクロはコンパイル時に展開される

2. **型安全性**:
   - PHPのリフレクションは動的で型安全性が低い
   - Rustのマクロは静的で型安全性が高い

3. **構文の複雑さ**:
   - PHPのリフレクションやマクロは比較的シンプル
   - Rustのマクロは強力だが構文が複雑

4. **ユースケース**:
   - PHPでは主にフレームワークやライブラリの内部で使用
   - Rustでは一般的なコードでも頻繁に使用される

5. **デバッグの難しさ**:
   - PHPのリフレクションは実行時にデバッグできる
   - Rustのマクロはコンパイル時に展開されるため、デバッグが難しい

## 演習問題

### 演習1: デバッグマクロの拡張

**目標**: 先ほど紹介したデバッグマクロを拡張して、ファイル名、行番号、関数名を含めるようにする

**要件**:
1. マクロは`debug!`という名前で定義する
2. デバッグビルド時にのみメッセージを出力する
3. 出力には、ファイル名、行番号、関数名、およびユーザーが指定したメッセージを含める
4. フォーマット文字列と可変長引数をサポートする

**ヒント**:
- `file!`、`line!`、`function!`マクロを使用して、現在のファイル名、行番号、関数名を取得できる
- `cfg`属性を使用して、デバッグビルド時にのみコードを含めることができる

**スケルトンコード**:

```rust
// TODO: デバッグマクロを定義する

fn example_function() {
    let x = 42;
    debug!("x = {}", x);
}

fn main() {
    debug!("デバッグメッセージ");
    example_function();
    
    let name = "Rust";
    let version = "1.60";
    debug!("言語: {}, バージョン: {}", name, version);
}
```

### 演習2: JSONシリアライザマクロの実装

**目標**: 構造体をJSONに変換するマクロを実装する

**要件**:
1. マクロは`to_json!`という名前で定義する
2. 構造体のインスタンスを受け取り、JSONフォーマットの文字列を返す
3. 基本的なデータ型（文字列、数値、ブール値）をサポートする
4. ネストされた構造体もサポートする

**ヒント**:
- 再帰的なマクロを使用して、ネストされた構造体を処理できる
- `stringify!`マクロを使用して、フィールド名を文字列に変換できる
- 型に応じて異なる処理を行うために、複数のパターンを定義する

**スケルトンコード**:

```rust
// TODO: to_json!マクロを定義する

struct Person {
    name: String,
    age: u32,
    is_active: bool,
}

struct Team {
    name: String,
    members: Vec<Person>,
}

fn main() {
    let alice = Person {
        name: "Alice".to_string(),
        age: 30,
        is_active: true,
    };
    
    let bob = Person {
        name: "Bob".to_string(),
        age: 25,
        is_active: false,
    };
    
    let team = Team {
        name: "開発チーム".to_string(),
        members: vec![alice, bob],
    };
    
    println!("Person JSON: {}", to_json!(alice));
    println!("Team JSON: {}", to_json!(team));
}
```

### 演習3: ビルダーパターンマクロの拡張

**目標**: 先ほど紹介したビルダーパターンマクロを拡張して、オプショナルなフィールドをサポートする

**要件**:
1. マクロは`builder!`という名前で定義する
2. 必須フィールドとオプショナルフィールドを区別できるようにする
3. ビルド時に必須フィールドのみをチェックする
4. オプショナルフィールドには、デフォルト値を指定できるようにする

**ヒント**:
- フィールドの定義構文を拡張して、オプショナルフィールドを示す記法を追加する
- オプショナルフィールドには、`Option<T>`型を使用する
- デフォルト値は、`build`メソッド内で`unwrap_or`を使用して適用できる

**スケルトンコード**:

```rust
// TODO: 拡張されたbuilder!マクロを定義する

// マクロを使用して構造体とビルダーを定義
builder! {
    Person {
        // 必須フィールド
        name: String,
        email: String,
        
        // オプショナルフィールド（デフォルト値付き）
        age?: u32 = 18,
        is_active?: bool = true,
    }
}

fn main() -> Result<(), String> {
    // 必須フィールドのみを指定
    let person1 = Person::builder()
        .name("Alice".to_string())
        .email("alice@example.com".to_string())
        .build()?;
    
    println!("Person1: {} ({}, {})", person1.name, person1.age, person1.email);
    
    // すべてのフィールドを指定
    let person2 = Person::builder()
        .name("Bob".to_string())
        .email("bob@example.com".to_string())
        .age(25)
        .is_active(false)
        .build()?;
    
    println!("Person2: {} ({}, {})", person2.name, person2.age, person2.email);
    
    // 必須フィールドが欠けている場合はエラー
    let result = Person::builder()
        .name("Charlie".to_string())
        // emailが欠けている
        .build();
    
    match result {
        Ok(_) => println!("成功（ここには到達しないはず）"),
        Err(e) => println!("エラー: {}", e),
    }
    
    Ok(())
}
```

これらの演習問題を通じて、Rustのマクロシステムを実践的に学ぶことができます。マクロは強力なツールですが、適切に使用するには練習が必要です。演習を通じて、マクロの構文やパターンマッチングの仕組みに慣れていきましょう。

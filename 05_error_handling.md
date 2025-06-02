# トピック1: より高度なエラー処理と堅牢なアプリケーション設計

## 学習目標

このセクションを学ぶことで、以下のことができるようになります：

- `thiserror`と`anyhow`クレートの違いを理解し、適切に使い分ける
- アプリケーションに最適なカスタムエラー型を設計・実装する
- エラーを適切に伝播させ、上位レイヤーで集約する方法を習得する
- パニックの適切な扱い方と、ライブラリ設計時の考慮点を理解する

## 主要な概念の説明

### Rustのエラー処理の復習

Rustのエラー処理は、例外を使用する言語（PHPなど）とは大きく異なります。Rustでは、エラーは値として扱われ、`Result<T, E>`型を通じて明示的に処理されます。

```rust
enum Result<T, E> {
    Ok(T),   // 成功した場合の値
    Err(E),  // エラーの場合の値
}
```

基本的なエラー処理パターンを復習しましょう：

```rust
fn read_username_from_file() -> Result<String, io::Error> {
    let mut username = String::new();
    
    let mut file = match File::open("username.txt") {
        Ok(file) => file,
        Err(e) => return Err(e),
    };
    
    match file.read_to_string(&mut username) {
        Ok(_) => Ok(username),
        Err(e) => Err(e),
    }
}
```

`?`演算子を使うと、より簡潔に書けます：

```rust
fn read_username_from_file() -> Result<String, io::Error> {
    let mut username = String::new();
    let mut file = File::open("username.txt")?;
    file.read_to_string(&mut username)?;
    Ok(username)
}
```

さらに簡潔に：

```rust
fn read_username_from_file() -> Result<String, io::Error> {
    let mut username = String::new();
    File::open("username.txt")?.read_to_string(&mut username)?;
    Ok(username)
}
```

または、標準ライブラリの関数を使って：

```rust
fn read_username_from_file() -> Result<String, io::Error> {
    fs::read_to_string("username.txt")
}
```

### エラー型の変換と伝播

実際のアプリケーションでは、複数の種類のエラーが発生します。例えば、ファイル操作、ネットワーク通信、JSONパース、バリデーションなど。これらのエラーを統一的に扱うには、エラー型の変換が必要です。

標準ライブラリでは、`From`トレイトを実装することでエラー変換ができます：

```rust
#[derive(Debug)]
enum AppError {
    IoError(io::Error),
    ParseError(serde_json::Error),
    ValidationError(String),
}

impl From<io::Error> for AppError {
    fn from(error: io::Error) -> Self {
        AppError::IoError(error)
    }
}

impl From<serde_json::Error> for AppError {
    fn from(error: serde_json::Error) -> Self {
        AppError::ParseError(error)
    }
}
```

これにより、`?`演算子を使ってエラーを自動的に変換できます：

```rust
fn read_and_parse_config() -> Result<Config, AppError> {
    let config_text = fs::read_to_string("config.json")?; // io::Error → AppError
    let config: Config = serde_json::from_str(&config_text)?; // serde_json::Error → AppError
    
    if !config.is_valid() {
        return Err(AppError::ValidationError("Invalid configuration".to_string()));
    }
    
    Ok(config)
}
```

しかし、多くのエラー型を扱うアプリケーションでは、これらの変換を手動で実装するのは面倒です。ここで`thiserror`と`anyhow`クレートの出番です。

### `thiserror`によるカスタムエラー定義

`thiserror`クレートは、カスタムエラー型の定義を簡素化します。エラーメッセージの生成や`From`トレイトの実装を自動化してくれます。

まず、依存関係を追加します：

```toml
[dependencies]
thiserror = "1.0"
```

次に、カスタムエラー型を定義します：

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum AppError {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
    
    #[error("Parse error: {0}")]
    Parse(#[from] serde_json::Error),
    
    #[error("Validation error: {0}")]
    Validation(String),
    
    #[error("User not found with ID: {0}")]
    UserNotFound(u64),
    
    #[error("Database error: {source}")]
    Database {
        #[from]
        source: sqlx::Error,
        backtrace: std::backtrace::Backtrace,
    },
}
```

`#[error("...")]`属性は、エラーメッセージのフォーマットを指定します。`#[from]`属性は、`From`トレイトの実装を自動生成します。

これにより、先ほどの例は次のように書けます：

```rust
fn read_and_parse_config() -> Result<Config, AppError> {
    let config_text = fs::read_to_string("config.json")?;
    let config: Config = serde_json::from_str(&config_text)?;
    
    if !config.is_valid() {
        return Err(AppError::Validation("Invalid configuration".to_string()));
    }
    
    Ok(config)
}
```

`thiserror`は、ライブラリやアプリケーションの下位レイヤーで使用するのに適しています。エラーの種類が明確で、呼び出し元に具体的なエラー情報を提供したい場合に使用します。

### `anyhow`によるエラー集約

`anyhow`クレートは、様々な種類のエラーを単一の型で扱いたい場合に便利です。特にアプリケーションの上位レイヤーや、エラーの種類を区別する必要がない場合に適しています。

依存関係を追加します：

```toml
[dependencies]
anyhow = "1.0"
```

`anyhow`を使うと、次のように書けます：

```rust
use anyhow::{Result, Context, anyhow};

fn read_and_parse_config() -> Result<Config> {
    let config_text = std::fs::read_to_string("config.json")
        .context("Failed to read config file")?;
    
    let config: Config = serde_json::from_str(&config_text)
        .context("Failed to parse config file")?;
    
    if !config.is_valid() {
        return Err(anyhow!("Invalid configuration"));
    }
    
    Ok(config)
}
```

`anyhow::Result<T>`は`Result<T, anyhow::Error>`のエイリアスです。`anyhow::Error`は、様々なエラー型をラップできる汎用的なエラー型です。

`context`メソッドは、エラーに追加の文脈情報を付加します。これにより、エラーが発生した場所や理由を特定しやすくなります。

### `thiserror`と`anyhow`の使い分け

- **`thiserror`**: ライブラリや下位レイヤーで使用。エラーの種類が明確で、呼び出し元に具体的なエラー情報を提供したい場合。
- **`anyhow`**: アプリケーションの上位レイヤーや、エラーの種類を区別する必要がない場合。様々なエラーを単一の型で扱いたい場合。

一般的なパターンとして、ライブラリでは`thiserror`を使ってカスタムエラー型を定義し、アプリケーションでは`anyhow`を使ってそれらのエラーを集約します。

### パニックの適切な扱い方

Rustでは、回復不可能なエラーに対して`panic!`マクロを使用します。パニックが発生すると、デフォルトではスタックの巻き戻しが行われ、プログラムは終了します。

パニックは以下の場合に適しています：

1. 回復不可能なエラー（メモリ不足など）
2. プログラムの不変条件の違反（アサーション）
3. 開発中のデバッグ

一方、以下の場合はパニックを避け、`Result`を使うべきです：

1. 予期可能なエラー（ファイルが存在しない、ネットワーク接続の失敗など）
2. ユーザー入力に関連するエラー
3. 呼び出し元がエラーから回復する可能性がある場合

ライブラリを設計する際は、特に注意が必要です：

```rust
// 良くない例：ライブラリ関数でのパニック
pub fn parse_config(config_str: &str) -> Config {
    serde_json::from_str(config_str).expect("Invalid JSON")
}

// 良い例：エラーを返す
pub fn parse_config(config_str: &str) -> Result<Config, serde_json::Error> {
    serde_json::from_str(config_str)
}
```

パニックをキャッチする必要がある場合は、`std::panic::catch_unwind`を使用できます：

```rust
use std::panic::{self, AssertUnwindSafe};

fn run_operation() -> Result<(), String> {
    let result = panic::catch_unwind(AssertUnwindSafe(|| {
        // パニックする可能性のある操作
        if rand::random::<bool>() {
            panic!("Something went wrong!");
        }
    }));
    
    match result {
        Ok(()) => Ok(()),
        Err(e) => {
            if let Some(s) = e.downcast_ref::<&str>() {
                Err(s.to_string())
            } else if let Some(s) = e.downcast_ref::<String>() {
                Err(s.clone())
            } else {
                Err("Unknown panic".to_string())
            }
        }
    }
}
```

ただし、`catch_unwind`はFFIやスレッド間の境界でのみ使用するべきで、通常のエラー処理には`Result`を使用するべきです。

### エラー処理のベストプラクティス

1. **エラーの種類に応じた適切な処理**:
   - 回復可能なエラー → `Result`
   - 回復不可能なエラー → `panic!`

2. **エラーの伝播**:
   - 下位レイヤーでは具体的なエラー型（`thiserror`）
   - 上位レイヤーでは汎用的なエラー型（`anyhow`）

3. **エラーメッセージの充実**:
   - エラーの原因を明確に
   - コンテキスト情報を追加（`context`）

4. **エラーのログ記録**:
   - エラーを適切なレベルでログに記録
   - スタックトレースを含める

5. **ユーザーへのフィードバック**:
   - 技術的な詳細は隠し、ユーザーフレンドリーなメッセージを表示
   - 可能な解決策を提案

## 具体的なコード例

### 例1: Webアプリケーションのエラー処理

実際のWebアプリケーションでのエラー処理の例を見てみましょう。ここでは、ユーザー情報を取得するAPIを実装します。

まず、必要なクレートを追加します：

```toml
[dependencies]
thiserror = "1.0"
anyhow = "1.0"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
reqwest = { version = "0.11", features = ["json"] }
tokio = { version = "1", features = ["full"] }
```

次に、エラー型を定義します：

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ApiError {
    #[error("Network error: {0}")]
    Network(#[from] reqwest::Error),
    
    #[error("JSON parsing error: {0}")]
    Json(#[from] serde_json::Error),
    
    #[error("User not found with ID: {0}")]
    UserNotFound(u64),
    
    #[error("Rate limit exceeded")]
    RateLimited,
    
    #[error("Unauthorized access")]
    Unauthorized,
    
    #[error("Internal server error: {0}")]
    Internal(String),
}
```

ユーザー情報を取得する関数を実装します：

```rust
use anyhow::{Context, Result};
use serde::Deserialize;

#[derive(Deserialize)]
pub struct User {
    id: u64,
    name: String,
    email: String,
}

async fn fetch_user(client: &reqwest::Client, user_id: u64) -> Result<User, ApiError> {
    let url = format!("https://api.example.com/users/{}", user_id);
    
    let response = client.get(&url)
        .send()
        .await?; // reqwest::Error → ApiError::Network
    
    if response.status() == reqwest::StatusCode::NOT_FOUND {
        return Err(ApiError::UserNotFound(user_id));
    } else if response.status() == reqwest::StatusCode::UNAUTHORIZED {
        return Err(ApiError::Unauthorized);
    } else if response.status() == reqwest::StatusCode::TOO_MANY_REQUESTS {
        return Err(ApiError::RateLimited);
    } else if !response.status().is_success() {
        return Err(ApiError::Internal(format!(
            "Unexpected status code: {}", response.status()
        )));
    }
    
    let user = response.json::<User>().await?; // reqwest::Error → ApiError::Network
    Ok(user)
}
```

アプリケーションの上位レイヤーでは、`anyhow`を使ってエラーを集約します：

```rust
async fn get_user_info(client: &reqwest::Client, user_id: u64) -> Result<String> {
    let user = fetch_user(client, user_id)
        .await
        .context(format!("Failed to fetch user with ID: {}", user_id))?;
    
    let info = format!("User: {} ({})", user.name, user.email);
    Ok(info)
}

#[tokio::main]
async fn main() -> Result<()> {
    let client = reqwest::Client::new();
    
    match get_user_info(&client, 42).await {
        Ok(info) => println!("{}", info),
        Err(e) => {
            eprintln!("Error: {}", e);
            
            // エラーの種類に応じた処理
            if let Some(api_err) = e.downcast_ref::<ApiError>() {
                match api_err {
                    ApiError::UserNotFound(_) => {
                        println!("User not found. Please check the ID and try again.");
                    }
                    ApiError::Unauthorized => {
                        println!("Please log in to access this information.");
                    }
                    ApiError::RateLimited => {
                        println!("Too many requests. Please try again later.");
                    }
                    _ => {
                        println!("An unexpected error occurred. Please try again later.");
                    }
                }
            }
        }
    }
    
    Ok(())
}
```

### 例2: カスタムエラー型の設計と実装

より複雑なアプリケーションでは、ドメイン固有のエラーを定義することが重要です。以下は、ユーザー認証システムのエラー型の例です：

```rust
use thiserror::Error;
use std::time::Duration;

#[derive(Error, Debug)]
pub enum AuthError {
    #[error("Invalid credentials")]
    InvalidCredentials,
    
    #[error("Account locked: too many failed attempts")]
    AccountLocked,
    
    #[error("Account not verified")]
    AccountNotVerified,
    
    #[error("Password expired: please reset your password")]
    PasswordExpired,
    
    #[error("Token expired")]
    TokenExpired,
    
    #[error("Invalid token")]
    InvalidToken,
    
    #[error("Rate limited: please try again after {0:?}")]
    RateLimited(Duration),
    
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
    
    #[error("Internal error: {0}")]
    Internal(String),
}

// 認証結果を表す型
pub enum AuthResult<T> {
    Success(T),
    Failure(AuthError),
    NeedsSecondFactor(String), // 二要素認証が必要な場合のトークン
}

impl<T> AuthResult<T> {
    // 結果を取得するか、エラーを返す
    pub fn unwrap(self) -> Result<T, AuthError> {
        match self {
            Self::Success(value) => Ok(value),
            Self::Failure(err) => Err(err),
            Self::NeedsSecondFactor(_) => Err(AuthError::Internal(
                "Second factor required but not provided".to_string()
            )),
        }
    }
    
    // 二要素認証が必要かどうかをチェック
    pub fn needs_second_factor(&self) -> bool {
        matches!(self, Self::NeedsSecondFactor(_))
    }
    
    // 二要素認証トークンを取得
    pub fn second_factor_token(&self) -> Option<&str> {
        match self {
            Self::NeedsSecondFactor(token) => Some(token),
            _ => None,
        }
    }
}

// 認証サービス
pub struct AuthService {
    // データベース接続などのフィールド
}

impl AuthService {
    pub async fn login(&self, username: &str, password: &str) -> AuthResult<String> {
        // ユーザーの検証ロジック（実際の実装では、データベースからユーザー情報を取得）
        if username == "admin" && password == "password" {
            // 成功: JWTトークンを返す
            return AuthResult::Success("jwt-token-here".to_string());
        } else if username == "locked_user" {
            // アカウントロック
            return AuthResult::Failure(AuthError::AccountLocked);
        } else if username == "unverified_user" {
            // 未検証アカウント
            return AuthResult::Failure(AuthError::AccountNotVerified);
        } else if username == "2fa_user" {
            // 二要素認証が必要
            return AuthResult::NeedsSecondFactor("2fa-token-here".to_string());
        } else {
            // 無効な認証情報
            return AuthResult::Failure(AuthError::InvalidCredentials);
        }
    }
    
    pub async fn verify_second_factor(&self, token: &str, code: &str) -> Result<String, AuthError> {
        // 二要素認証コードの検証ロジック
        if token == "2fa-token-here" && code == "123456" {
            Ok("jwt-token-here".to_string())
        } else {
            Err(AuthError::InvalidToken)
        }
    }
}

// 使用例
async fn authenticate_user(
    auth_service: &AuthService,
    username: &str,
    password: &str,
    second_factor_code: Option<&str>,
) -> Result<String, AuthError> {
    let result = auth_service.login(username, password).await;
    
    if result.needs_second_factor() {
        let token = result.second_factor_token()
            .expect("Second factor token should be present");
        
        let code = second_factor_code.ok_or_else(|| {
            AuthError::Internal("Second factor code required".to_string())
        })?;
        
        auth_service.verify_second_factor(token, code).await
    } else {
        result.unwrap()
    }
}
```

## PHP/Laravel開発者向けのポイント

### PHPの例外処理とRustのエラー処理の比較

PHPでは、エラー処理に例外（Exception）を使用します：

```php
try {
    $file = fopen('non_existent_file.txt', 'r');
    if ($file === false) {
        throw new Exception('Failed to open file');
    }
    // ファイル操作
} catch (Exception $e) {
    echo 'Caught exception: ',  $e->getMessage(), "\n";
} finally {
    if (isset($file) && $file !== false) {
        fclose($file);
    }
}
```

Laravelでは、カスタム例外クラスを定義することが一般的です：

```php
class UserNotFoundException extends Exception
{
    protected $userId;
    
    public function __construct($userId)
    {
        $this->userId = $userId;
        parent::__construct("User not found with ID: {$userId}");
    }
    
    public function getUserId()
    {
        return $this->userId;
    }
}
```

Rustでは、例外の代わりに`Result`型を使用します：

```rust
fn open_file(path: &str) -> Result<std::fs::File, std::io::Error> {
    std::fs::File::open(path)
}

fn main() {
    match open_file("non_existent_file.txt") {
        Ok(file) => {
            // ファイル操作
        }
        Err(e) => {
            println!("Failed to open file: {}", e);
        }
    }
}
```

### 主な違い

1. **明示性**:
   - PHP: 例外は暗黙的に伝播し、どの関数が例外を投げるかは型シグネチャから分からない
   - Rust: エラーは明示的に返され、関数の戻り値型（`Result<T, E>`）からエラーの可能性が分かる

2. **エラーの種類**:
   - PHP: 例外は階層構造（継承）で表現される
   - Rust: エラーは列挙型（enum）で表現される

3. **リソース管理**:
   - PHP: `finally`ブロックでリソースを解放
   - Rust: RAII（Resource Acquisition Is Initialization）パターンにより、スコープを抜けると自動的にリソースが解放される

4. **パフォーマンス**:
   - PHP: 例外はパフォーマンスコストが高い（特に深いコールスタックの場合）
   - Rust: `Result`型はゼロコスト抽象化であり、パフォーマンスへの影響は最小限

### Laravelのバリデーションエラーとの対比

Laravelでは、フォームバリデーションのエラーは`ValidationException`として扱われます：

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($request->all(), [
    'title' => 'required|max:255',
    'body' => 'required',
]);

if ($validator->fails()) {
    throw new \Illuminate\Validation\ValidationException($validator);
}
```

Rustでは、バリデーションエラーもカスタムエラー型として定義できます：

```rust
use thiserror::Error;
use std::collections::HashMap;

#[derive(Error, Debug)]
#[error("Validation failed: {errors:?}")]
struct ValidationError {
    errors: HashMap<String, Vec<String>>,
}

fn validate_user(user: &User) -> Result<(), ValidationError> {
    let mut errors = HashMap::new();
    
    if user.name.is_empty() {
        errors.entry("name".to_string())
            .or_insert_with(Vec::new)
            .push("Name is required".to_string());
    }
    
    if user.email.is_empty() {
        errors.entry("email".to_string())
            .or_insert_with(Vec::new)
            .push("Email is required".to_string());
    } else if !user.email.contains('@') {
        errors.entry("email".to_string())
            .or_insert_with(Vec::new)
            .push("Email must be valid".to_string());
    }
    
    if errors.is_empty() {
        Ok(())
    } else {
        Err(ValidationError { errors })
    }
}
```

### エラー処理の移行ポイント

1. **例外からResultへの考え方の転換**:
   - PHPでは「例外的な状況」として扱うものも、Rustでは通常の戻り値として扱う
   - エラーを値として扱い、明示的に処理する習慣をつける

2. **エラー型の設計**:
   - PHPの例外階層をRustの列挙型に変換する
   - 共通のエラートレイトを実装して、エラー処理を統一する

3. **コンテキスト情報の追加**:
   - PHPでは例外にコンテキスト情報を追加するカスタムメソッドを定義
   - Rustでは`anyhow::Context`や独自のメソッドを使用

4. **エラーの伝播**:
   - PHPでは例外は自動的に伝播する
   - Rustでは`?`演算子を使って明示的に伝播させる

5. **リソース管理**:
   - PHPの`try/finally`パターンからRustのRAIIパターンへの移行
   - `Drop`トレイトを実装して、リソースの自動解放を確保

## 演習問題

### 演習1: カスタムエラー型の設計と実装

**目標**: オンラインショッピングアプリケーションのカスタムエラー型を設計・実装する

**要件**:
1. 以下のエラー種類を含むカスタムエラー型を定義する:
   - 商品が見つからない（ProductNotFound）
   - 在庫不足（InsufficientStock）
   - 無効な支払い方法（InvalidPaymentMethod）
   - 支払い処理エラー（PaymentProcessingError）
   - データベースエラー（DatabaseError）
   - ネットワークエラー（NetworkError）

2. 各エラーに適切なフィールドとエラーメッセージを定義する
3. 必要に応じて、他のエラー型からの変換を実装する
4. エラーを処理するユーティリティ関数を実装する

**ヒント**:
- `thiserror`クレートを使用する
- エラーメッセージには、問題の詳細と可能な解決策を含める
- エラー型は、アプリケーションの要件に合わせて設計する

**スケルトンコード**:

```rust
use thiserror::Error;

// TODO: カスタムエラー型を定義する

// 商品情報
struct Product {
    id: u64,
    name: String,
    price: f64,
    stock: u32,
}

// 支払い方法
enum PaymentMethod {
    CreditCard(String),
    PayPal(String),
    BankTransfer,
}

// 注文処理関数
fn process_order(
    product_id: u64,
    quantity: u32,
    payment_method: PaymentMethod,
) -> Result<String, ShopError> {
    // TODO: 商品の検索、在庫チェック、支払い処理を実装する
    // エラーが発生した場合は、適切なエラーを返す
    
    Ok("Order processed successfully".to_string())
}

// メイン関数
fn main() {
    // TODO: process_order関数を呼び出し、結果を処理する
}
```

### 演習2: エラー処理パターンの実装

**目標**: 複数のエラーソースを持つアプリケーションでのエラー処理パターンを実装する

**要件**:
1. 以下の機能を持つコマンドラインツールを実装する:
   - 設定ファイル（JSON）の読み込み
   - APIからのデータ取得
   - 取得したデータの処理と保存

2. 各ステップで発生する可能性のあるエラーを適切に処理する
3. エラーに文脈情報を追加する
4. ユーザーフレンドリーなエラーメッセージを表示する

**ヒント**:
- `thiserror`と`anyhow`の両方を使用する
- 下位レイヤーでは具体的なエラー型を定義し、上位レイヤーでは`anyhow::Result`を使用する
- `context`メソッドを使って、エラーに文脈情報を追加する

**スケルトンコード**:

```rust
use std::fs;
use std::path::Path;
use anyhow::{Context, Result};
use serde::Deserialize;
use thiserror::Error;

// 設定ファイルの構造
#[derive(Deserialize)]
struct Config {
    api_url: String,
    api_key: String,
    output_file: String,
}

// TODO: カスタムエラー型を定義する

// 設定ファイルを読み込む関数
fn load_config(path: &Path) -> Result<Config> {
    // TODO: ファイルを読み込み、JSONをパースする
    // エラーが発生した場合は、文脈情報を追加する
    
    Ok(Config {
        api_url: "https://api.example.com".to_string(),
        api_key: "dummy-api-key".to_string(),
        output_file: "output.json".to_string(),
    })
}

// APIからデータを取得する関数
async fn fetch_data(config: &Config) -> Result<Vec<String>> {
    // TODO: APIからデータを取得する
    // エラーが発生した場合は、適切なエラーを返す
    
    Ok(vec!["data1".to_string(), "data2".to_string()])
}

// データを処理して保存する関数
fn process_and_save_data(data: Vec<String>, output_file: &str) -> Result<()> {
    // TODO: データを処理して保存する
    // エラーが発生した場合は、文脈情報を追加する
    
    Ok(())
}

// メイン関数
#[tokio::main]
async fn main() -> Result<()> {
    // TODO: 各関数を呼び出し、エラーを適切に処理する
    
    Ok(())
}
```

### 演習3: パニックの適切な扱い方

**目標**: パニックの適切な扱い方を理解し、実装する

**要件**:
1. 以下の機能を持つライブラリを実装する:
   - 安全なAPIを提供する公開関数
   - 内部的にパニックする可能性のある関数
   - パニックをキャッチして適切に処理する関数

2. パニックが発生する可能性のある操作を特定し、適切に処理する
3. ライブラリの利用者に安全なAPIを提供する

**ヒント**:
- `std::panic::catch_unwind`を使用する
- パニックをキャッチする場合は、適切なエラーに変換する
- 公開APIでは、パニックを発生させないようにする

**スケルトンコード**:

```rust
use std::panic::{self, AssertUnwindSafe};
use thiserror::Error;

// カスタムエラー型
#[derive(Error, Debug)]
enum LibraryError {
    #[error("Invalid input: {0}")]
    InvalidInput(String),
    
    #[error("Internal error: {0}")]
    Internal(String),
}

// 内部的にパニックする可能性のある関数
fn internal_operation(value: i32) -> i32 {
    // TODO: 特定の条件でパニックする関数を実装する
    
    value * 2
}

// パニックをキャッチして適切に処理する関数
fn safe_operation(value: i32) -> Result<i32, LibraryError> {
    // TODO: internal_operation関数を呼び出し、パニックをキャッチする
    // パニックが発生した場合は、適切なエラーに変換する
    
    Ok(value * 2)
}

// 公開API
pub fn process_value(value: i32) -> Result<i32, LibraryError> {
    // TODO: 入力値を検証し、safe_operation関数を呼び出す
    // エラーが発生した場合は、適切に処理する
    
    Ok(value * 2)
}

// テスト関数
fn main() {
    // TODO: process_value関数を様々な入力値で呼び出し、結果を確認する
}
```

これらの演習問題を通じて、Rustのエラー処理パターンを実践的に学ぶことができます。各演習は、実際のアプリケーション開発で遭遇する可能性のある状況を模しています。

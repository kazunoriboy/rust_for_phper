# トピック7: 実践的アプリケーション開発ワークフロー

## 学習目標

このセクションを学ぶことで、以下のことができるようになります：

- Rustの実践的なアプリケーション開発ワークフローを理解し、適用する
- 効率的なプロジェクト構造とモジュール設計を実装する
- テスト駆動開発（TDD）をRustで実践する
- エラー処理、ロギング、設定管理の実践的パターンを適用する
- CLIアプリケーションとWebサービスの開発手法を習得する

## 主要な概念の説明

### 効率的なプロジェクト構造とモジュール設計

Rustでは、適切なプロジェクト構造とモジュール設計が、コードの保守性と拡張性に大きく影響します。

#### プロジェクト構造のベストプラクティス

Rustプロジェクトの標準的な構造は以下のようになります：

```
my_project/
├── Cargo.toml          # プロジェクト設定ファイル
├── Cargo.lock          # 依存関係のロックファイル
├── src/                # ソースコードディレクトリ
│   ├── main.rs         # バイナリのエントリーポイント
│   ├── lib.rs          # ライブラリのルートモジュール
│   ├── bin/            # 複数のバイナリを含むディレクトリ
│   │   └── tool.rs     # 追加のバイナリ
│   └── module_name/    # サブモジュールディレクトリ
│       ├── mod.rs      # モジュールのエントリーポイント
│       └── submodule.rs # サブモジュール
├── tests/              # 統合テスト
│   └── integration_test.rs
├── benches/            # ベンチマーク
│   └── benchmark.rs
├── examples/           # 使用例
│   └── example.rs
└── docs/               # ドキュメント
    └── README.md
```

この構造には以下の利点があります：

1. **明確な責任分離**: 各ファイルとディレクトリが特定の責任を持つ
2. **スケーラビリティ**: プロジェクトの成長に合わせて拡張可能
3. **可読性**: 新しい開発者が素早くプロジェクト構造を理解できる

#### モジュールシステムの効果的な使用

Rustのモジュールシステムを使用して、コードを論理的な単位に分割できます：

```rust
// lib.rs
pub mod config;
pub mod models;
pub mod services;
pub mod utils;

// models/mod.rs
pub mod user;
pub mod product;

// models/user.rs
pub struct User {
    pub id: u64,
    pub name: String,
    pub email: String,
}

impl User {
    pub fn new(id: u64, name: String, email: String) -> Self {
        User { id, name, email }
    }
}
```

モジュールの可視性を制御するために、`pub`キーワードを使用します：

```rust
// 公開モジュール
pub mod public_module {
    // 公開関数
    pub fn public_function() {
        println!("This is public");
        private_function();
    }
    
    // 非公開関数（同じモジュール内からのみアクセス可能）
    fn private_function() {
        println!("This is private");
    }
    
    // 公開構造体
    pub struct PublicStruct {
        pub public_field: i32,
        private_field: i32,
    }
    
    impl PublicStruct {
        pub fn new(public: i32, private: i32) -> Self {
            PublicStruct {
                public_field: public,
                private_field: private,
            }
        }
        
        pub fn get_private_field(&self) -> i32 {
            self.private_field
        }
    }
}
```

#### 再エクスポートパターン

モジュール階層を簡素化するために、再エクスポートパターンを使用できます：

```rust
// lib.rs
mod config;
mod models;
mod services;
mod utils;

// 公開APIを再エクスポート
pub use config::Config;
pub use models::{User, Product};
pub use services::{UserService, ProductService};
```

これにより、ユーザーは以下のようにシンプルにインポートできます：

```rust
use my_library::{Config, User, Product, UserService};
```

### テスト駆動開発（TDD）

Rustは、テスト駆動開発（TDD）をサポートする優れたテストフレームワークを提供しています。

#### ユニットテスト

ユニットテストは、通常、テスト対象のコードと同じファイル内に記述します：

```rust
// src/utils.rs
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
        assert_eq!(add(-1, 1), 0);
        assert_eq!(add(0, 0), 0);
    }
}
```

`#[cfg(test)]`属性は、テストコードが通常のビルドには含まれず、テスト実行時のみコンパイルされることを保証します。

#### 統合テスト

統合テストは、`tests`ディレクトリに配置します：

```rust
// tests/integration_test.rs
use my_library::{Config, User, UserService};

#[test]
fn test_user_service_with_config() {
    let config = Config::new("test_config.json");
    let user = User::new(1, "Test User".to_string(), "test@example.com".to_string());
    let service = UserService::new(config);
    
    let result = service.save_user(user);
    assert!(result.is_ok());
}
```

#### テストフィクスチャとセットアップ

共通のセットアップコードを再利用するために、テストフィクスチャを作成できます：

```rust
// tests/common/mod.rs
use my_library::{Config, User};
use std::sync::Once;

static INIT: Once = Once::new();

pub fn setup() -> Config {
    // 初期化コードを一度だけ実行
    INIT.call_once(|| {
        // テスト環境のセットアップ
        std::fs::write("test_config.json", r#"{"env": "test"}"#).unwrap();
    });
    
    Config::new("test_config.json")
}

pub fn create_test_user(id: u64) -> User {
    User::new(id, format!("User {}", id), format!("user{}@example.com", id))
}

// tests/user_service_test.rs
mod common;

use my_library::UserService;
use common::{setup, create_test_user};

#[test]
fn test_user_service() {
    let config = setup();
    let user = create_test_user(1);
    let service = UserService::new(config);
    
    let result = service.save_user(user);
    assert!(result.is_ok());
}
```

#### モック

テスト時に外部依存関係をモックするために、`mockall`クレートなどを使用できます：

```rust
use mockall::predicate::*;
use mockall::*;

#[automock]
trait Database {
    fn save_user(&self, user: User) -> Result<(), Error>;
    fn get_user(&self, id: u64) -> Result<User, Error>;
}

struct UserService {
    db: Box<dyn Database>,
}

impl UserService {
    fn new(db: Box<dyn Database>) -> Self {
        UserService { db }
    }
    
    fn save_user(&self, user: User) -> Result<(), Error> {
        self.db.save_user(user)
    }
    
    fn get_user(&self, id: u64) -> Result<User, Error> {
        self.db.get_user(id)
    }
}

#[test]
fn test_user_service_with_mock() {
    let mut mock_db = MockDatabase::new();
    
    let test_user = User::new(1, "Test User".to_string(), "test@example.com".to_string());
    
    mock_db.expect_save_user()
        .with(eq(test_user.clone()))
        .times(1)
        .returning(|_| Ok(()));
    
    let service = UserService::new(Box::new(mock_db));
    let result = service.save_user(test_user);
    
    assert!(result.is_ok());
}
```

#### プロパティベーステスト

`proptest`クレートを使用して、プロパティベーステストを実装できます：

```rust
use proptest::prelude::*;

fn sort_numbers(mut numbers: Vec<i32>) -> Vec<i32> {
    numbers.sort();
    numbers
}

proptest! {
    #[test]
    fn test_sort_numbers(numbers in prop::collection::vec(-100..100, 0..100)) {
        let sorted = sort_numbers(numbers.clone());
        
        // ソート後の配列は元の配列と同じ長さ
        prop_assert_eq!(sorted.len(), numbers.len());
        
        // ソート後の配列は昇順
        for i in 1..sorted.len() {
            prop_assert!(sorted[i-1] <= sorted[i]);
        }
        
        // ソート後の配列は元の配列の要素をすべて含む
        let mut numbers_copy = numbers.clone();
        numbers_copy.sort();
        prop_assert_eq!(sorted, numbers_copy);
    }
}
```

### エラー処理、ロギング、設定管理

#### 実践的なエラー処理パターン

Rustでは、`thiserror`や`anyhow`クレートを使用して、エラー処理を簡素化できます：

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum AppError {
    #[error("設定ファイルが見つかりません: {0}")]
    ConfigNotFound(String),
    
    #[error("データベース接続エラー: {0}")]
    DatabaseConnection(#[from] sqlx::Error),
    
    #[error("ユーザーが見つかりません: ID = {0}")]
    UserNotFound(u64),
    
    #[error("不正なリクエスト: {0}")]
    InvalidRequest(String),
    
    #[error("内部サーバーエラー: {0}")]
    InternalServer(String),
}

// サービス層でのエラー処理
fn get_user(id: u64) -> Result<User, AppError> {
    let db_pool = get_db_pool().map_err(|e| AppError::InternalServer(e.to_string()))?;
    
    let user = db_pool.get_user(id)
        .map_err(|e| match e {
            sqlx::Error::RowNotFound => AppError::UserNotFound(id),
            other => AppError::DatabaseConnection(other),
        })?;
    
    Ok(user)
}

// APIレイヤーでのエラー処理
fn handle_get_user(id: u64) -> Result<impl warp::Reply, warp::Rejection> {
    match get_user(id) {
        Ok(user) => Ok(warp::reply::json(&user)),
        Err(e) => match e {
            AppError::UserNotFound(_) => Err(warp::reject::not_found()),
            AppError::InvalidRequest(msg) => Err(warp::reject::custom(BadRequest(msg))),
            _ => Err(warp::reject::server_error()),
        },
    }
}
```

#### 構造化ロギング

`tracing`クレートを使用して、構造化ロギングを実装できます：

```rust
use tracing::{info, error, warn, debug, instrument};
use tracing_subscriber::{fmt, EnvFilter};

fn main() {
    // ロガーの初期化
    tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::from_default_env())
        .init();
    
    info!("アプリケーションを開始しています");
    
    let result = process_data(42);
    
    if let Err(e) = result {
        error!(error = %e, "データ処理中にエラーが発生しました");
    }
}

#[instrument(skip(input), fields(input_value = %input))]
fn process_data(input: i32) -> Result<i32, String> {
    debug!("データ処理を開始します");
    
    if input < 0 {
        warn!("負の入力値: {}", input);
        return Err("負の値は処理できません".to_string());
    }
    
    let result = input * 2;
    info!("処理が完了しました: 結果 = {}", result);
    
    Ok(result)
}
```

`#[instrument]`属性は、関数の入出力を自動的にトレースします。

#### 設定管理

`config`クレートを使用して、複数のソースから設定を読み込むことができます：

```rust
use config::{Config, ConfigError, File, Environment};
use serde::Deserialize;
use std::env;

#[derive(Debug, Deserialize)]
struct DatabaseConfig {
    url: String,
    max_connections: u32,
}

#[derive(Debug, Deserialize)]
struct ServerConfig {
    host: String,
    port: u16,
}

#[derive(Debug, Deserialize)]
struct AppConfig {
    debug: bool,
    database: DatabaseConfig,
    server: ServerConfig,
}

fn load_config() -> Result<AppConfig, ConfigError> {
    let env = env::var("RUN_ENV").unwrap_or_else(|_| "development".to_string());
    
    let config = Config::builder()
        // デフォルト設定
        .add_source(File::with_name("config/default"))
        // 環境固有の設定
        .add_source(File::with_name(&format!("config/{}", env)).required(false))
        // 環境変数（APP_DATABASE__URL など）
        .add_source(Environment::with_prefix("APP").separator("__"))
        .build()?;
    
    config.try_deserialize()
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = load_config()?;
    
    println!("設定を読み込みました:");
    println!("データベースURL: {}", config.database.url);
    println!("サーバーポート: {}", config.server.port);
    
    Ok(())
}
```

### CLIアプリケーション開発

Rustは、コマンドラインインターフェース（CLI）アプリケーションの開発に適しています。

#### `clap`を使用したコマンドライン引数の処理

```rust
use clap::{App, Arg, SubCommand};

fn main() {
    let matches = App::new("My CLI App")
        .version("1.0")
        .author("Your Name <your.email@example.com>")
        .about("A sample CLI application")
        .arg(
            Arg::with_name("config")
                .short("c")
                .long("config")
                .value_name("FILE")
                .help("設定ファイルへのパス")
                .takes_value(true),
        )
        .arg(
            Arg::with_name("verbose")
                .short("v")
                .multiple(true)
                .help("詳細なログ出力"),
        )
        .subcommand(
            SubCommand::with_name("add")
                .about("アイテムを追加")
                .arg(
                    Arg::with_name("name")
                        .help("アイテム名")
                        .required(true)
                        .index(1),
                ),
        )
        .subcommand(
            SubCommand::with_name("list")
                .about("アイテム一覧を表示"),
        )
        .get_matches();
    
    // 設定ファイルのパスを取得
    let config_path = matches.value_of("config").unwrap_or("config.toml");
    println!("設定ファイル: {}", config_path);
    
    // 詳細度レベルを取得
    let verbosity = matches.occurrences_of("verbose");
    println!("詳細度レベル: {}", verbosity);
    
    // サブコマンドを処理
    match matches.subcommand() {
        ("add", Some(add_matches)) => {
            let name = add_matches.value_of("name").unwrap();
            println!("アイテム '{}' を追加します", name);
        },
        ("list", Some(_)) => {
            println!("アイテム一覧を表示します");
        },
        _ => println!("サブコマンドが指定されていません"),
    }
}
```

#### インタラクティブCLIの作成

`dialoguer`クレートを使用して、インタラクティブなCLIを作成できます：

```rust
use dialoguer::{Input, Select, Confirm, Password, MultiSelect};
use console::style;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("{}", style("インタラクティブCLIデモ").bold().cyan());
    
    // テキスト入力
    let name: String = Input::new()
        .with_prompt("あなたの名前は？")
        .default("Guest".into())
        .interact_text()?;
    
    // パスワード入力
    let password = Password::new()
        .with_prompt("パスワードを入力してください")
        .with_confirmation("パスワードを確認してください", "パスワードが一致しません")
        .interact()?;
    
    // 選択肢
    let options = vec!["Option A", "Option B", "Option C"];
    let selection = Select::new()
        .with_prompt("オプションを選択してください")
        .default(0)
        .items(&options)
        .interact()?;
    
    // 複数選択
    let items = vec!["Item 1", "Item 2", "Item 3", "Item 4"];
    let chosen = MultiSelect::new()
        .with_prompt("アイテムを選択してください")
        .items(&items)
        .interact()?;
    
    // 確認
    let confirmed = Confirm::new()
        .with_prompt("続行しますか？")
        .default(true)
        .interact()?;
    
    println!("\n結果:");
    println!("名前: {}", name);
    println!("パスワード: {}", "*".repeat(password.len()));
    println!("選択したオプション: {}", options[selection]);
    println!("選択したアイテム: {:?}", chosen.iter().map(|&i| items[i]).collect::<Vec<_>>());
    println!("確認: {}", if confirmed { "はい" } else { "いいえ" });
    
    Ok(())
}
```

#### プログレスバーとスピナー

`indicatif`クレートを使用して、プログレスバーとスピナーを表示できます：

```rust
use indicatif::{ProgressBar, ProgressStyle, MultiProgress};
use std::{thread, time::Duration};

fn main() {
    let multi = MultiProgress::new();
    
    // スピナー
    let spinner = multi.add(ProgressBar::new_spinner());
    spinner.set_style(
        ProgressStyle::default_spinner()
            .tick_chars("⠁⠂⠄⡀⢀⠠⠐⠈")
            .template("{spinner} {msg}")
            .unwrap(),
    );
    
    // プログレスバー
    let pb = multi.add(ProgressBar::new(100));
    pb.set_style(
        ProgressStyle::default_bar()
            .template("{bar:40.cyan/blue} {pos}/{len} ({eta})")
            .unwrap()
            .progress_chars("##-"),
    );
    
    // スピナーとプログレスバーを別スレッドで更新
    let spinner_handle = thread::spawn(move || {
        for i in 0..100 {
            spinner.set_message(format!("処理中... {}/100", i + 1));
            thread::sleep(Duration::from_millis(50));
            spinner.tick();
        }
        spinner.finish_with_message("処理完了！");
    });
    
    let pb_handle = thread::spawn(move || {
        for i in 0..100 {
            pb.inc(1);
            thread::sleep(Duration::from_millis(100));
        }
        pb.finish_with_message("完了！");
    });
    
    spinner_handle.join().unwrap();
    pb_handle.join().unwrap();
}
```

### Webサービス開発

Rustには、Webサービスを開発するための様々なフレームワークがあります。

#### `actix-web`を使用したRESTful API

```rust
use actix_web::{web, App, HttpResponse, HttpServer, Responder};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: u64,
    name: String,
    email: String,
}

async fn get_users() -> impl Responder {
    let users = vec![
        User { id: 1, name: "Alice".to_string(), email: "alice@example.com".to_string() },
        User { id: 2, name: "Bob".to_string(), email: "bob@example.com".to_string() },
    ];
    
    HttpResponse::Ok().json(users)
}

async fn get_user(path: web::Path<(u64,)>) -> impl Responder {
    let user_id = path.0;
    
    // 実際のアプリケーションではデータベースから取得
    if user_id == 1 {
        let user = User {
            id: 1,
            name: "Alice".to_string(),
            email: "alice@example.com".to_string(),
        };
        HttpResponse::Ok().json(user)
    } else {
        HttpResponse::NotFound().body(format!("User with ID {} not found", user_id))
    }
}

#[derive(Debug, Deserialize)]
struct CreateUserRequest {
    name: String,
    email: String,
}

async fn create_user(user: web::Json<CreateUserRequest>) -> impl Responder {
    // 実際のアプリケーションではデータベースに保存
    let new_user = User {
        id: 3, // 通常は自動生成
        name: user.name.clone(),
        email: user.email.clone(),
    };
    
    HttpResponse::Created().json(new_user)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/users", web::get().to(get_users))
            .route("/users/{id}", web::get().to(get_user))
            .route("/users", web::post().to(create_user))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

#### データベース接続

`sqlx`クレートを使用して、データベースに接続できます：

```rust
use sqlx::{postgres::PgPoolOptions, Pool, Postgres};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize, sqlx::FromRow)]
struct User {
    id: i64,
    name: String,
    email: String,
    created_at: chrono::DateTime<chrono::Utc>,
}

struct UserRepository {
    pool: Pool<Postgres>,
}

impl UserRepository {
    fn new(pool: Pool<Postgres>) -> Self {
        UserRepository { pool }
    }
    
    async fn find_all(&self) -> Result<Vec<User>, sqlx::Error> {
        sqlx::query_as::<_, User>("SELECT * FROM users ORDER BY id")
            .fetch_all(&self.pool)
            .await
    }
    
    async fn find_by_id(&self, id: i64) -> Result<Option<User>, sqlx::Error> {
        sqlx::query_as::<_, User>("SELECT * FROM users WHERE id = $1")
            .bind(id)
            .fetch_optional(&self.pool)
            .await
    }
    
    async fn create(&self, name: &str, email: &str) -> Result<User, sqlx::Error> {
        sqlx::query_as::<_, User>(
            "INSERT INTO users (name, email, created_at) VALUES ($1, $2, $3) RETURNING *",
        )
        .bind(name)
        .bind(email)
        .bind(chrono::Utc::now())
        .fetch_one(&self.pool)
        .await
    }
    
    async fn update(&self, id: i64, name: &str, email: &str) -> Result<Option<User>, sqlx::Error> {
        sqlx::query_as::<_, User>(
            "UPDATE users SET name = $1, email = $2 WHERE id = $3 RETURNING *",
        )
        .bind(name)
        .bind(email)
        .bind(id)
        .fetch_optional(&self.pool)
        .await
    }
    
    async fn delete(&self, id: i64) -> Result<bool, sqlx::Error> {
        let result = sqlx::query("DELETE FROM users WHERE id = $1")
            .bind(id)
            .execute(&self.pool)
            .await?;
        
        Ok(result.rows_affected() > 0)
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // データベース接続プールを作成
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect("postgres://postgres:password@localhost/testdb")
        .await?;
    
    // マイグレーションを実行
    sqlx::migrate!("./migrations").run(&pool).await?;
    
    // リポジトリを作成
    let repo = UserRepository::new(pool);
    
    // ユーザーを作成
    let user = repo.create("Alice", "alice@example.com").await?;
    println!("作成されたユーザー: {:?}", user);
    
    // すべてのユーザーを取得
    let users = repo.find_all().await?;
    println!("すべてのユーザー: {:?}", users);
    
    // ユーザーを更新
    let updated_user = repo.update(user.id, "Alicia", "alicia@example.com").await?;
    println!("更新されたユーザー: {:?}", updated_user);
    
    // ユーザーを削除
    let deleted = repo.delete(user.id).await?;
    println!("ユーザーが削除されました: {}", deleted);
    
    Ok(())
}
```

#### 非同期処理とバックグラウンドタスク

`tokio`クレートを使用して、非同期処理とバックグラウンドタスクを実装できます：

```rust
use tokio::time::{sleep, Duration};
use tokio::task::JoinHandle;
use std::sync::Arc;
use tokio::sync::Mutex;

struct TaskManager {
    tasks: Arc<Mutex<Vec<JoinHandle<()>>>>,
}

impl TaskManager {
    fn new() -> Self {
        TaskManager {
            tasks: Arc::new(Mutex::new(Vec::new())),
        }
    }
    
    async fn add_task<F>(&self, task: F)
    where
        F: FnOnce() -> () + Send + 'static,
    {
        let handle = tokio::spawn(async move {
            task();
        });
        
        self.tasks.lock().await.push(handle);
    }
    
    async fn wait_all(&self) {
        let mut tasks = self.tasks.lock().await;
        
        while let Some(task) = tasks.pop() {
            if let Err(e) = task.await {
                eprintln!("タスクの実行中にエラーが発生しました: {:?}", e);
            }
        }
    }
}

#[tokio::main]
async fn main() {
    let manager = TaskManager::new();
    
    // タスクを追加
    for i in 0..5 {
        let i = i; // クロージャにムーブ
        manager.add_task(move || {
            println!("タスク {} を開始します", i);
            // 非同期コードをブロッキングコンテキストで実行
            let rt = tokio::runtime::Handle::current();
            rt.block_on(async {
                sleep(Duration::from_secs(i + 1)).await;
                println!("タスク {} が完了しました", i);
            });
        }).await;
    }
    
    println!("すべてのタスクが開始されました");
    
    // すべてのタスクが完了するのを待つ
    manager.wait_all().await;
    
    println!("すべてのタスクが完了しました");
}
```

## 具体的なコード例

### 例1: 構造化されたCLIアプリケーション

以下は、`clap`と`config`を使用した構造化されたCLIアプリケーションの例です：

```rust
use clap::{App, Arg, SubCommand};
use config::{Config, ConfigError, File, Environment};
use serde::Deserialize;
use std::path::Path;
use std::fs;
use std::error::Error;

#[derive(Debug, Deserialize)]
struct AppConfig {
    data_dir: String,
    verbose: bool,
}

fn load_config(config_path: &str) -> Result<AppConfig, ConfigError> {
    let config = Config::builder()
        .add_source(File::with_name(config_path).required(false))
        .add_source(Environment::with_prefix("APP"))
        .build()?;
    
    config.try_deserialize()
}

fn add_item(name: &str, config: &AppConfig) -> Result<(), Box<dyn Error>> {
    let data_dir = Path::new(&config.data_dir);
    
    if !data_dir.exists() {
        fs::create_dir_all(data_dir)?;
    }
    
    let file_path = data_dir.join(format!("{}.txt", name));
    fs::write(&file_path, format!("Item: {}\nCreated: {}", name, chrono::Local::now()))?;
    
    if config.verbose {
        println!("アイテム '{}' を作成しました: {:?}", name, file_path);
    } else {
        println!("アイテム '{}' を作成しました", name);
    }
    
    Ok(())
}

fn list_items(config: &AppConfig) -> Result<(), Box<dyn Error>> {
    let data_dir = Path::new(&config.data_dir);
    
    if !data_dir.exists() {
        println!("アイテムがありません");
        return Ok(());
    }
    
    let mut items = Vec::new();
    
    for entry in fs::read_dir(data_dir)? {
        let entry = entry?;
        let path = entry.path();
        
        if path.is_file() && path.extension().map_or(false, |ext| ext == "txt") {
            if let Some(name) = path.file_stem().and_then(|s| s.to_str()) {
                items.push(name.to_string());
            }
        }
    }
    
    if items.is_empty() {
        println!("アイテムがありません");
    } else {
        println!("アイテム一覧:");
        for (i, item) in items.iter().enumerate() {
            println!("{}. {}", i + 1, item);
        }
    }
    
    Ok(())
}

fn delete_item(name: &str, config: &AppConfig) -> Result<(), Box<dyn Error>> {
    let data_dir = Path::new(&config.data_dir);
    let file_path = data_dir.join(format!("{}.txt", name));
    
    if file_path.exists() {
        fs::remove_file(&file_path)?;
        println!("アイテム '{}' を削除しました", name);
    } else {
        println!("アイテム '{}' が見つかりません", name);
    }
    
    Ok(())
}

fn main() -> Result<(), Box<dyn Error>> {
    let matches = App::new("Item Manager")
        .version("1.0")
        .author("Your Name <your.email@example.com>")
        .about("アイテム管理CLIアプリケーション")
        .arg(
            Arg::with_name("config")
                .short("c")
                .long("config")
                .value_name("FILE")
                .help("設定ファイルへのパス")
                .takes_value(true)
                .default_value("config.toml"),
        )
        .arg(
            Arg::with_name("verbose")
                .short("v")
                .long("verbose")
                .help("詳細なログ出力"),
        )
        .subcommand(
            SubCommand::with_name("add")
                .about("アイテムを追加")
                .arg(
                    Arg::with_name("name")
                        .help("アイテム名")
                        .required(true)
                        .index(1),
                ),
        )
        .subcommand(
            SubCommand::with_name("list")
                .about("アイテム一覧を表示"),
        )
        .subcommand(
            SubCommand::with_name("delete")
                .about("アイテムを削除")
                .arg(
                    Arg::with_name("name")
                        .help("アイテム名")
                        .required(true)
                        .index(1),
                ),
        )
        .get_matches();
    
    // 設定ファイルのパスを取得
    let config_path = matches.value_of("config").unwrap();
    
    // 設定を読み込む
    let mut config = load_config(config_path)?;
    
    // コマンドラインオプションで設定を上書き
    if matches.is_present("verbose") {
        config.verbose = true;
    }
    
    // サブコマンドを処理
    match matches.subcommand() {
        ("add", Some(add_matches)) => {
            let name = add_matches.value_of("name").unwrap();
            add_item(name, &config)?;
        },
        ("list", Some(_)) => {
            list_items(&config)?;
        },
        ("delete", Some(delete_matches)) => {
            let name = delete_matches.value_of("name").unwrap();
            delete_item(name, &config)?;
        },
        _ => {
            println!("サブコマンドが指定されていません。ヘルプを表示するには '--help' を使用してください。");
        },
    }
    
    Ok(())
}
```

### 例2: レイヤードアーキテクチャを使用したWebアプリケーション

以下は、レイヤードアーキテクチャを使用したWebアプリケーションの例です：

```rust
// main.rs
use actix_web::{web, App, HttpServer};
use dotenv::dotenv;
use sqlx::postgres::PgPoolOptions;
use std::env;

mod config;
mod models;
mod repositories;
mod services;
mod handlers;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // 環境変数を読み込む
    dotenv().ok();
    
    // ロガーを初期化
    env_logger::init_from_env(env_logger::Env::default().default_filter_or("info"));
    
    // 設定を読み込む
    let config = config::load_config().expect("設定の読み込みに失敗しました");
    
    // データベース接続プールを作成
    let pool = PgPoolOptions::new()
        .max_connections(config.database.max_connections)
        .connect(&config.database.url)
        .await
        .expect("データベース接続に失敗しました");
    
    // リポジトリを作成
    let user_repository = repositories::UserRepository::new(pool.clone());
    
    // サービスを作成
    let user_service = services::UserService::new(user_repository);
    
    // サービスをアプリケーションの状態として共有
    let app_state = web::Data::new(services::AppState {
        user_service,
    });
    
    // HTTPサーバーを起動
    log::info!("サーバーを開始します: {}:{}", config.server.host, config.server.port);
    
    HttpServer::new(move || {
        App::new()
            .app_data(app_state.clone())
            .service(
                web::scope("/api")
                    .service(
                        web::scope("/users")
                            .route("", web::get().to(handlers::user::get_users))
                            .route("", web::post().to(handlers::user::create_user))
                            .route("/{id}", web::get().to(handlers::user::get_user))
                            .route("/{id}", web::put().to(handlers::user::update_user))
                            .route("/{id}", web::delete().to(handlers::user::delete_user))
                    )
            )
    })
    .bind(format!("{}:{}", config.server.host, config.server.port))?
    .run()
    .await
}

// config.rs
use serde::Deserialize;
use config::{Config, ConfigError, File, Environment};

#[derive(Debug, Deserialize)]
pub struct DatabaseConfig {
    pub url: String,
    pub max_connections: u32,
}

#[derive(Debug, Deserialize)]
pub struct ServerConfig {
    pub host: String,
    pub port: u16,
}

#[derive(Debug, Deserialize)]
pub struct AppConfig {
    pub database: DatabaseConfig,
    pub server: ServerConfig,
}

pub fn load_config() -> Result<AppConfig, ConfigError> {
    let env = std::env::var("RUN_ENV").unwrap_or_else(|_| "development".to_string());
    
    let config = Config::builder()
        .add_source(File::with_name("config/default"))
        .add_source(File::with_name(&format!("config/{}", env)).required(false))
        .add_source(Environment::with_prefix("APP").separator("__"))
        .build()?;
    
    config.try_deserialize()
}

// models.rs
use serde::{Deserialize, Serialize};
use chrono::{DateTime, Utc};

#[derive(Debug, Serialize, Deserialize, sqlx::FromRow)]
pub struct User {
    pub id: i64,
    pub name: String,
    pub email: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct CreateUserRequest {
    pub name: String,
    pub email: String,
}

#[derive(Debug, Deserialize)]
pub struct UpdateUserRequest {
    pub name: Option<String>,
    pub email: Option<String>,
}

// repositories.rs
use sqlx::{Pool, Postgres};
use crate::models::{User, CreateUserRequest, UpdateUserRequest};

pub struct UserRepository {
    pool: Pool<Postgres>,
}

impl UserRepository {
    pub fn new(pool: Pool<Postgres>) -> Self {
        UserRepository { pool }
    }
    
    pub async fn find_all(&self) -> Result<Vec<User>, sqlx::Error> {
        sqlx::query_as::<_, User>("SELECT * FROM users ORDER BY id")
            .fetch_all(&self.pool)
            .await
    }
    
    pub async fn find_by_id(&self, id: i64) -> Result<Option<User>, sqlx::Error> {
        sqlx::query_as::<_, User>("SELECT * FROM users WHERE id = $1")
            .bind(id)
            .fetch_optional(&self.pool)
            .await
    }
    
    pub async fn create(&self, user: &CreateUserRequest) -> Result<User, sqlx::Error> {
        let now = chrono::Utc::now();
        
        sqlx::query_as::<_, User>(
            "INSERT INTO users (name, email, created_at, updated_at) VALUES ($1, $2, $3, $4) RETURNING *",
        )
        .bind(&user.name)
        .bind(&user.email)
        .bind(now)
        .bind(now)
        .fetch_one(&self.pool)
        .await
    }
    
    pub async fn update(&self, id: i64, user: &UpdateUserRequest) -> Result<Option<User>, sqlx::Error> {
        let current_user = self.find_by_id(id).await?;
        
        if current_user.is_none() {
            return Ok(None);
        }
        
        let current_user = current_user.unwrap();
        let name = user.name.as_ref().unwrap_or(&current_user.name);
        let email = user.email.as_ref().unwrap_or(&current_user.email);
        let now = chrono::Utc::now();
        
        sqlx::query_as::<_, User>(
            "UPDATE users SET name = $1, email = $2, updated_at = $3 WHERE id = $4 RETURNING *",
        )
        .bind(name)
        .bind(email)
        .bind(now)
        .bind(id)
        .fetch_optional(&self.pool)
        .await
    }
    
    pub async fn delete(&self, id: i64) -> Result<bool, sqlx::Error> {
        let result = sqlx::query("DELETE FROM users WHERE id = $1")
            .bind(id)
            .execute(&self.pool)
            .await?;
        
        Ok(result.rows_affected() > 0)
    }
}

// services.rs
use crate::models::{User, CreateUserRequest, UpdateUserRequest};
use crate::repositories::UserRepository;
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ServiceError {
    #[error("データベースエラー: {0}")]
    DatabaseError(#[from] sqlx::Error),
    
    #[error("ユーザーが見つかりません: ID = {0}")]
    UserNotFound(i64),
    
    #[error("不正なリクエスト: {0}")]
    InvalidRequest(String),
}

pub struct UserService {
    repository: UserRepository,
}

impl UserService {
    pub fn new(repository: UserRepository) -> Self {
        UserService { repository }
    }
    
    pub async fn get_users(&self) -> Result<Vec<User>, ServiceError> {
        Ok(self.repository.find_all().await?)
    }
    
    pub async fn get_user(&self, id: i64) -> Result<User, ServiceError> {
        let user = self.repository.find_by_id(id).await?;
        
        user.ok_or(ServiceError::UserNotFound(id))
    }
    
    pub async fn create_user(&self, user: CreateUserRequest) -> Result<User, ServiceError> {
        // バリデーション
        if user.name.is_empty() {
            return Err(ServiceError::InvalidRequest("名前は必須です".to_string()));
        }
        
        if user.email.is_empty() {
            return Err(ServiceError::InvalidRequest("メールアドレスは必須です".to_string()));
        }
        
        // ユーザーを作成
        Ok(self.repository.create(&user).await?)
    }
    
    pub async fn update_user(&self, id: i64, user: UpdateUserRequest) -> Result<User, ServiceError> {
        let updated_user = self.repository.update(id, &user).await?;
        
        updated_user.ok_or(ServiceError::UserNotFound(id))
    }
    
    pub async fn delete_user(&self, id: i64) -> Result<bool, ServiceError> {
        let deleted = self.repository.delete(id).await?;
        
        if !deleted {
            return Err(ServiceError::UserNotFound(id));
        }
        
        Ok(true)
    }
}

pub struct AppState {
    pub user_service: UserService,
}

// handlers.rs
pub mod user {
    use actix_web::{web, HttpResponse, ResponseError};
    use serde::Serialize;
    use crate::models::{CreateUserRequest, UpdateUserRequest};
    use crate::services::{AppState, ServiceError};
    
    #[derive(Serialize)]
    struct ErrorResponse {
        error: String,
    }
    
    impl ResponseError for ServiceError {
        fn error_response(&self) -> HttpResponse {
            match self {
                ServiceError::UserNotFound(_) => {
                    HttpResponse::NotFound().json(ErrorResponse {
                        error: self.to_string(),
                    })
                }
                ServiceError::InvalidRequest(_) => {
                    HttpResponse::BadRequest().json(ErrorResponse {
                        error: self.to_string(),
                    })
                }
                _ => {
                    log::error!("内部サーバーエラー: {:?}", self);
                    HttpResponse::InternalServerError().json(ErrorResponse {
                        error: "内部サーバーエラー".to_string(),
                    })
                }
            }
        }
    }
    
    pub async fn get_users(state: web::Data<AppState>) -> Result<HttpResponse, ServiceError> {
        let users = state.user_service.get_users().await?;
        Ok(HttpResponse::Ok().json(users))
    }
    
    pub async fn get_user(
        state: web::Data<AppState>,
        path: web::Path<(i64,)>,
    ) -> Result<HttpResponse, ServiceError> {
        let user_id = path.0;
        let user = state.user_service.get_user(user_id).await?;
        Ok(HttpResponse::Ok().json(user))
    }
    
    pub async fn create_user(
        state: web::Data<AppState>,
        user: web::Json<CreateUserRequest>,
    ) -> Result<HttpResponse, ServiceError> {
        let user = state.user_service.create_user(user.into_inner()).await?;
        Ok(HttpResponse::Created().json(user))
    }
    
    pub async fn update_user(
        state: web::Data<AppState>,
        path: web::Path<(i64,)>,
        user: web::Json<UpdateUserRequest>,
    ) -> Result<HttpResponse, ServiceError> {
        let user_id = path.0;
        let user = state.user_service.update_user(user_id, user.into_inner()).await?;
        Ok(HttpResponse::Ok().json(user))
    }
    
    pub async fn delete_user(
        state: web::Data<AppState>,
        path: web::Path<(i64,)>,
    ) -> Result<HttpResponse, ServiceError> {
        let user_id = path.0;
        state.user_service.delete_user(user_id).await?;
        Ok(HttpResponse::NoContent().finish())
    }
}
```

### 例3: テスト駆動開発（TDD）を使用したライブラリ

以下は、テスト駆動開発（TDD）を使用したライブラリの例です：

```rust
// lib.rs
pub mod calculator;

// calculator.rs
#[derive(Debug, PartialEq)]
pub enum CalculatorError {
    DivisionByZero,
    Overflow,
}

pub struct Calculator;

impl Calculator {
    pub fn new() -> Self {
        Calculator
    }
    
    pub fn add(&self, a: i32, b: i32) -> Result<i32, CalculatorError> {
        a.checked_add(b).ok_or(CalculatorError::Overflow)
    }
    
    pub fn subtract(&self, a: i32, b: i32) -> Result<i32, CalculatorError> {
        a.checked_sub(b).ok_or(CalculatorError::Overflow)
    }
    
    pub fn multiply(&self, a: i32, b: i32) -> Result<i32, CalculatorError> {
        a.checked_mul(b).ok_or(CalculatorError::Overflow)
    }
    
    pub fn divide(&self, a: i32, b: i32) -> Result<i32, CalculatorError> {
        if b == 0 {
            return Err(CalculatorError::DivisionByZero);
        }
        
        a.checked_div(b).ok_or(CalculatorError::Overflow)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_add() {
        let calculator = Calculator::new();
        
        assert_eq!(calculator.add(2, 3), Ok(5));
        assert_eq!(calculator.add(-1, 1), Ok(0));
        assert_eq!(calculator.add(0, 0), Ok(0));
        
        // オーバーフローのテスト
        assert_eq!(calculator.add(i32::MAX, 1), Err(CalculatorError::Overflow));
    }
    
    #[test]
    fn test_subtract() {
        let calculator = Calculator::new();
        
        assert_eq!(calculator.subtract(5, 3), Ok(2));
        assert_eq!(calculator.subtract(1, 1), Ok(0));
        assert_eq!(calculator.subtract(0, 0), Ok(0));
        
        // オーバーフローのテスト
        assert_eq!(calculator.subtract(i32::MIN, 1), Err(CalculatorError::Overflow));
    }
    
    #[test]
    fn test_multiply() {
        let calculator = Calculator::new();
        
        assert_eq!(calculator.multiply(2, 3), Ok(6));
        assert_eq!(calculator.multiply(-1, 1), Ok(-1));
        assert_eq!(calculator.multiply(0, 0), Ok(0));
        
        // オーバーフローのテスト
        assert_eq!(calculator.multiply(i32::MAX, 2), Err(CalculatorError::Overflow));
    }
    
    #[test]
    fn test_divide() {
        let calculator = Calculator::new();
        
        assert_eq!(calculator.divide(6, 3), Ok(2));
        assert_eq!(calculator.divide(-6, 3), Ok(-2));
        assert_eq!(calculator.divide(0, 1), Ok(0));
        
        // ゼロ除算のテスト
        assert_eq!(calculator.divide(1, 0), Err(CalculatorError::DivisionByZero));
        
        // オーバーフローのテスト
        assert_eq!(calculator.divide(i32::MIN, -1), Err(CalculatorError::Overflow));
    }
}

// tests/integration_test.rs
use calculator_lib::calculator::{Calculator, CalculatorError};

#[test]
fn test_calculator_operations() {
    let calculator = Calculator::new();
    
    // 複数の操作を組み合わせたテスト
    let a = 10;
    let b = 5;
    
    let sum = calculator.add(a, b).unwrap();
    assert_eq!(sum, 15);
    
    let product = calculator.multiply(sum, 2).unwrap();
    assert_eq!(product, 30);
    
    let difference = calculator.subtract(product, a).unwrap();
    assert_eq!(difference, 20);
    
    let quotient = calculator.divide(difference, b).unwrap();
    assert_eq!(quotient, 4);
}

#[test]
fn test_error_handling() {
    let calculator = Calculator::new();
    
    // エラー処理のテスト
    match calculator.divide(10, 0) {
        Ok(_) => panic!("ゼロ除算が成功しました"),
        Err(e) => assert_eq!(e, CalculatorError::DivisionByZero),
    }
    
    // Result型のメソッドを使用したエラー処理
    let result = calculator.divide(10, 0)
        .and_then(|n| calculator.multiply(n, 2))
        .and_then(|n| calculator.add(n, 5));
    
    assert_eq!(result, Err(CalculatorError::DivisionByZero));
}
```

## PHP/Laravel開発者向けのポイント

### PHPのプロジェクト構造とRustのプロジェクト構造の比較

PHPのLaravelプロジェクトとRustのCargoプロジェクトの構造を比較します：

#### Laravelプロジェクト構造

```
laravel-project/
├── app/                  # アプリケーションコード
│   ├── Console/          # コンソールコマンド
│   ├── Exceptions/       # 例外ハンドラ
│   ├── Http/             # コントローラ、ミドルウェア
│   ├── Models/           # Eloquentモデル
│   ├── Providers/        # サービスプロバイダ
│   └── Services/         # サービスクラス
├── bootstrap/            # アプリケーション起動ファイル
├── config/               # 設定ファイル
├── database/             # マイグレーション、シード
├── public/               # 公開ディレクトリ
├── resources/            # ビュー、アセット
├── routes/               # ルート定義
├── storage/              # キャッシュ、ログ
├── tests/                # テスト
├── vendor/               # 依存関係
├── .env                  # 環境変数
└── composer.json         # 依存関係定義
```

#### Rustプロジェクト構造

```
rust-project/
├── src/                  # ソースコード
│   ├── main.rs           # バイナリのエントリーポイント
│   ├── lib.rs            # ライブラリのルートモジュール
│   ├── bin/              # 複数のバイナリ
│   ├── models/           # データモデル
│   ├── services/         # ビジネスロジック
│   ├── repositories/     # データアクセス
│   └── handlers/         # リクエストハンドラ
├── tests/                # 統合テスト
├── benches/              # ベンチマーク
├── examples/             # 使用例
├── target/               # ビルド成果物
├── .env                  # 環境変数
└── Cargo.toml            # 依存関係定義
```

主な違い：

1. **ディレクトリ構造**:
   - Laravelは、フレームワークによって厳格に定義された構造を持つ
   - Rustは、Cargoによって基本的な構造が提供されるが、内部構造はより柔軟

2. **モジュール化**:
   - Laravelは、名前空間とディレクトリ構造によってモジュール化される
   - Rustは、モジュールシステムによってモジュール化される

3. **設定管理**:
   - Laravelは、`config`ディレクトリに設定ファイルを配置する
   - Rustは、通常、外部クレート（`config`など）を使用して設定を管理する

### PHPのテストとRustのテストの比較

PHPのPHPUnitとRustの組み込みテストフレームワークを比較します：

#### PHPUnitのテスト

```php
<?php
// tests/CalculatorTest.php
use PHPUnit\Framework\TestCase;
use App\Calculator;

class CalculatorTest extends TestCase
{
    private $calculator;
    
    protected function setUp(): void
    {
        $this->calculator = new Calculator();
    }
    
    public function testAdd()
    {
        $this->assertEquals(5, $this->calculator->add(2, 3));
        $this->assertEquals(0, $this->calculator->add(-1, 1));
        $this->assertEquals(0, $this->calculator->add(0, 0));
    }
    
    public function testDivide()
    {
        $this->assertEquals(2, $this->calculator->divide(6, 3));
        $this->assertEquals(-2, $this->calculator->divide(-6, 3));
        $this->assertEquals(0, $this->calculator->divide(0, 1));
    }
    
    public function testDivideByZero()
    {
        $this->expectException(\DivisionByZeroError::class);
        $this->calculator->divide(1, 0);
    }
}
```

#### Rustのテスト

```rust
// src/calculator.rs
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_add() {
        let calculator = Calculator::new();
        
        assert_eq!(calculator.add(2, 3), Ok(5));
        assert_eq!(calculator.add(-1, 1), Ok(0));
        assert_eq!(calculator.add(0, 0), Ok(0));
    }
    
    #[test]
    fn test_divide() {
        let calculator = Calculator::new();
        
        assert_eq!(calculator.divide(6, 3), Ok(2));
        assert_eq!(calculator.divide(-6, 3), Ok(-2));
        assert_eq!(calculator.divide(0, 1), Ok(0));
    }
    
    #[test]
    fn test_divide_by_zero() {
        let calculator = Calculator::new();
        
        assert_eq!(calculator.divide(1, 0), Err(CalculatorError::DivisionByZero));
    }
}
```

主な違い：

1. **テストの配置**:
   - PHPUnitでは、テストは通常、別のディレクトリに配置される
   - Rustでは、ユニットテストはコードと同じファイルに配置され、統合テストは`tests`ディレクトリに配置される

2. **テストの実行**:
   - PHPUnitでは、`phpunit`コマンドを使用してテストを実行する
   - Rustでは、`cargo test`コマンドを使用してテストを実行する

3. **アサーション**:
   - PHPUnitでは、`assertEquals`などのメソッドを使用する
   - Rustでは、`assert_eq!`などのマクロを使用する

4. **例外処理**:
   - PHPでは、例外をスローしてエラーを処理する
   - Rustでは、`Result`型を使用してエラーを処理する

### Laravelのルーティングとミドルウェアとの対比

Laravelのルーティングとミドルウェアを、Rustのwebフレームワーク（例：`actix-web`）と比較します：

#### Laravelのルーティングとミドルウェア

```php
<?php
// routes/api.php
use App\Http\Controllers\UserController;

Route::middleware('auth:api')->group(function () {
    Route::get('/users', [UserController::class, 'index']);
    Route::post('/users', [UserController::class, 'store']);
    Route::get('/users/{id}', [UserController::class, 'show']);
    Route::put('/users/{id}', [UserController::class, 'update']);
    Route::delete('/users/{id}', [UserController::class, 'destroy']);
});

// app/Http/Middleware/Authenticate.php
namespace App\Http\Middleware;

use Illuminate\Auth\Middleware\Authenticate as Middleware;

class Authenticate extends Middleware
{
    protected function redirectTo($request)
    {
        if (! $request->expectsJson()) {
            return route('login');
        }
    }
}

// app/Http/Controllers/UserController.php
namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;

class UserController extends Controller
{
    public function index()
    {
        return User::all();
    }
    
    public function store(Request $request)
    {
        $validated = $request->validate([
            'name' => 'required|string|max:255',
            'email' => 'required|string|email|max:255|unique:users',
        ]);
        
        return User::create($validated);
    }
    
    public function show($id)
    {
        return User::findOrFail($id);
    }
    
    public function update(Request $request, $id)
    {
        $user = User::findOrFail($id);
        
        $validated = $request->validate([
            'name' => 'string|max:255',
            'email' => 'string|email|max:255|unique:users,email,' . $id,
        ]);
        
        $user->update($validated);
        
        return $user;
    }
    
    public function destroy($id)
    {
        $user = User::findOrFail($id);
        $user->delete();
        
        return response()->noContent();
    }
}
```

#### Rustの`actix-web`によるルーティングとミドルウェア

```rust
// main.rs
use actix_web::{web, App, HttpServer, middleware};
use actix_web_httpauth::middleware::HttpAuthentication;
use actix_web_httpauth::extractors::bearer::BearerAuth;

mod handlers;
mod models;
mod auth;

async fn validator(req: actix_web::dev::ServiceRequest, credentials: BearerAuth) -> Result<actix_web::dev::ServiceRequest, actix_web::Error> {
    let token = credentials.token();
    
    if auth::validate_token(token) {
        Ok(req)
    } else {
        Err(actix_web::error::ErrorUnauthorized("Invalid token"))
    }
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        let auth = HttpAuthentication::bearer(validator);
        
        App::new()
            .wrap(middleware::Logger::default())
            .service(
                web::scope("/api")
                    .wrap(auth)
                    .service(
                        web::scope("/users")
                            .route("", web::get().to(handlers::user::get_users))
                            .route("", web::post().to(handlers::user::create_user))
                            .route("/{id}", web::get().to(handlers::user::get_user))
                            .route("/{id}", web::put().to(handlers::user::update_user))
                            .route("/{id}", web::delete().to(handlers::user::delete_user))
                    )
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}

// handlers/user.rs
use actix_web::{web, HttpResponse, ResponseError};
use serde::{Deserialize, Serialize};
use validator::Validate;

#[derive(Debug, Serialize, Deserialize, Validate)]
struct CreateUserRequest {
    #[validate(length(min = 1, max = 255))]
    name: String,
    
    #[validate(email)]
    email: String,
}

#[derive(Debug, Serialize, Deserialize, Validate)]
struct UpdateUserRequest {
    #[validate(length(min = 1, max = 255))]
    name: Option<String>,
    
    #[validate(email)]
    email: Option<String>,
}

pub async fn get_users() -> HttpResponse {
    // ユーザー一覧を取得
    HttpResponse::Ok().json(vec![/* ユーザー一覧 */])
}

pub async fn create_user(user: web::Json<CreateUserRequest>) -> HttpResponse {
    // バリデーション
    if let Err(errors) = user.validate() {
        return HttpResponse::BadRequest().json(errors);
    }
    
    // ユーザーを作成
    HttpResponse::Created().json(/* 作成されたユーザー */)
}

pub async fn get_user(path: web::Path<(i64,)>) -> HttpResponse {
    let user_id = path.0;
    
    // ユーザーを取得
    HttpResponse::Ok().json(/* ユーザー */)
}

pub async fn update_user(
    path: web::Path<(i64,)>,
    user: web::Json<UpdateUserRequest>,
) -> HttpResponse {
    let user_id = path.0;
    
    // バリデーション
    if let Err(errors) = user.validate() {
        return HttpResponse::BadRequest().json(errors);
    }
    
    // ユーザーを更新
    HttpResponse::Ok().json(/* 更新されたユーザー */)
}

pub async fn delete_user(path: web::Path<(i64,)>) -> HttpResponse {
    let user_id = path.0;
    
    // ユーザーを削除
    HttpResponse::NoContent().finish()
}
```

主な違い：

1. **ルーティング**:
   - Laravelでは、ルートはグローバルな`Route`ファサードを使用して定義される
   - Rustでは、ルートはアプリケーションの構築時に定義される

2. **ミドルウェア**:
   - Laravelでは、ミドルウェアはクラスとして定義され、ルートに適用される
   - Rustでは、ミドルウェアは関数として定義され、アプリケーションまたはスコープに適用される

3. **コントローラ**:
   - Laravelでは、コントローラはクラスとして定義され、複数のアクションを持つ
   - Rustでは、ハンドラは個別の関数として定義される

4. **バリデーション**:
   - Laravelでは、バリデーションはリクエストクラスまたはコントローラ内で行われる
   - Rustでは、バリデーションはハンドラ内で行われるか、専用のクレート（`validator`など）を使用する

### 実践的なアプリケーション開発の移行ポイント

1. **プロジェクト構造の設計**:
   - PHPのディレクトリ構造からRustのモジュール構造への移行
   - 責任の分離と関心の分離の原則を維持

2. **テスト駆動開発の適用**:
   - PHPUnitのテストからRustのテストへの移行
   - テストファーストの開発アプローチの維持

3. **エラー処理の戦略**:
   - PHPの例外処理からRustの`Result`型への移行
   - 構造化されたエラー型の設計

4. **ロギングと監視**:
   - Laravelのログシステムからtracing/logクレートへの移行
   - 構造化ロギングの実装

5. **設定管理**:
   - Laravelの設定システムからRustの設定クレートへの移行
   - 環境変数と設定ファイルの統合

6. **データベースアクセス**:
   - Eloquent ORMからsqlxやdieselなどのクレートへの移行
   - リポジトリパターンの実装

7. **APIの設計**:
   - RESTful APIの原則の維持
   - OpenAPIやSwaggerによるAPI文書化

## 演習問題

### 演習1: 構造化されたCLIアプリケーションの開発

**目標**: `clap`と`config`を使用して、構造化されたCLIアプリケーションを開発する

**要件**:
1. コマンドライン引数を処理するためのCLIアプリケーションを作成する
2. 設定ファイルから設定を読み込む機能を実装する
3. 環境変数から設定を上書きする機能を実装する
4. ログ出力を実装する
5. サブコマンドを実装する

**ヒント**:
- `clap`クレートを使用して、コマンドライン引数を処理する
- `config`クレートを使用して、設定ファイルと環境変数から設定を読み込む
- `log`クレートと`env_logger`クレートを使用して、ログ出力を実装する
- モジュール化された構造を使用して、コードを整理する

**スケルトンコード**:

```rust
use clap::{App, Arg, SubCommand};
use config::{Config, ConfigError, File, Environment};
use serde::Deserialize;
use log::{info, error, warn, debug};

#[derive(Debug, Deserialize)]
struct AppConfig {
    // TODO: 設定構造体を定義する
}

fn load_config(config_path: &str) -> Result<AppConfig, ConfigError> {
    // TODO: 設定を読み込む関数を実装する
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // TODO: コマンドライン引数を処理する
    // TODO: ロガーを初期化する
    // TODO: 設定を読み込む
    // TODO: サブコマンドを処理する
    
    Ok(())
}
```

### 演習2: レイヤードアーキテクチャを使用したWebアプリケーション

**目標**: `actix-web`と`sqlx`を使用して、レイヤードアーキテクチャを持つWebアプリケーションを開発する

**要件**:
1. ユーザーリソースのCRUD操作を提供するRESTful APIを実装する
2. レイヤードアーキテクチャ（ハンドラ、サービス、リポジトリ）を使用する
3. データベース接続を実装する
4. エラー処理を実装する
5. ロギングを実装する

**ヒント**:
- `actix-web`クレートを使用して、Webアプリケーションを実装する
- `sqlx`クレートを使用して、データベース接続を実装する
- `thiserror`クレートを使用して、エラー型を定義する
- `tracing`クレートを使用して、ロギングを実装する
- 責任の分離と関心の分離の原則に従って、コードを構造化する

**スケルトンコード**:

```rust
use actix_web::{web, App, HttpServer, HttpResponse};
use sqlx::{postgres::PgPoolOptions, Pool, Postgres};
use serde::{Deserialize, Serialize};
use thiserror::Error;

// モデル
#[derive(Debug, Serialize, Deserialize, sqlx::FromRow)]
struct User {
    // TODO: ユーザーモデルを定義する
}

// リクエスト/レスポンス
#[derive(Debug, Deserialize)]
struct CreateUserRequest {
    // TODO: ユーザー作成リクエストを定義する
}

// エラー
#[derive(Error, Debug)]
enum AppError {
    // TODO: アプリケーションエラーを定義する
}

// リポジトリ
struct UserRepository {
    // TODO: ユーザーリポジトリを定義する
}

// サービス
struct UserService {
    // TODO: ユーザーサービスを定義する
}

// ハンドラ
mod handlers {
    // TODO: ハンドラを定義する
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // TODO: ロガーを初期化する
    // TODO: 設定を読み込む
    // TODO: データベース接続プールを作成する
    // TODO: リポジトリとサービスを作成する
    // TODO: HTTPサーバーを起動する
    
    Ok(())
}
```

### 演習3: テスト駆動開発を使用したライブラリ

**目標**: テスト駆動開発（TDD）を使用して、ライブラリを開発する

**要件**:
1. テストファーストの開発アプローチを使用する
2. ユニットテストと統合テストを実装する
3. エラー処理を実装する
4. ドキュメントコメントを追加する
5. ベンチマークを実装する

**ヒント**:
- テストを先に書いてから、実装を行う
- `#[cfg(test)]`属性を使用して、ユニットテストを実装する
- `tests`ディレクトリに統合テストを配置する
- `Result`型を使用して、エラー処理を実装する
- `///`コメントを使用して、ドキュメントコメントを追加する
- `criterion`クレートを使用して、ベンチマークを実装する

**スケルトンコード**:

```rust
// lib.rs
pub mod calculator;

// calculator.rs
#[derive(Debug, PartialEq)]
pub enum CalculatorError {
    // TODO: エラー型を定義する
}

pub struct Calculator;

impl Calculator {
    // TODO: 電卓の機能を実装する
}

#[cfg(test)]
mod tests {
    // TODO: ユニットテストを実装する
}

// tests/integration_test.rs
// TODO: 統合テストを実装する

// benches/benchmark.rs
// TODO: ベンチマークを実装する
```

これらの演習問題を通じて、Rustの実践的なアプリケーション開発ワークフローを学ぶことができます。プロジェクト構造、テスト駆動開発、エラー処理、ロギング、設定管理など、実際のアプリケーション開発に必要なスキルを習得することが重要です。

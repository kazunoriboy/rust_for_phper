# Rust TODOアプリチュートリアル：PHP開発者向け

## はじめに

このチュートリアルでは、PHPからRustへの移行を学ぶために、シンプルなTODOアプリを作成します。このプロジェクトを通じて、Rustの基本的な概念を実践的に学び、実際のアプリケーション開発の流れを理解することができます。

## プロジェクトの概要

作成するTODOアプリは以下の機能を持ちます：

1. タスクの追加
2. タスクの一覧表示
3. タスクの完了/未完了の切り替え
4. タスクの削除
5. タスクのファイルへの保存と読み込み

## 前提条件

このチュートリアルを始める前に、以下のツールがインストールされていることを確認してください：

- Rust（rustc、cargo）
- テキストエディタまたはIDE

Rustのインストールは、[公式サイト](https://www.rust-lang.org/tools/install)の手順に従ってください。

## ステップ1：プロジェクトのセットアップ

まず、新しいRustプロジェクトを作成します。

```bash
cargo new todo_app
cd todo_app
```

これにより、`todo_app`ディレクトリが作成され、基本的なプロジェクト構造が生成されます。

## ステップ2：データモデルの定義

まず、TODOアプリのデータモデルを定義します。`src/main.rs`を開き、以下のコードを追加します：

```rust
use std::fs;
use std::io::{self, Write};
use std::path::Path;
use std::time::{SystemTime, UNIX_EPOCH};

// タスクを表す構造体
#[derive(Debug, Clone)]
struct Task {
    id: u64,
    title: String,
    completed: bool,
}

impl Task {
    // 新しいタスクを作成するコンストラクタ
    fn new(title: String) -> Self {
        let id = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs();
        
        Task {
            id,
            title,
            completed: false,
        }
    }
    
    // タスクの完了状態を切り替えるメソッド
    fn toggle_completed(&mut self) {
        self.completed = !self.completed;
    }
}

// タスクのコレクションを管理する構造体
struct TaskManager {
    tasks: Vec<Task>,
}

impl TaskManager {
    // 新しいタスクマネージャーを作成するコンストラクタ
    fn new() -> Self {
        TaskManager {
            tasks: Vec::new(),
        }
    }
    
    // タスクを追加するメソッド
    fn add_task(&mut self, title: String) {
        let task = Task::new(title);
        self.tasks.push(task);
    }
    
    // タスクを削除するメソッド
    fn remove_task(&mut self, id: u64) -> Result<(), String> {
        let index = self.tasks.iter().position(|task| task.id == id);
        
        match index {
            Some(idx) => {
                self.tasks.remove(idx);
                Ok(())
            }
            None => Err(format!("ID {}のタスクが見つかりません", id)),
        }
    }
    
    // タスクの完了状態を切り替えるメソッド
    fn toggle_task(&mut self, id: u64) -> Result<(), String> {
        let task = self.tasks.iter_mut().find(|task| task.id == id);
        
        match task {
            Some(task) => {
                task.toggle_completed();
                Ok(())
            }
            None => Err(format!("ID {}のタスクが見つかりません", id)),
        }
    }
    
    // すべてのタスクを表示するメソッド
    fn list_tasks(&self) {
        if self.tasks.is_empty() {
            println!("タスクはありません");
            return;
        }
        
        println!("タスク一覧:");
        for task in &self.tasks {
            let status = if task.completed { "[x]" } else { "[ ]" };
            println!("{} {} - {}", task.id, status, task.title);
        }
    }
    
    // タスクをファイルに保存するメソッド
    fn save_to_file(&self, filename: &str) -> io::Result<()> {
        let mut content = String::new();
        
        for task in &self.tasks {
            content.push_str(&format!("{},{},{}\n", task.id, task.title, task.completed));
        }
        
        fs::write(filename, content)
    }
    
    // ファイルからタスクを読み込むメソッド
    fn load_from_file(&mut self, filename: &str) -> io::Result<()> {
        if !Path::new(filename).exists() {
            return Ok(());
        }
        
        let content = fs::read_to_string(filename)?;
        self.tasks.clear();
        
        for line in content.lines() {
            let parts: Vec<&str> = line.split(',').collect();
            if parts.len() >= 3 {
                let id = parts[0].parse::<u64>().unwrap_or(0);
                let title = parts[1].to_string();
                let completed = parts[2].parse::<bool>().unwrap_or(false);
                
                self.tasks.push(Task {
                    id,
                    title,
                    completed,
                });
            }
        }
        
        Ok(())
    }
}

fn main() {
    // メイン関数は後で実装します
}
```

このコードでは、以下の要素を定義しています：

1. `Task`構造体：タスクのID、タイトル、完了状態を保持します
2. `TaskManager`構造体：タスクのコレクションを管理し、追加、削除、表示などの操作を提供します
3. ファイル操作のメソッド：タスクをファイルに保存したり、ファイルから読み込んだりする機能を提供します

## ステップ3：コマンドラインインターフェースの実装

次に、ユーザーがコマンドラインからアプリケーションを操作できるようにします。`Cargo.toml`ファイルを開き、依存関係を追加します：

```toml
[package]
name = "todo_app"
version = "0.1.0"
edition = "2021"

[dependencies]
clap = { version = "4.0", features = ["derive"] }
```

次に、`src/main.rs`の`main`関数を以下のように実装します：

```rust
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(author, version, about, long_about = None)]
struct Cli {
    #[command(subcommand)]
    command: Option<Commands>,
}

#[derive(Subcommand)]
enum Commands {
    /// タスクを追加
    Add {
        /// タスクのタイトル
        #[arg(required = true)]
        title: String,
    },
    /// タスクを削除
    Remove {
        /// タスクのID
        #[arg(required = true)]
        id: u64,
    },
    /// タスクの完了状態を切り替え
    Toggle {
        /// タスクのID
        #[arg(required = true)]
        id: u64,
    },
    /// タスク一覧を表示
    List,
}

fn main() {
    let cli = Cli::parse();
    let filename = "tasks.txt";
    let mut task_manager = TaskManager::new();
    
    // ファイルからタスクを読み込む
    if let Err(err) = task_manager.load_from_file(filename) {
        eprintln!("ファイルの読み込みに失敗しました: {}", err);
    }
    
    match &cli.command {
        Some(Commands::Add { title }) => {
            task_manager.add_task(title.clone());
            println!("タスクを追加しました: {}", title);
        }
        Some(Commands::Remove { id }) => {
            match task_manager.remove_task(*id) {
                Ok(()) => println!("ID {}のタスクを削除しました", id),
                Err(err) => eprintln!("エラー: {}", err),
            }
        }
        Some(Commands::Toggle { id }) => {
            match task_manager.toggle_task(*id) {
                Ok(()) => println!("ID {}のタスクの状態を切り替えました", id),
                Err(err) => eprintln!("エラー: {}", err),
            }
        }
        Some(Commands::List) => {
            task_manager.list_tasks();
        }
        None => {
            // コマンドが指定されていない場合はタスク一覧を表示
            task_manager.list_tasks();
        }
    }
    
    // タスクをファイルに保存
    if let Err(err) = task_manager.save_to_file(filename) {
        eprintln!("ファイルの保存に失敗しました: {}", err);
    }
}
```

このコードでは、`clap`クレートを使用してコマンドラインインターフェースを実装しています。ユーザーは以下のコマンドを使用できます：

- `add`：新しいタスクを追加
- `remove`：指定したIDのタスクを削除
- `toggle`：指定したIDのタスクの完了状態を切り替え
- `list`：すべてのタスクを表示

## ステップ4：アプリケーションのビルドと実行

プロジェクトをビルドして実行します：

```bash
cargo build
```

これにより、アプリケーションがビルドされます。次に、以下のコマンドでアプリケーションを実行できます：

```bash
# タスクを追加
cargo run -- add "牛乳を買う"
cargo run -- add "レポートを書く"

# タスク一覧を表示
cargo run -- list

# タスクの完了状態を切り替え
cargo run -- toggle <タスクID>

# タスクを削除
cargo run -- remove <タスクID>
```

## ステップ5：対話的なインターフェースの追加（オプション）

コマンドラインインターフェースに加えて、対話的なインターフェースを追加することもできます。`src/main.rs`に以下のコードを追加します：

```rust
fn interactive_mode(task_manager: &mut TaskManager, filename: &str) {
    println!("TODOアプリ - 対話モード");
    println!("コマンド: add <タイトル>, list, toggle <ID>, remove <ID>, quit");
    
    loop {
        print!("> ");
        io::stdout().flush().unwrap();
        
        let mut input = String::new();
        io::stdin().read_line(&mut input).unwrap();
        let input = input.trim();
        
        let parts: Vec<&str> = input.splitn(2, ' ').collect();
        let command = parts[0];
        
        match command {
            "add" => {
                if parts.len() < 2 || parts[1].is_empty() {
                    println!("エラー: タイトルを指定してください");
                    continue;
                }
                
                let title = parts[1].to_string();
                task_manager.add_task(title.clone());
                println!("タスクを追加しました: {}", title);
            }
            "list" => {
                task_manager.list_tasks();
            }
            "toggle" => {
                if parts.len() < 2 || parts[1].is_empty() {
                    println!("エラー: IDを指定してください");
                    continue;
                }
                
                match parts[1].parse::<u64>() {
                    Ok(id) => {
                        match task_manager.toggle_task(id) {
                            Ok(()) => println!("ID {}のタスクの状態を切り替えました", id),
                            Err(err) => println!("エラー: {}", err),
                        }
                    }
                    Err(_) => println!("エラー: 無効なID"),
                }
            }
            "remove" => {
                if parts.len() < 2 || parts[1].is_empty() {
                    println!("エラー: IDを指定してください");
                    continue;
                }
                
                match parts[1].parse::<u64>() {
                    Ok(id) => {
                        match task_manager.remove_task(id) {
                            Ok(()) => println!("ID {}のタスクを削除しました", id),
                            Err(err) => println!("エラー: {}", err),
                        }
                    }
                    Err(_) => println!("エラー: 無効なID"),
                }
            }
            "quit" => {
                break;
            }
            _ => {
                println!("不明なコマンド: {}", command);
                println!("コマンド: add <タイトル>, list, toggle <ID>, remove <ID>, quit");
            }
        }
        
        // タスクをファイルに保存
        if let Err(err) = task_manager.save_to_file(filename) {
            eprintln!("ファイルの保存に失敗しました: {}", err);
        }
    }
}
```

そして、`main`関数に以下のコードを追加します：

```rust
fn main() {
    let cli = Cli::parse();
    let filename = "tasks.txt";
    let mut task_manager = TaskManager::new();
    
    // ファイルからタスクを読み込む
    if let Err(err) = task_manager.load_from_file(filename) {
        eprintln!("ファイルの読み込みに失敗しました: {}", err);
    }
    
    // 対話モードのフラグを追加
    if let Some(Commands::Interactive) = &cli.command {
        interactive_mode(&mut task_manager, filename);
        return;
    }
    
    // 既存のコマンド処理...
}
```

また、`Commands`列挙型に新しいコマンドを追加します：

```rust
#[derive(Subcommand)]
enum Commands {
    // 既存のコマンド...
    
    /// 対話モードを開始
    Interactive,
}
```

これにより、`cargo run -- interactive`コマンドで対話モードを開始できるようになります。

## ステップ6：機能の拡張

基本的なTODOアプリが完成しましたが、さらに機能を拡張することもできます。以下はいくつかのアイデアです：

1. タスクの優先度を追加
2. タスクのカテゴリ分け
3. タスクの期限設定
4. タスクの検索機能
5. JSONやSQLiteなどを使用したデータ保存

例えば、タスクに優先度を追加するには、`Task`構造体を以下のように変更します：

```rust
#[derive(Debug, Clone)]
struct Task {
    id: u64,
    title: String,
    completed: bool,
    priority: Priority,
}

#[derive(Debug, Clone, PartialEq)]
enum Priority {
    Low,
    Medium,
    High,
}

impl Task {
    fn new(title: String, priority: Priority) -> Self {
        let id = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap()
            .as_secs();
        
        Task {
            id,
            title,
            completed: false,
            priority,
        }
    }
    
    // 既存のメソッド...
}
```

そして、`TaskManager`の`add_task`メソッドと関連するコードを更新します。

## ステップ7：テストの追加

最後に、アプリケーションの信頼性を確保するためにテストを追加します。`src/main.rs`の末尾に以下のコードを追加します：

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_add_task() {
        let mut manager = TaskManager::new();
        manager.add_task("テストタスク".to_string());
        assert_eq!(manager.tasks.len(), 1);
        assert_eq!(manager.tasks[0].title, "テストタスク");
        assert_eq!(manager.tasks[0].completed, false);
    }
    
    #[test]
    fn test_toggle_task() {
        let mut manager = TaskManager::new();
        manager.add_task("テストタスク".to_string());
        let id = manager.tasks[0].id;
        
        // タスクの状態を切り替え
        let result = manager.toggle_task(id);
        assert!(result.is_ok());
        assert_eq!(manager.tasks[0].completed, true);
        
        // 存在しないタスクの状態を切り替え
        let result = manager.toggle_task(999);
        assert!(result.is_err());
    }
    
    #[test]
    fn test_remove_task() {
        let mut manager = TaskManager::new();
        manager.add_task("テストタスク1".to_string());
        manager.add_task("テストタスク2".to_string());
        let id = manager.tasks[0].id;
        
        // タスクを削除
        let result = manager.remove_task(id);
        assert!(result.is_ok());
        assert_eq!(manager.tasks.len(), 1);
        assert_eq!(manager.tasks[0].title, "テストタスク2");
        
        // 存在しないタスクを削除
        let result = manager.remove_task(999);
        assert!(result.is_err());
    }
    
    #[test]
    fn test_file_operations() {
        let mut manager = TaskManager::new();
        manager.add_task("テストタスク1".to_string());
        manager.add_task("テストタスク2".to_string());
        
        // ファイルに保存
        let filename = "test_tasks.txt";
        let result = manager.save_to_file(filename);
        assert!(result.is_ok());
        
        // 新しいマネージャーを作成してファイルから読み込み
        let mut new_manager = TaskManager::new();
        let result = new_manager.load_from_file(filename);
        assert!(result.is_ok());
        assert_eq!(new_manager.tasks.len(), 2);
        
        // テスト後にファイルを削除
        let _ = fs::remove_file(filename);
    }
}
```

テストを実行するには、以下のコマンドを使用します：

```bash
cargo test
```

## PHPとRustの比較

PHPでTODOアプリを作成する場合と比較して、Rustでの実装には以下のような違いがあります：

1. **型システム**：Rustは静的型付け言語であり、コンパイル時に型エラーを検出します。PHPは動的型付け言語であり、実行時に型エラーが発生する可能性があります。

2. **メモリ管理**：Rustは所有権システムを使用してメモリを管理し、ガベージコレクションを必要としません。PHPはガベージコレクションを使用してメモリを管理します。

3. **エラー処理**：Rustは`Result`型を使用してエラーを明示的に処理します。PHPは例外を使用してエラーを処理します。

4. **パフォーマンス**：Rustはネイティブコードにコンパイルされるため、PHPよりも高速に実行できます。

5. **コンパイル vs インタープリタ**：Rustはコンパイル言語であり、コードを実行する前にコンパイルする必要があります。PHPはインタープリタ言語であり、コードを直接実行できます。

## まとめ

このチュートリアルでは、RustでシンプルなTODOアプリを作成する方法を学びました。以下の概念を実践的に理解することができました：

1. Rustの基本的な構文と構造体
2. ファイル操作
3. コマンドラインインターフェース
4. エラー処理
5. テスト

このプロジェクトをベースに、さらに機能を追加したり、ウェブアプリケーションに拡張したりすることもできます。Rustの学習を続け、より複雑なアプリケーションを開発していきましょう。

## 次のステップ

TODOアプリをさらに発展させるためのアイデア：

1. **ウェブインターフェースの追加**：`warp`や`rocket`などのウェブフレームワークを使用して、ウェブインターフェースを追加する
2. **データベースの使用**：ファイルの代わりにSQLiteやPostgreSQLなどのデータベースを使用する
3. **ユーザー認証**：複数のユーザーがそれぞれのタスクを管理できるようにする
4. **RESTful API**：他のアプリケーションと連携できるAPIを提供する

Rustの学習を続けるためのリソース：
- [The Rust Programming Language](https://doc.rust-lang.org/book/)（公式ドキュメント）
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)（例を通じて学ぶ）
- [Rustlings](https://github.com/rust-lang/rustlings)（小さな演習問題）
- [Rust Cookbook](https://rust-lang-nursery.github.io/rust-cookbook/)（実用的なレシピ集）

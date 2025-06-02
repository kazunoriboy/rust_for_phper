# トピック2: 並行処理とマルチスレッドプログラミング

## 学習目標

このセクションを学ぶことで、以下のことができるようになります：

- Rustのスレッドを作成し、効率的に管理する
- メッセージパッシングを使用してスレッド間で安全に通信する
- 共有状態を同期プリミティブ（Mutex、RwLock、Arc）を使って安全に管理する
- Rayonクレートを使用してデータ並列処理を実装する
- 非同期プログラミングの深い理解を得て、tokioやasync-stdなどのランタイムを活用する

## 主要な概念の説明

### スレッドの基本概念

Rustでは、標準ライブラリの`std::thread`モジュールを使用して、OSレベルのスレッドを作成できます。スレッドは並行処理の基本単位であり、同時に複数の処理を実行するために使用されます。

```rust
use std::thread;
use std::time::Duration;

fn main() {
    // 新しいスレッドを生成
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("スレッド内: {}", i);
            thread::sleep(Duration::from_millis(1));
        }
    });

    // メインスレッドでの処理
    for i in 1..5 {
        println!("メインスレッド: {}", i);
        thread::sleep(Duration::from_millis(1));
    }

    // スレッドの終了を待機
    handle.join().unwrap();
}
```

`thread::spawn`は新しいスレッドを生成し、クロージャ内のコードを実行します。戻り値の`JoinHandle`を使用して、スレッドの終了を待機できます。

#### スレッドとクロージャのキャプチャ

スレッド内のクロージャは、環境の変数をキャプチャできます。ただし、Rustの所有権システムにより、いくつかの制約があります：

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    // エラー: `v`の所有権がスレッドに移動するが、メインスレッドでも使用される
    // let handle = thread::spawn(|| {
    //     println!("ベクトル: {:?}", v);
    // });
    
    // println!("メインスレッドでのベクトル: {:?}", v);

    // 正しい方法: moveキーワードを使用して所有権を明示的に移動
    let handle = thread::spawn(move || {
        println!("ベクトル: {:?}", v);
    });

    // ここでは`v`は使用できない（所有権がスレッドに移動している）
    
    handle.join().unwrap();
}
```

`move`キーワードを使用すると、クロージャは環境の変数の所有権を取得します。これにより、スレッドが安全に変数にアクセスできるようになります。

### メッセージパッシングによる通信

スレッド間の通信には、メッセージパッシングが効果的です。Rustの標準ライブラリには、`std::sync::mpsc`（Multiple Producer, Single Consumer）チャネルが用意されています。

```rust
use std::thread;
use std::sync::mpsc;

fn main() {
    // チャネルを作成（送信者と受信者のペア）
    let (tx, rx) = mpsc::channel();

    // 別のスレッドで送信者を使用
    thread::spawn(move || {
        let messages = vec![
            "こんにちは".to_string(),
            "from".to_string(),
            "the".to_string(),
            "thread".to_string(),
        ];

        for msg in messages {
            tx.send(msg).unwrap();
            thread::sleep(std::time::Duration::from_millis(100));
        }
    });

    // メインスレッドで受信者を使用
    for received in rx {
        println!("受信: {}", received);
    }
}
```

このコードでは、1つのスレッドがメッセージを送信し、別のスレッド（この場合はメインスレッド）がそれらを受信します。`rx`をイテレートすると、チャネルが閉じられるまでブロックされます。

#### 複数の送信者

`mpsc::channel`は、複数の送信者と1つの受信者をサポートしています：

```rust
use std::thread;
use std::sync::mpsc;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();
    
    // 送信者をクローンして複数のスレッドで使用
    let tx1 = tx.clone();
    thread::spawn(move || {
        let messages = vec!["A1".to_string(), "A2".to_string(), "A3".to_string()];
        for msg in messages {
            tx1.send(msg).unwrap();
            thread::sleep(Duration::from_millis(100));
        }
    });
    
    thread::spawn(move || {
        let messages = vec!["B1".to_string(), "B2".to_string(), "B3".to_string()];
        for msg in messages {
            tx.send(msg).unwrap();
            thread::sleep(Duration::from_millis(150));
        }
    });
    
    // 両方のスレッドからのメッセージを受信
    for received in rx {
        println!("受信: {}", received);
    }
}
```

#### 同期チャネルと非同期チャネル

標準の`mpsc::channel`は非同期チャネルで、送信者はメッセージを送信した後、受信者がそれを処理するのを待たずに続行できます。一方、`mpsc::sync_channel`は同期チャネルで、バッファサイズを超えるメッセージを送信しようとすると、受信者がメッセージを処理するまでブロックされます：

```rust
use std::thread;
use std::sync::mpsc;
use std::time::Duration;

fn main() {
    // バッファサイズ2の同期チャネルを作成
    let (tx, rx) = mpsc::sync_channel(2);
    
    thread::spawn(move || {
        let messages = vec![1, 2, 3, 4, 5];
        for msg in messages {
            println!("送信: {}", msg);
            tx.send(msg).unwrap();
            println!("送信完了: {}", msg);
        }
    });
    
    thread::sleep(Duration::from_secs(2)); // 受信を遅らせる
    
    for received in rx {
        println!("受信: {}", received);
        thread::sleep(Duration::from_millis(500));
    }
}
```

このコードでは、送信者は最初の2つのメッセージをすぐに送信できますが、3つ目のメッセージを送信しようとすると、受信者が少なくとも1つのメッセージを処理するまでブロックされます。

### 共有状態の同期

複数のスレッドが同じデータにアクセスする必要がある場合、共有状態を安全に管理するための同期プリミティブが必要です。

#### Mutex（相互排除）

`Mutex<T>`は、一度に1つのスレッドだけがデータにアクセスできるようにします：

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    // Mutexで保護された共有カウンター
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];
    
    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
            // ロックはスコープを抜けると自動的に解放される
        });
        handles.push(handle);
    }
    
    for handle in handles {
        handle.join().unwrap();
    }
    
    println!("結果: {}", *counter.lock().unwrap());
}
```

`Mutex::lock`メソッドは、ロックを取得し、内部データへの可変参照を返します。ロックはスコープを抜けると自動的に解放されます。

`Arc<T>`（Atomic Reference Counting）は、スレッド間で安全に共有できる参照カウントポインタです。`Rc<T>`はスレッド間で共有できないため、マルチスレッド環境では`Arc<T>`を使用する必要があります。

#### RwLock（読み書きロック）

`RwLock<T>`は、複数の読み取りまたは1つの書き込みを許可するロックです：

```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let data = Arc::new(RwLock::new(vec![1, 2, 3]));
    let mut handles = vec![];
    
    // 読み取りスレッド
    for i in 0..3 {
        let data = Arc::clone(&data);
        handles.push(thread::spawn(move || {
            let data = data.read().unwrap();
            println!("読み取りスレッド {}: {:?}", i, *data);
        }));
    }
    
    // 書き込みスレッド
    let data_clone = Arc::clone(&data);
    handles.push(thread::spawn(move || {
        let mut data = data_clone.write().unwrap();
        data.push(4);
        println!("書き込みスレッド: {:?}", *data);
    }));
    
    for handle in handles {
        handle.join().unwrap();
    }
    
    println!("最終結果: {:?}", *data.read().unwrap());
}
```

`RwLock`は、読み取りが多く書き込みが少ない場合に効率的です。複数のスレッドが同時に読み取りロックを取得できますが、書き込みロックは排他的です。

#### デッドロックの回避

デッドロックは、2つ以上のスレッドが互いにロックを待っている状態です。デッドロックを回避するためのいくつかの戦略：

1. **ロックの順序を一貫させる**: 複数のロックを取得する場合は、常に同じ順序で取得する
2. **ロックのスコープを最小限に保つ**: ロックを保持する時間を最小限にする
3. **デッドロックの検出**: タイムアウト付きのロック取得を使用する

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

fn main() {
    let resource_a = Arc::new(Mutex::new(1));
    let resource_b = Arc::new(Mutex::new(2));
    
    let a_clone = Arc::clone(&resource_a);
    let b_clone = Arc::clone(&resource_b);
    
    // スレッド1: A -> B の順でロック
    let thread1 = thread::spawn(move || {
        let _a = a_clone.lock().unwrap();
        println!("スレッド1: リソースAをロック");
        
        thread::sleep(Duration::from_millis(100)); // デッドロックを発生させやすくするための遅延
        
        let _b = b_clone.lock().unwrap();
        println!("スレッド1: リソースBをロック");
    });
    
    // スレッド2: B -> A の順でロック（デッドロックの可能性あり）
    let thread2 = thread::spawn(move || {
        // デッドロックを避けるため、A -> B の順でロックする
        let _a = resource_a.lock().unwrap();
        println!("スレッド2: リソースAをロック");
        
        let _b = resource_b.lock().unwrap();
        println!("スレッド2: リソースBをロック");
    });
    
    thread1.join().unwrap();
    thread2.join().unwrap();
}
```

### Rayonによるデータ並列処理

`rayon`クレートは、データ並列処理を簡単に実装するためのツールを提供します。内部的には、ワークスティーリングアルゴリズムを使用して効率的にタスクを分散します。

まず、依存関係を追加します：

```toml
[dependencies]
rayon = "1.5"
```

#### イテレータの並列化

`rayon`の`par_iter`と`par_iter_mut`を使用して、イテレータを並列化できます：

```rust
use rayon::prelude::*;

fn main() {
    let mut numbers: Vec<i32> = (0..100).collect();
    
    // 並列イテレーション
    numbers.par_iter_mut().for_each(|n| {
        *n = *n * 2;
    });
    
    println!("最初の10個: {:?}", &numbers[0..10]);
    
    // 並列マップ
    let squares: Vec<i32> = numbers.par_iter()
        .map(|&n| n * n)
        .collect();
    
    println!("最初の10個の二乗: {:?}", &squares[0..10]);
    
    // 並列フィルター
    let even: Vec<i32> = numbers.par_iter()
        .filter(|&&n| n % 2 == 0)
        .cloned()
        .collect();
    
    println!("最初の10個の偶数: {:?}", &even[0..10]);
}
```

#### カスタムスレッドプール

`rayon`では、カスタムスレッドプールを作成して、並列処理のリソース使用量を制御できます：

```rust
use rayon::ThreadPoolBuilder;
use std::time::Instant;

fn main() {
    // 4スレッドのカスタムプールを作成
    let pool = ThreadPoolBuilder::new()
        .num_threads(4)
        .build()
        .unwrap();
    
    let start = Instant::now();
    
    // プール内で並列処理を実行
    pool.install(|| {
        let result: u64 = (0..1_000_000u64)
            .into_par_iter()
            .map(|n| n * n)
            .sum();
        
        println!("合計: {}", result);
    });
    
    println!("実行時間: {:?}", start.elapsed());
}
```

#### 並列クイックソート

`rayon`を使用した並列クイックソートの実装例：

```rust
use rayon::prelude::*;

fn quick_sort<T: Ord + Send>(v: &mut [T]) {
    if v.len() <= 1 {
        return;
    }
    
    let mid = partition(v);
    let (lo, hi) = v.split_at_mut(mid);
    
    // 配列サイズが十分大きい場合のみ並列処理
    if v.len() > 1000 {
        rayon::join(|| quick_sort(lo), || quick_sort(hi));
    } else {
        quick_sort(lo);
        quick_sort(hi);
    }
}

fn partition<T: Ord>(v: &mut [T]) -> usize {
    let pivot = v.len() - 1;
    let mut i = 0;
    
    for j in 0..pivot {
        if v[j] <= v[pivot] {
            v.swap(i, j);
            i += 1;
        }
    }
    
    v.swap(i, pivot);
    i
}

fn main() {
    let mut v = vec![5, 8, 1, 3, 7, 9, 2, 6, 4];
    quick_sort(&mut v);
    println!("ソート結果: {:?}", v);
    
    // 大きな配列でのテスト
    let mut large_v: Vec<i32> = (0..100_000).rev().collect();
    let start = std::time::Instant::now();
    quick_sort(&mut large_v);
    println!("大きな配列のソート時間: {:?}", start.elapsed());
    assert!(large_v.windows(2).all(|w| w[0] <= w[1]));
}
```

### 非同期プログラミング

Rustの非同期プログラミングは、`async`/`await`構文と`Future`トレイトを中心に構築されています。非同期プログラミングは、I/O操作などのブロッキング操作を効率的に処理するのに役立ちます。

#### Future特性と実行モデル

`Future`は、将来的に値を生成する可能性のある計算を表します：

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

`Future`は、値が準備できるまで`Pending`を返し、準備ができたら`Ready(T)`を返します。`poll`メソッドは、`Future`の状態を確認するために呼び出されます。

`async`/`await`構文は、`Future`を扱うための糖衣構文です：

```rust
async fn fetch_data(url: &str) -> Result<String, reqwest::Error> {
    let response = reqwest::get(url).await?;
    let body = response.text().await?;
    Ok(body)
}
```

この関数は、`Future`を返します。`await`キーワードは、`Future`が完了するまで現在のタスクを一時停止します。

#### 非同期ランタイム

Rustの標準ライブラリには非同期ランタイムが含まれていないため、`tokio`や`async-std`などのクレートを使用する必要があります。

##### Tokioの基本

`tokio`は、最も広く使用されている非同期ランタイムです：

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

基本的な使用例：

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    println!("開始");
    
    let task1 = tokio::spawn(async {
        sleep(Duration::from_millis(100)).await;
        println!("タスク1完了");
        "タスク1の結果"
    });
    
    let task2 = tokio::spawn(async {
        sleep(Duration::from_millis(50)).await;
        println!("タスク2完了");
        "タスク2の結果"
    });
    
    let result1 = task1.await.unwrap();
    let result2 = task2.await.unwrap();
    
    println!("結果: {} and {}", result1, result2);
    println!("終了");
}
```

`#[tokio::main]`属性は、`main`関数を非同期関数に変換し、Tokioランタイム上で実行します。`tokio::spawn`は、新しい非同期タスクを生成します。

##### 非同期I/O

Tokioは、非同期I/O操作のための多くのユーティリティを提供します：

```rust
use tokio::fs::File;
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> io::Result<()> {
    // 非同期ファイル読み込み
    let mut file = File::open("input.txt").await?;
    let mut contents = String::new();
    file.read_to_string(&mut contents).await?;
    println!("ファイル内容: {}", contents);
    
    // 非同期ファイル書き込み
    let mut output = File::create("output.txt").await?;
    output.write_all(b"Hello, async world!").await?;
    
    Ok(())
}
```

##### チャネルを使った非同期通信

Tokioは、非同期タスク間の通信のためのチャネルも提供します：

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(100);
    
    // 送信タスク
    let tx_task = tokio::spawn(async move {
        for i in 0..10 {
            tx.send(i).await.unwrap();
            sleep(Duration::from_millis(100)).await;
        }
    });
    
    // 受信タスク
    let rx_task = tokio::spawn(async move {
        while let Some(value) = rx.recv().await {
            println!("受信: {}", value);
        }
    });
    
    // 両方のタスクが完了するのを待つ
    let _ = tokio::join!(tx_task, rx_task);
}
```

##### Select操作

`tokio::select!`マクロを使用すると、複数の非同期操作のうち最初に完了したものを選択できます：

```rust
use tokio::sync::mpsc;
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let (tx1, mut rx1) = mpsc::channel(1);
    let (tx2, mut rx2) = mpsc::channel(1);
    
    // 送信タスク1
    tokio::spawn(async move {
        sleep(Duration::from_millis(500)).await;
        tx1.send("タスク1からのメッセージ").await.unwrap();
    });
    
    // 送信タスク2
    tokio::spawn(async move {
        sleep(Duration::from_millis(200)).await;
        tx2.send("タスク2からのメッセージ").await.unwrap();
    });
    
    // 最初に完了したタスクを選択
    tokio::select! {
        msg = rx1.recv() => {
            println!("rx1から受信: {:?}", msg);
        }
        msg = rx2.recv() => {
            println!("rx2から受信: {:?}", msg);
        }
    }
}
```

#### 非同期プログラミングのベストプラクティス

1. **ブロッキング操作を避ける**: 非同期コンテキスト内でブロッキング操作（標準ライブラリのファイルI/Oなど）を実行しないでください。代わりに、非同期バージョン（`tokio::fs`など）を使用してください。

2. **適切なタスク粒度**: 非常に小さなタスクを多数生成すると、オーバーヘッドが大きくなります。適切なタスク粒度を選択してください。

3. **エラー処理**: 非同期コードでも適切なエラー処理が重要です。`?`演算子は`async`関数内でも使用できます。

4. **リソース管理**: 非同期コードでも、リソースの適切な管理（ファイルハンドルのクローズなど）が重要です。

5. **キャンセレーション**: 長時間実行される非同期タスクには、キャンセレーションメカニズムを提供することを検討してください。

## 具体的なコード例

### 例1: 並行Webクローラー

複数のWebページを並行して取得する簡単なWebクローラーを実装します：

```rust
use std::collections::{HashSet, VecDeque};
use std::sync::{Arc, Mutex};
use std::time::Instant;
use reqwest::Client;
use tokio::task;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let start_url = "https://www.rust-lang.org";
    let max_depth = 2;
    let max_pages = 10;
    
    let client = Client::new();
    let visited = Arc::new(Mutex::new(HashSet::new()));
    let queue = Arc::new(Mutex::new(VecDeque::new()));
    
    // 初期URLをキューに追加
    queue.lock().unwrap().push_back((start_url.to_string(), 0));
    visited.lock().unwrap().insert(start_url.to_string());
    
    let start_time = Instant::now();
    let mut handles = vec![];
    
    // ワーカータスクを生成
    for worker_id in 0..4 {
        let client = client.clone();
        let visited = Arc::clone(&visited);
        let queue = Arc::clone(&queue);
        
        let handle = task::spawn(async move {
            let mut pages_processed = 0;
            
            loop {
                // キューからURLを取得
                let (url, depth) = {
                    let mut queue = queue.lock().unwrap();
                    if queue.is_empty() {
                        break;
                    }
                    queue.pop_front().unwrap()
                };
                
                println!("ワーカー {}: 処理中 {} (深さ {})", worker_id, url, depth);
                
                // ページを取得
                match client.get(&url).send().await {
                    Ok(response) => {
                        if let Ok(text) = response.text().await {
                            pages_processed += 1;
                            
                            // 最大深さに達していない場合、リンクを抽出
                            if depth < max_depth {
                                let links = extract_links(&text, &url);
                                
                                let mut visited = visited.lock().unwrap();
                                let mut queue = queue.lock().unwrap();
                                
                                for link in links {
                                    if !visited.contains(&link) && visited.len() < max_pages {
                                        visited.insert(link.clone());
                                        queue.push_back((link, depth + 1));
                                    }
                                }
                            }
                        }
                    }
                    Err(e) => {
                        println!("エラー: {}: {}", url, e);
                    }
                }
                
                // 最大ページ数に達したかチェック
                if visited.lock().unwrap().len() >= max_pages {
                    break;
                }
            }
            
            pages_processed
        });
        
        handles.push(handle);
    }
    
    // すべてのワーカーが完了するのを待つ
    let mut total_pages = 0;
    for handle in handles {
        total_pages += handle.await?;
    }
    
    let elapsed = start_time.elapsed();
    println!("クロール完了: {} ページを {:.2}秒で処理", total_pages, elapsed.as_secs_f64());
    
    // 訪問したURLを表示
    println!("訪問したURL:");
    for url in visited.lock().unwrap().iter() {
        println!("- {}", url);
    }
    
    Ok(())
}

// HTMLからリンクを抽出する簡易関数
fn extract_links(html: &str, base_url: &str) -> Vec<String> {
    let mut links = Vec::new();
    let base_url = url::Url::parse(base_url).unwrap();
    
    // 非常に単純なリンク抽出（実際のクローラーではHTMLパーサーを使用すべき）
    for cap in regex::Regex::new(r#"href=["']([^"']+)["']"#).unwrap().captures_iter(html) {
        if let Some(link) = cap.get(1) {
            if let Ok(absolute_url) = base_url.join(link.as_str()) {
                if absolute_url.scheme() == "http" || absolute_url.scheme() == "https" {
                    links.push(absolute_url.to_string());
                }
            }
        }
    }
    
    links
}
```

このコードを実行するには、以下の依存関係が必要です：

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
reqwest = "0.11"
regex = "1"
url = "2"
```

### 例2: 並列画像処理

画像処理は、並列処理の恩恵を受けやすいタスクの一つです。以下は、複数の画像を並列で処理する例です：

```rust
use std::path::{Path, PathBuf};
use std::sync::Arc;
use rayon::prelude::*;
use image::{GenericImageView, ImageBuffer, Rgba};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let input_dir = "input_images";
    let output_dir = "output_images";
    
    // 出力ディレクトリを作成
    std::fs::create_dir_all(output_dir)?;
    
    // 入力ディレクトリから画像ファイルを取得
    let entries = std::fs::read_dir(input_dir)?;
    let image_paths: Vec<PathBuf> = entries
        .filter_map(Result::ok)
        .map(|entry| entry.path())
        .filter(|path| {
            let extension = path.extension().and_then(|ext| ext.to_str()).unwrap_or("");
            extension == "jpg" || extension == "png" || extension == "jpeg"
        })
        .collect();
    
    println!("処理する画像: {} 個", image_paths.len());
    
    // 画像を並列処理
    let start = std::time::Instant::now();
    let results: Vec<Result<(), String>> = image_paths.par_iter()
        .map(|path| {
            let filename = path.file_name().unwrap().to_str().unwrap();
            let output_path = Path::new(output_dir).join(filename);
            
            match process_image(path, &output_path) {
                Ok(()) => {
                    println!("処理完了: {}", filename);
                    Ok(())
                }
                Err(e) => {
                    println!("エラー: {}: {}", filename, e);
                    Err(format!("{}: {}", filename, e))
                }
            }
        })
        .collect();
    
    let elapsed = start.elapsed();
    let success_count = results.iter().filter(|r| r.is_ok()).count();
    
    println!("処理完了: {}/{} 画像を {:.2}秒で処理", 
             success_count, image_paths.len(), elapsed.as_secs_f64());
    
    Ok(())
}

// 画像処理関数（グレースケール変換とぼかし処理）
fn process_image(input_path: &Path, output_path: &Path) -> Result<(), String> {
    // 画像を読み込み
    let img = image::open(input_path)
        .map_err(|e| format!("画像の読み込みに失敗: {}", e))?;
    
    // グレースケールに変換
    let gray_img = img.grayscale();
    
    // ぼかし処理（簡易的なガウスぼかし）
    let width = gray_img.width();
    let height = gray_img.height();
    let mut blurred = ImageBuffer::new(width, height);
    
    // 各ピクセルに対してぼかし処理
    for y in 1..height-1 {
        for x in 1..width-1 {
            let mut sum = 0.0;
            let mut weight = 0.0;
            
            // 3x3カーネル
            for dy in -1..=1 {
                for dx in -1..=1 {
                    let nx = (x as i32 + dx) as u32;
                    let ny = (y as i32 + dy) as u32;
                    
                    if nx < width && ny < height {
                        let pixel = gray_img.get_pixel(nx, ny);
                        let w = if dx == 0 && dy == 0 { 4.0 } else { 1.0 };
                        sum += pixel[0] as f32 * w;
                        weight += w;
                    }
                }
            }
            
            let avg = (sum / weight) as u8;
            blurred.put_pixel(x, y, Rgba([avg, avg, avg, 255]));
        }
    }
    
    // 結果を保存
    blurred.save(output_path)
        .map_err(|e| format!("画像の保存に失敗: {}", e))?;
    
    Ok(())
}
```

このコードを実行するには、以下の依存関係が必要です：

```toml
[dependencies]
rayon = "1.5"
image = "0.24"
```

### 例3: 非同期APIサーバー

Tokioとaxumを使用した非同期APIサーバーの例：

```rust
use axum::{
    routing::{get, post},
    http::StatusCode,
    Json, Router, extract::{Path, State},
};
use serde::{Deserialize, Serialize};
use std::sync::{Arc, Mutex};
use std::collections::HashMap;
use std::net::SocketAddr;

// ユーザーモデル
#[derive(Debug, Serialize, Deserialize, Clone)]
struct User {
    id: u64,
    name: String,
    email: String,
}

// アプリケーション状態
#[derive(Clone)]
struct AppState {
    users: Arc<Mutex<HashMap<u64, User>>>,
}

#[tokio::main]
async fn main() {
    // アプリケーション状態を初期化
    let state = AppState {
        users: Arc::new(Mutex::new(HashMap::new())),
    };
    
    // サンプルユーザーを追加
    {
        let mut users = state.users.lock().unwrap();
        users.insert(1, User {
            id: 1,
            name: "Alice".to_string(),
            email: "alice@example.com".to_string(),
        });
        users.insert(2, User {
            id: 2,
            name: "Bob".to_string(),
            email: "bob@example.com".to_string(),
        });
    }
    
    // ルーターを設定
    let app = Router::new()
        .route("/users", get(get_users))
        .route("/users/:id", get(get_user))
        .route("/users", post(create_user))
        .with_state(state);
    
    // サーバーを起動
    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    println!("サーバーを起動: {}", addr);
    
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}

// すべてのユーザーを取得
async fn get_users(State(state): State<AppState>) -> Json<Vec<User>> {
    let users = state.users.lock().unwrap();
    let users_vec: Vec<User> = users.values().cloned().collect();
    Json(users_vec)
}

// 特定のユーザーを取得
async fn get_user(
    Path(id): Path<u64>,
    State(state): State<AppState>,
) -> Result<Json<User>, StatusCode> {
    let users = state.users.lock().unwrap();
    
    if let Some(user) = users.get(&id) {
        Ok(Json(user.clone()))
    } else {
        Err(StatusCode::NOT_FOUND)
    }
}

// 新しいユーザーを作成
async fn create_user(
    State(state): State<AppState>,
    Json(mut user): Json<User>,
) -> Result<Json<User>, StatusCode> {
    let mut users = state.users.lock().unwrap();
    
    // 新しいIDを生成
    let new_id = users.keys().max().unwrap_or(&0) + 1;
    user.id = new_id;
    
    users.insert(new_id, user.clone());
    Ok(Json(user))
}
```

このコードを実行するには、以下の依存関係が必要です：

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
axum = "0.6"
serde = { version = "1.0", features = ["derive"] }
```

## PHP/Laravel開発者向けのポイント

### PHPのプロセスモデルとRustの並行処理の違い

PHPのWebアプリケーション（特にApache+mod_phpやPHP-FPM）は、リクエストごとに新しいPHPプロセスまたはスレッドが処理を担当します。各リクエストは独立しており、共有状態は明示的に管理する必要があります（セッション、データベースなど）。

```php
// PHPでの並行処理（各リクエストは独立）
// request1.php
<?php
session_start();
$_SESSION['count'] = ($_SESSION['count'] ?? 0) + 1;
echo "Count: " . $_SESSION['count'];
```

Rustでは、1つのプロセス内で複数のスレッドまたは非同期タスクを使用して並行処理を実現します。共有状態は明示的に管理する必要があります。

```rust
// Rustでの並行処理
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];
    
    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        let handle = thread::spawn(move || {
            let mut count = counter.lock().unwrap();
            *count += 1;
            println!("Count: {}", *count);
        });
        handles.push(handle);
    }
    
    for handle in handles {
        handle.join().unwrap();
    }
}
```

### Laravelのキューシステムとの比較

Laravelのキューシステムは、非同期処理のための仕組みを提供します：

```php
// Laravelでのジョブ定義
class ProcessPodcast implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    protected $podcast;
    
    public function __construct(Podcast $podcast)
    {
        $this->podcast = $podcast;
    }
    
    public function handle()
    {
        // ポッドキャストを処理...
    }
}

// ジョブのディスパッチ
ProcessPodcast::dispatch($podcast);
```

Rustでは、非同期処理は言語レベルでサポートされています：

```rust
use tokio::sync::mpsc;

#[derive(Debug)]
struct Podcast {
    id: u64,
    title: String,
}

async fn process_podcast(podcast: Podcast) {
    println!("ポッドキャスト処理中: {}", podcast.title);
    // 処理ロジック...
    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
    println!("ポッドキャスト処理完了: {}", podcast.title);
}

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(100);
    
    // ワーカータスク
    let worker = tokio::spawn(async move {
        while let Some(podcast) = rx.recv().await {
            process_podcast(podcast).await;
        }
    });
    
    // ジョブをキューに追加
    let podcasts = vec![
        Podcast { id: 1, title: "Rust Talk".to_string() },
        Podcast { id: 2, title: "Async Programming".to_string() },
        Podcast { id: 3, title: "Concurrency Patterns".to_string() },
    ];
    
    for podcast in podcasts {
        tx.send(podcast).await.unwrap();
    }
    
    // 送信者をドロップしてチャネルを閉じる
    drop(tx);
    
    // ワーカーが完了するのを待つ
    worker.await.unwrap();
}
```

### ReactPHPとの対比

ReactPHPは、PHPでイベント駆動の非同期プログラミングを可能にするライブラリです：

```php
// ReactPHPでの非同期HTTP要求
$loop = React\EventLoop\Factory::create();
$client = new React\Http\Browser($loop);

$client->get('https://api.example.com/users')
    ->then(
        function (Psr\Http\Message\ResponseInterface $response) {
            echo $response->getBody();
        },
        function (Exception $error) {
            echo 'Error: ' . $error->getMessage();
        }
    );

$loop->run();
```

Rustの非同期プログラミングは、より統合されており、`async`/`await`構文を使用します：

```rust
use reqwest;

async fn fetch_users() -> Result<(), Box<dyn std::error::Error>> {
    let response = reqwest::get("https://api.example.com/users").await?;
    let body = response.text().await?;
    println!("{}", body);
    Ok(())
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    fetch_users().await?;
    Ok(())
}
```

### 並行処理の移行ポイント

1. **プロセスモデルからスレッドモデルへの移行**:
   - PHPでは各リクエストが独立したプロセス/スレッドで処理される
   - Rustでは1つのプロセス内で複数のスレッド/タスクを管理する

2. **共有状態の管理**:
   - PHPではセッションやデータベースを使用して状態を共有
   - Rustでは`Arc`、`Mutex`などを使用して明示的に共有状態を管理

3. **非同期処理の考え方**:
   - PHPではコールバックやPromiseベースの非同期処理
   - Rustでは`async`/`await`を使用した構造化された非同期処理

4. **リソース効率**:
   - PHPは各リクエストに対して新しいプロセス/スレッドを使用
   - Rustは少数のスレッドで多数の非同期タスクを処理できる

5. **エラー処理**:
   - PHPでは例外を使用してエラーを処理
   - Rustでは`Result`型を使用して非同期コンテキストでもエラーを処理

## 演習問題

### 演習1: スレッドプールの実装

**目標**: 基本的なスレッドプールを実装し、タスクを並行して実行する

**要件**:
1. 指定された数のワーカースレッドを持つスレッドプールを実装する
2. タスク（クロージャ）をプールに送信できるようにする
3. 結果を取得できるようにする
4. プールの終了処理を実装する

**ヒント**:
- `std::thread`と`std::sync::mpsc`を使用する
- タスクは`FnOnce() -> T + Send + 'static`のようなトレイト境界を持つクロージャとして表現できる
- 結果を返すために、別のチャネルを使用する

**スケルトンコード**:

```rust
use std::thread;
use std::sync::mpsc;

// スレッドプールの実装
struct ThreadPool<T> {
    workers: Vec<Worker>,
    sender: mpsc::Sender<Job<T>>,
}

// ワーカースレッド
struct Worker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>,
}

// ジョブ型（クロージャとその結果を送信するためのチャネル）
type Job<T> = Box<dyn FnOnce() -> T + Send + 'static>;

impl<T: Send + 'static> ThreadPool<T> {
    // 新しいスレッドプールを作成
    fn new(size: usize) -> (Self, mpsc::Receiver<T>) {
        // TODO: スレッドプールを初期化し、結果を受信するためのチャネルを作成
        
        unimplemented!()
    }
    
    // タスクをプールに送信
    fn execute<F>(&self, f: F)
    where
        F: FnOnce() -> T + Send + 'static,
    {
        // TODO: タスクをワーカースレッドに送信
        
        unimplemented!()
    }
}

impl<T> Drop for ThreadPool<T> {
    fn drop(&mut self) {
        // TODO: すべてのワーカースレッドを適切に終了させる
        
        unimplemented!()
    }
}

impl Worker {
    // 新しいワーカーを作成
    fn new<T: Send + 'static>(
        id: usize,
        receiver: mpsc::Receiver<Job<T>>,
        result_sender: mpsc::Sender<T>,
    ) -> Self {
        // TODO: ワーカースレッドを作成し、ジョブを処理する
        
        unimplemented!()
    }
}

fn main() {
    // スレッドプールを作成
    let (pool, results) = ThreadPool::new(4);
    
    // タスクを送信
    for i in 0..8 {
        pool.execute(move || {
            println!("処理中: タスク {}", i);
            thread::sleep(std::time::Duration::from_secs(1));
            i
        });
    }
    
    // 結果を収集
    drop(pool); // すべてのタスクが送信されたらプールをドロップ
    
    let mut results_vec = Vec::new();
    while let Ok(result) = results.recv() {
        results_vec.push(result);
    }
    
    println!("結果: {:?}", results_vec);
}
```

### 演習2: 並列データ処理パイプライン

**目標**: 複数のステージからなるデータ処理パイプラインを実装する

**要件**:
1. 以下のステージを持つデータ処理パイプラインを実装する:
   - データ生成: ランダムな数値のストリームを生成
   - フィルタリング: 特定の条件を満たすデータのみを通過させる
   - 変換: データに対して何らかの変換を適用
   - 集約: 結果を集計する

2. 各ステージは並行して動作し、チャネルを通じてデータを受け渡す
3. パイプラインの処理速度を測定する

**ヒント**:
- `std::thread`と`std::sync::mpsc`を使用する
- 各ステージを別々のスレッドで実行する
- バッファサイズを調整して、パフォーマンスへの影響を観察する

**スケルトンコード**:

```rust
use std::thread;
use std::sync::mpsc;
use std::time::Instant;
use rand::Rng;

fn main() {
    let start = Instant::now();
    let data_count = 10_000;
    
    // TODO: データ生成ステージ
    // ランダムな数値を生成し、次のステージに送信
    
    // TODO: フィルタリングステージ
    // 偶数のみを通過させる
    
    // TODO: 変換ステージ
    // 各数値を2倍にする
    
    // TODO: 集約ステージ
    // 結果の合計を計算
    
    let elapsed = start.elapsed();
    println!("処理時間: {:?}", elapsed);
}

// データ生成関数
fn generate_data(count: usize, sender: mpsc::Sender<i32>) {
    // TODO: ランダムなデータを生成し、チャネルに送信
    
    unimplemented!()
}

// フィルタリング関数
fn filter_data(receiver: mpsc::Receiver<i32>, sender: mpsc::Sender<i32>) {
    // TODO: 条件を満たすデータのみを次のステージに送信
    
    unimplemented!()
}

// 変換関数
fn transform_data(receiver: mpsc::Receiver<i32>, sender: mpsc::Sender<i32>) {
    // TODO: データを変換して次のステージに送信
    
    unimplemented!()
}

// 集約関数
fn aggregate_data(receiver: mpsc::Receiver<i32>) -> i32 {
    // TODO: 結果を集計して返す
    
    unimplemented!()
}
```

### 演習3: 非同期Webクライアント

**目標**: 複数のAPIエンドポイントから並行してデータを取得し、結果を集約する非同期Webクライアントを実装する

**要件**:
1. 複数のAPIエンドポイントから並行してデータを取得する
2. 取得したデータを集約して表示する
3. タイムアウト処理を実装する
4. エラー処理を適切に行う

**ヒント**:
- `tokio`と`reqwest`クレートを使用する
- `tokio::spawn`を使用して複数のリクエストを並行して実行する
- `tokio::select!`を使用してタイムアウト処理を実装する
- `tokio::try_join!`を使用して複数のタスクの結果を集約する

**スケルトンコード**:

```rust
use std::time::Duration;
use tokio::time;
use reqwest;
use serde::Deserialize;
use anyhow::{Result, Context};

// ユーザーデータ構造体
#[derive(Debug, Deserialize)]
struct User {
    id: i32,
    name: String,
    email: String,
}

// 投稿データ構造体
#[derive(Debug, Deserialize)]
struct Post {
    id: i32,
    title: String,
    body: String,
    userId: i32,
}

// コメントデータ構造体
#[derive(Debug, Deserialize)]
struct Comment {
    id: i32,
    postId: i32,
    name: String,
    email: String,
    body: String,
}

#[tokio::main]
async fn main() -> Result<()> {
    let base_url = "https://jsonplaceholder.typicode.com";
    let timeout = Duration::from_secs(5);
    
    println!("データ取得開始...");
    let start = std::time::Instant::now();
    
    // TODO: ユーザー、投稿、コメントを並行して取得
    // 各APIエンドポイントからデータを取得し、タイムアウト処理を実装
    
    // TODO: 取得したデータを集約
    // 例: 各ユーザーの投稿数とコメント数を表示
    
    let elapsed = start.elapsed();
    println!("処理時間: {:?}", elapsed);
    
    Ok(())
}

// ユーザーを取得する関数
async fn fetch_users(base_url: &str) -> Result<Vec<User>> {
    // TODO: ユーザーデータを取得
    
    unimplemented!()
}

// 投稿を取得する関数
async fn fetch_posts(base_url: &str) -> Result<Vec<Post>> {
    // TODO: 投稿データを取得
    
    unimplemented!()
}

// コメントを取得する関数
async fn fetch_comments(base_url: &str) -> Result<Vec<Comment>> {
    // TODO: コメントデータを取得
    
    unimplemented!()
}

// タイムアウト付きでタスクを実行する関数
async fn with_timeout<T>(
    task: impl std::future::Future<Output = Result<T>>,
    timeout_duration: Duration,
) -> Result<T> {
    // TODO: タイムアウト処理を実装
    
    unimplemented!()
}
```

これらの演習問題を通じて、Rustの並行処理と非同期プログラミングの概念を実践的に学ぶことができます。各演習は、実際のアプリケーション開発で遭遇する可能性のある状況を模しています。

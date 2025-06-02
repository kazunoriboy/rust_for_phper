# トピック8: WebAssembly (Wasm) への挑戦

## 学習目標

このセクションを学ぶことで、以下のことができるようになります：

- WebAssembly (Wasm) の基本概念と利点を理解する
- RustコードをWebAssemblyにコンパイルし、Webブラウザで実行する
- `wasm-bindgen`を使用してRustとJavaScript間でデータをやり取りする
- WebAssemblyを使用したWebアプリケーションを開発する
- PHP/Laravelアプリケーションと連携するWasmモジュールを作成する

## 主要な概念の説明

### WebAssemblyの基本

WebAssembly（略してWasm）は、モダンなWebブラウザで動作する新しい低レベルのバイナリフォーマットです。C、C++、Rustなどの言語で書かれたコードをコンパイルし、ブラウザ上で近ネイティブの速度で実行することができます。

#### WebAssemblyの主な特徴

1. **高速実行**: 最適化されたバイナリフォーマットで、JavaScriptよりも高速に実行できる
2. **安全性**: サンドボックス環境で実行され、ホストシステムへの直接アクセスはない
3. **言語非依存**: 様々なプログラミング言語からコンパイル可能
4. **互換性**: 主要なWebブラウザ（Chrome、Firefox、Safari、Edge）でサポートされている
5. **小さなバイナリサイズ**: 効率的なバイナリエンコーディングにより、ダウンロードサイズが小さい

#### WebAssemblyのユースケース

- 計算負荷の高い処理（画像/動画処理、暗号化、物理シミュレーションなど）
- ゲーム開発
- デスクトップアプリケーションのWeb移植
- パフォーマンスクリティカルなWebアプリケーション
- 既存のC/C++/Rustライブラリの再利用

### RustからWebAssemblyへのコンパイル

RustはWebAssemblyへのコンパイルを一級市民としてサポートしており、優れたツールチェーンを提供しています。

#### 必要なツール

1. **Rustツールチェーン**: rustup, cargo
2. **wasm32ターゲット**: `rustup target add wasm32-unknown-unknown`
3. **wasm-pack**: `cargo install wasm-pack`

#### 基本的なコンパイル手順

1. Wasmをターゲットとするプロジェクトを作成
2. 必要なクレートと依存関係を追加
3. `wasm-pack build`コマンドでコンパイル
4. 生成されたWasmモジュールをWebアプリケーションで使用

```bash
# Wasmプロジェクトの作成
cargo new --lib hello-wasm
cd hello-wasm

# Cargo.tomlの設定
# [lib]
# crate-type = ["cdylib", "rlib"]
#
# [dependencies]
# wasm-bindgen = "0.2"

# ビルド
wasm-pack build --target web
```

### wasm-bindgenの使用

`wasm-bindgen`は、RustとJavaScript間のインターフェースを簡単に作成するためのツールです。これにより、複雑なデータ型の受け渡しや関数の呼び出しが可能になります。

#### 基本的な使用方法

```rust
use wasm_bindgen::prelude::*;

// JavaScriptから呼び出し可能な関数
#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

// JavaScriptの関数を呼び出す
#[wasm_bindgen]
extern "C" {
    fn alert(s: &str);
}

// JavaScriptのalert関数を使用する関数
#[wasm_bindgen]
pub fn greet(name: &str) {
    alert(&format!("Hello, {}!", name));
}
```

#### 複雑なデータ型の受け渡し

```rust
use wasm_bindgen::prelude::*;

// JavaScriptに公開する構造体
#[wasm_bindgen]
pub struct Point {
    x: f64,
    y: f64,
}

#[wasm_bindgen]
impl Point {
    // コンストラクタ
    #[wasm_bindgen(constructor)]
    pub fn new(x: f64, y: f64) -> Point {
        Point { x, y }
    }
    
    // メソッド
    pub fn distance_from_origin(&self) -> f64 {
        (self.x * self.x + self.y * self.y).sqrt()
    }
    
    // ゲッター
    pub fn x(&self) -> f64 {
        self.x
    }
    
    pub fn y(&self) -> f64 {
        self.y
    }
    
    // セッター
    pub fn set_x(&mut self, x: f64) {
        self.x = x;
    }
    
    pub fn set_y(&mut self, y: f64) {
        self.y = y;
    }
}
```

#### JavaScriptオブジェクトの操作

```rust
use wasm_bindgen::prelude::*;
use js_sys::{Array, Object, Reflect};
use web_sys::{Document, Element, HtmlElement, Window};

#[wasm_bindgen]
pub fn manipulate_dom() -> Result<(), JsValue> {
    // windowオブジェクトを取得
    let window = web_sys::window().expect("windowオブジェクトが見つかりません");
    
    // documentオブジェクトを取得
    let document = window.document().expect("documentオブジェクトが見つかりません");
    
    // 新しい要素を作成
    let p = document.create_element("p")?;
    p.set_text_content(Some("Rustから作成された段落です！"));
    
    // bodyに追加
    let body = document.body().expect("bodyが見つかりません");
    body.append_child(&p)?;
    
    Ok(())
}
```

### WebAssemblyとJavaScriptの連携

WebAssemblyは単独で使用されることはほとんどなく、通常はJavaScriptと連携して使用されます。

#### WebAssemblyモジュールの読み込み

```javascript
// ES Modules方式
import init, { add, greet } from './pkg/hello_wasm.js';

async function run() {
  // Wasmモジュールの初期化
  await init();
  
  // Rust関数の呼び出し
  console.log(add(1, 2)); // 3
  greet("World"); // "Hello, World!"
}

run();
```

#### パフォーマンスの考慮事項

1. **データの受け渡し**: JavaScriptとWasm間のデータの受け渡しにはオーバーヘッドがあるため、頻繁な小さなデータの受け渡しは避ける
2. **メモリ管理**: 大きなデータセットを扱う場合は、Wasmのメモリに直接アクセスする方が効率的
3. **計算負荷の高い処理**: 計算負荷の高い処理をWasmに移行することで、最大のパフォーマンス向上が得られる

```rust
use wasm_bindgen::prelude::*;

// 大きな配列の処理（効率的）
#[wasm_bindgen]
pub fn process_array(data: &[u8]) -> Vec<u8> {
    // 大量のデータを一度に処理
    data.iter().map(|&x| x * 2).collect()
}

// メモリを直接操作（より効率的）
#[wasm_bindgen]
pub fn process_memory() -> *const u8 {
    // 結果を格納するメモリを確保
    let result = vec![1, 2, 3, 4, 5];
    // メモリをリークさせないように注意（実際のコードではもっと複雑な管理が必要）
    let ptr = result.as_ptr();
    std::mem::forget(result);
    ptr
}
```

### WebAssemblyアプリケーションの開発

#### プロジェクト構造

典型的なRust+Wasmプロジェクトの構造は以下のようになります：

```
my-wasm-app/
├── Cargo.toml          # Rustの依存関係
├── src/                # Rustソースコード
│   └── lib.rs          # Wasmにコンパイルされるコード
├── www/                # Webアプリケーション
│   ├── index.html      # HTMLエントリーポイント
│   ├── index.js        # JavaScriptエントリーポイント
│   └── package.json    # JavaScriptの依存関係
└── tests/              # テスト
```

#### 開発ワークフロー

1. Rustコードを作成・編集
2. `wasm-pack build`でWasmにコンパイル
3. Webアプリケーションで生成されたモジュールを使用
4. ブラウザでテスト
5. 必要に応じて1に戻る

#### テンプレートの使用

`wasm-pack`は、新しいプロジェクトを簡単に始めるためのテンプレートを提供しています：

```bash
# テンプレートを使用して新しいプロジェクトを作成
cargo generate --git https://github.com/rustwasm/wasm-pack-template.git --name my-wasm-project
cd my-wasm-project

# ビルド
wasm-pack build

# Webアプリケーションテンプレートを作成
npm init wasm-app www
cd www
npm install

# 生成されたWasmパッケージをリンク
npm link ../pkg

# 開発サーバーを起動
npm start
```

### WebAssemblyのテストとデバッグ

#### ユニットテスト

Rustのユニットテストは通常通り書くことができます：

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
    }
}
```

#### wasm-packによるテスト

`wasm-pack test`コマンドを使用して、ブラウザ環境でテストを実行できます：

```bash
# Chromeでテストを実行
wasm-pack test --chrome

# Firefoxでテストを実行
wasm-pack test --firefox

# ヘッドレスブラウザでテストを実行
wasm-pack test --headless
```

#### デバッグ

WebAssemblyのデバッグは、通常のRustコードよりも複雑ですが、いくつかの方法があります：

1. **コンソールログ**: `web_sys::console::log_1`を使用してブラウザのコンソールにログを出力
2. **デバッガー**: Chromeの開発者ツールでWasmコードをデバッグ
3. **エラーハンドリング**: `Result<T, JsValue>`を使用して、エラーを適切に処理

```rust
use wasm_bindgen::prelude::*;
use web_sys::console;

// コンソールログを使用したデバッグ
#[wasm_bindgen]
pub fn debug_function(value: i32) -> Result<i32, JsValue> {
    console::log_1(&JsValue::from_str(&format!("Input value: {}", value)));
    
    if value < 0 {
        return Err(JsValue::from_str("値は0以上である必要があります"));
    }
    
    let result = value * 2;
    console::log_1(&JsValue::from_str(&format!("Result: {}", result)));
    
    Ok(result)
}
```

### WebAssemblyの最適化

#### サイズの最適化

Wasmバイナリのサイズを小さくするための方法：

1. **リリースビルド**: `--release`フラグを使用
2. **LTO (Link Time Optimization)**: Cargo.tomlで`lto = true`を設定
3. **コードサイズの最適化**: `opt-level = 'z'`を設定
4. **不要な機能の無効化**: 使用しないクレートの機能を無効化

```toml
# Cargo.toml
[profile.release]
opt-level = 'z'     # サイズ最適化
lto = true          # リンク時最適化
codegen-units = 1   # 並列コード生成を無効化
panic = 'abort'     # パニック時のスタックトレース生成を無効化
```

#### パフォーマンスの最適化

Wasmのパフォーマンスを向上させるための方法：

1. **SIMD命令の使用**: `wasm-bindgen`の`--target no-modules`オプションを使用
2. **メモリ管理の最適化**: 不要なメモリコピーを避ける
3. **アルゴリズムの最適化**: 計算量の少ないアルゴリズムを選択
4. **並列処理**: Web Workersと組み合わせて並列処理を実現

```rust
// SIMDを使用した最適化（実験的機能）
#[cfg(target_feature = "simd128")]
use wasm_bindgen::prelude::*;

#[cfg(target_feature = "simd128")]
#[wasm_bindgen]
pub fn sum_vector_simd(v: &[f32]) -> f32 {
    use std::arch::wasm32::*;
    
    let mut sum = f32x4_splat(0.0);
    let mut i = 0;
    
    // 4要素ずつ処理
    while i + 4 <= v.len() {
        let chunk = f32x4_load(&v[i] as *const f32);
        sum = f32x4_add(sum, chunk);
        i += 4;
    }
    
    // 残りの要素を処理
    let mut result = f32x4_extract_lane::<0>(sum) + 
                    f32x4_extract_lane::<1>(sum) + 
                    f32x4_extract_lane::<2>(sum) + 
                    f32x4_extract_lane::<3>(sum);
    
    while i < v.len() {
        result += v[i];
        i += 1;
    }
    
    result
}
```

## 具体的なコード例

### 例1: 画像処理アプリケーション

以下は、Rustで実装した画像処理関数をWebAssemblyにコンパイルし、Webブラウザで使用する例です：

```rust
// lib.rs
use wasm_bindgen::prelude::*;
use web_sys::{ImageData, CanvasRenderingContext2d};

// グレースケール変換
#[wasm_bindgen]
pub fn grayscale(ctx: &CanvasRenderingContext2d, width: u32, height: u32) -> Result<(), JsValue> {
    // キャンバスからピクセルデータを取得
    let data = ctx.get_image_data(0.0, 0.0, width as f64, height as f64)?;
    let mut pixels = data.data();
    
    // グレースケール変換を適用
    for i in (0..pixels.len()).step_by(4) {
        let r = pixels[i] as f64;
        let g = pixels[i + 1] as f64;
        let b = pixels[i + 2] as f64;
        
        // 輝度を計算 (ITU-R BT.709 の係数を使用)
        let gray = (0.2126 * r + 0.7152 * g + 0.0722 * b) as u8;
        
        pixels[i] = gray;
        pixels[i + 1] = gray;
        pixels[i + 2] = gray;
        // アルファチャンネル (pixels[i + 3]) はそのまま
    }
    
    // 変換後のデータをキャンバスに戻す
    let new_image_data = ImageData::new_with_u8_clamped_array_and_sh(
        wasm_bindgen::Clamped(&mut pixels),
        width,
        height
    )?;
    
    ctx.put_image_data(&new_image_data, 0.0, 0.0)?;
    
    Ok(())
}

// セピア調変換
#[wasm_bindgen]
pub fn sepia(ctx: &CanvasRenderingContext2d, width: u32, height: u32) -> Result<(), JsValue> {
    let data = ctx.get_image_data(0.0, 0.0, width as f64, height as f64)?;
    let mut pixels = data.data();
    
    for i in (0..pixels.len()).step_by(4) {
        let r = pixels[i] as f64;
        let g = pixels[i + 1] as f64;
        let b = pixels[i + 2] as f64;
        
        // セピア変換の係数
        let new_r = (0.393 * r + 0.769 * g + 0.189 * b).min(255.0) as u8;
        let new_g = (0.349 * r + 0.686 * g + 0.168 * b).min(255.0) as u8;
        let new_b = (0.272 * r + 0.534 * g + 0.131 * b).min(255.0) as u8;
        
        pixels[i] = new_r;
        pixels[i + 1] = new_g;
        pixels[i + 2] = new_b;
    }
    
    let new_image_data = ImageData::new_with_u8_clamped_array_and_sh(
        wasm_bindgen::Clamped(&mut pixels),
        width,
        height
    )?;
    
    ctx.put_image_data(&new_image_data, 0.0, 0.0)?;
    
    Ok(())
}

// 輝度調整
#[wasm_bindgen]
pub fn adjust_brightness(
    ctx: &CanvasRenderingContext2d,
    width: u32,
    height: u32,
    factor: f64
) -> Result<(), JsValue> {
    let data = ctx.get_image_data(0.0, 0.0, width as f64, height as f64)?;
    let mut pixels = data.data();
    
    for i in (0..pixels.len()).step_by(4) {
        pixels[i] = ((pixels[i] as f64 * factor).min(255.0)) as u8;
        pixels[i + 1] = ((pixels[i + 1] as f64 * factor).min(255.0)) as u8;
        pixels[i + 2] = ((pixels[i + 2] as f64 * factor).min(255.0)) as u8;
    }
    
    let new_image_data = ImageData::new_with_u8_clamped_array_and_sh(
        wasm_bindgen::Clamped(&mut pixels),
        width,
        height
    )?;
    
    ctx.put_image_data(&new_image_data, 0.0, 0.0)?;
    
    Ok(())
}
```

対応するJavaScriptコード：

```javascript
// index.js
import init, { grayscale, sepia, adjust_brightness } from './pkg/image_processor.js';

async function run() {
  // Wasmモジュールを初期化
  await init();
  
  const canvas = document.getElementById('canvas');
  const ctx = canvas.getContext('2d');
  const img = new Image();
  
  img.onload = function() {
    canvas.width = img.width;
    canvas.height = img.height;
    ctx.drawImage(img, 0, 0);
    
    // フィルターボタンのイベントリスナーを設定
    document.getElementById('grayscale').addEventListener('click', () => {
      // 元の画像を描画
      ctx.drawImage(img, 0, 0);
      // グレースケールフィルターを適用
      grayscale(ctx, canvas.width, canvas.height);
    });
    
    document.getElementById('sepia').addEventListener('click', () => {
      ctx.drawImage(img, 0, 0);
      sepia(ctx, canvas.width, canvas.height);
    });
    
    document.getElementById('brightness').addEventListener('click', () => {
      ctx.drawImage(img, 0, 0);
      const factor = parseFloat(document.getElementById('brightness-factor').value);
      adjust_brightness(ctx, canvas.width, canvas.height, factor);
    });
  };
  
  img.src = 'example.jpg';
}

run();
```

HTML：

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Wasm Image Processor</title>
  <style>
    body { font-family: sans-serif; }
    .controls { margin: 20px 0; }
    button { margin-right: 10px; }
  </style>
</head>
<body>
  <h1>WebAssembly Image Processor</h1>
  
  <div class="controls">
    <button id="grayscale">グレースケール</button>
    <button id="sepia">セピア</button>
    <button id="brightness">輝度調整</button>
    <input type="range" id="brightness-factor" min="0.1" max="2" step="0.1" value="1.2">
    <span id="brightness-value">1.2</span>
  </div>
  
  <canvas id="canvas"></canvas>
  
  <script type="module" src="./index.js"></script>
</body>
</html>
```

### 例2: 暗号化ライブラリ

以下は、Rustで実装した暗号化関数をWebAssemblyにコンパイルし、Webブラウザで使用する例です：

```rust
// lib.rs
use wasm_bindgen::prelude::*;
use sha2::{Sha256, Digest};
use aes_gcm::{Aes256Gcm, Key, Nonce};
use aes_gcm::aead::{Aead, NewAead};
use rand::{Rng, rngs::OsRng};
use base64::{Engine as _, engine::general_purpose};

// SHA-256ハッシュ計算
#[wasm_bindgen]
pub fn sha256_hash(data: &str) -> String {
    let mut hasher = Sha256::new();
    hasher.update(data.as_bytes());
    let result = hasher.finalize();
    
    // 16進数文字列に変換
    format!("{:x}", result)
}

// AES-GCM暗号化
#[wasm_bindgen]
pub fn encrypt(plaintext: &str, password: &str) -> Result<String, JsValue> {
    // パスワードからキーを生成
    let mut key_bytes = [0u8; 32];
    let mut hasher = Sha256::new();
    hasher.update(password.as_bytes());
    let hash = hasher.finalize();
    key_bytes.copy_from_slice(&hash);
    
    let key = Key::from_slice(&key_bytes);
    let cipher = Aes256Gcm::new(key);
    
    // ランダムなノンスを生成
    let mut nonce_bytes = [0u8; 12];
    OsRng.fill(&mut nonce_bytes);
    let nonce = Nonce::from_slice(&nonce_bytes);
    
    // 暗号化
    let ciphertext = cipher.encrypt(nonce, plaintext.as_bytes())
        .map_err(|_| JsValue::from_str("暗号化に失敗しました"))?;
    
    // ノンスと暗号文を連結してBase64エンコード
    let mut result = Vec::with_capacity(nonce_bytes.len() + ciphertext.len());
    result.extend_from_slice(&nonce_bytes);
    result.extend_from_slice(&ciphertext);
    
    Ok(general_purpose::STANDARD.encode(result))
}

// AES-GCM復号化
#[wasm_bindgen]
pub fn decrypt(ciphertext_b64: &str, password: &str) -> Result<String, JsValue> {
    // Base64デコード
    let ciphertext_with_nonce = general_purpose::STANDARD.decode(ciphertext_b64)
        .map_err(|_| JsValue::from_str("Base64デコードに失敗しました"))?;
    
    if ciphertext_with_nonce.len() < 12 {
        return Err(JsValue::from_str("無効な暗号文です"));
    }
    
    // ノンスと暗号文を分離
    let nonce_bytes = &ciphertext_with_nonce[..12];
    let ciphertext = &ciphertext_with_nonce[12..];
    
    // パスワードからキーを生成
    let mut key_bytes = [0u8; 32];
    let mut hasher = Sha256::new();
    hasher.update(password.as_bytes());
    let hash = hasher.finalize();
    key_bytes.copy_from_slice(&hash);
    
    let key = Key::from_slice(&key_bytes);
    let cipher = Aes256Gcm::new(key);
    let nonce = Nonce::from_slice(nonce_bytes);
    
    // 復号化
    let plaintext = cipher.decrypt(nonce, ciphertext)
        .map_err(|_| JsValue::from_str("復号化に失敗しました。パスワードが正しくない可能性があります。"))?;
    
    // バイト列を文字列に変換
    String::from_utf8(plaintext)
        .map_err(|_| JsValue::from_str("UTF-8デコードに失敗しました"))
}

// パスワード強度チェック
#[wasm_bindgen]
pub fn check_password_strength(password: &str) -> u8 {
    let mut score = 0;
    
    // 長さによるスコア
    if password.len() >= 8 { score += 1; }
    if password.len() >= 12 { score += 1; }
    if password.len() >= 16 { score += 1; }
    
    // 文字種によるスコア
    if password.chars().any(|c| c.is_lowercase()) { score += 1; }
    if password.chars().any(|c| c.is_uppercase()) { score += 1; }
    if password.chars().any(|c| c.is_numeric()) { score += 1; }
    if password.chars().any(|c| !c.is_alphanumeric()) { score += 1; }
    
    // 最大スコアは10
    score.min(10)
}
```

対応するJavaScriptコード：

```javascript
// index.js
import init, { sha256_hash, encrypt, decrypt, check_password_strength } from './pkg/crypto_lib.js';

async function run() {
  // Wasmモジュールを初期化
  await init();
  
  // ハッシュ計算
  document.getElementById('hash-button').addEventListener('click', () => {
    const input = document.getElementById('hash-input').value;
    const output = sha256_hash(input);
    document.getElementById('hash-output').textContent = output;
  });
  
  // 暗号化
  document.getElementById('encrypt-button').addEventListener('click', () => {
    const plaintext = document.getElementById('encrypt-input').value;
    const password = document.getElementById('encrypt-password').value;
    
    try {
      const ciphertext = encrypt(plaintext, password);
      document.getElementById('encrypt-output').textContent = ciphertext;
    } catch (error) {
      document.getElementById('encrypt-output').textContent = `エラー: ${error}`;
    }
  });
  
  // 復号化
  document.getElementById('decrypt-button').addEventListener('click', () => {
    const ciphertext = document.getElementById('decrypt-input').value;
    const password = document.getElementById('decrypt-password').value;
    
    try {
      const plaintext = decrypt(ciphertext, password);
      document.getElementById('decrypt-output').textContent = plaintext;
    } catch (error) {
      document.getElementById('decrypt-output').textContent = `エラー: ${error}`;
    }
  });
  
  // パスワード強度チェック
  document.getElementById('password-input').addEventListener('input', (e) => {
    const password = e.target.value;
    const strength = check_password_strength(password);
    const strengthElement = document.getElementById('password-strength');
    
    // 強度に応じて表示を変更
    let strengthText = '';
    let strengthColor = '';
    
    if (strength < 4) {
      strengthText = '弱い';
      strengthColor = 'red';
    } else if (strength < 7) {
      strengthText = '普通';
      strengthColor = 'orange';
    } else {
      strengthText = '強い';
      strengthColor = 'green';
    }
    
    strengthElement.textContent = strengthText;
    strengthElement.style.color = strengthColor;
  });
}

run();
```

HTML：

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Wasm Crypto Library</title>
  <style>
    body { font-family: sans-serif; max-width: 800px; margin: 0 auto; padding: 20px; }
    .section { margin-bottom: 30px; border: 1px solid #ddd; padding: 20px; border-radius: 5px; }
    h2 { margin-top: 0; }
    label { display: block; margin-bottom: 5px; }
    input[type="text"], textarea { width: 100%; padding: 8px; margin-bottom: 10px; }
    button { padding: 8px 16px; background: #4CAF50; color: white; border: none; cursor: pointer; }
    .output { background: #f5f5f5; padding: 10px; border-radius: 3px; min-height: 20px; word-break: break-all; }
  </style>
</head>
<body>
  <h1>WebAssembly 暗号化ライブラリ</h1>
  
  <div class="section">
    <h2>SHA-256 ハッシュ</h2>
    <label for="hash-input">ハッシュ化するテキスト:</label>
    <input type="text" id="hash-input" placeholder="テキストを入力">
    <button id="hash-button">ハッシュ計算</button>
    <p>結果: <span class="output" id="hash-output"></span></p>
  </div>
  
  <div class="section">
    <h2>AES-GCM 暗号化</h2>
    <label for="encrypt-input">暗号化するテキスト:</label>
    <textarea id="encrypt-input" rows="3" placeholder="テキストを入力"></textarea>
    <label for="encrypt-password">パスワード:</label>
    <input type="text" id="encrypt-password" placeholder="パスワードを入力">
    <button id="encrypt-button">暗号化</button>
    <p>暗号文: <span class="output" id="encrypt-output"></span></p>
  </div>
  
  <div class="section">
    <h2>AES-GCM 復号化</h2>
    <label for="decrypt-input">復号化する暗号文:</label>
    <textarea id="decrypt-input" rows="3" placeholder="暗号文を入力"></textarea>
    <label for="decrypt-password">パスワード:</label>
    <input type="text" id="decrypt-password" placeholder="パスワードを入力">
    <button id="decrypt-button">復号化</button>
    <p>平文: <span class="output" id="decrypt-output"></span></p>
  </div>
  
  <div class="section">
    <h2>パスワード強度チェック</h2>
    <label for="password-input">パスワード:</label>
    <input type="text" id="password-input" placeholder="パスワードを入力">
    <p>強度: <span id="password-strength"></span></p>
  </div>
  
  <script type="module" src="./index.js"></script>
</body>
</html>
```

### 例3: PHP/Laravelアプリケーションと連携するWasmモジュール

以下は、PHP/Laravelアプリケーションと連携するWasmモジュールの例です。このモジュールは、クライアント側で重い計算処理を行い、サーバー負荷を軽減します。

```rust
// lib.rs
use wasm_bindgen::prelude::*;
use serde::{Serialize, Deserialize};
use wasm_bindgen::JsCast;
use web_sys::{Request, RequestInit, RequestMode, Response};
use js_sys::{Promise, JSON};

// APIから取得するデータの型
#[derive(Serialize, Deserialize)]
struct ApiData {
    id: u32,
    values: Vec<f64>,
}

// 計算結果の型
#[derive(Serialize, Deserialize)]
struct CalculationResult {
    id: u32,
    mean: f64,
    median: f64,
    std_dev: f64,
    min: f64,
    max: f64,
}

// APIからデータを取得し、統計計算を行う関数
#[wasm_bindgen]
pub async fn fetch_and_calculate(api_url: String) -> Result<JsValue, JsValue> {
    // APIリクエストの設定
    let mut opts = RequestInit::new();
    opts.method("GET");
    opts.mode(RequestMode::Cors);
    
    // リクエストを作成
    let request = Request::new_with_str_and_init(&api_url, &opts)?;
    request.headers().set("Accept", "application/json")?;
    
    // fetchを使用してAPIを呼び出す
    let window = web_sys::window().unwrap();
    let resp_value = JsFuture::from(window.fetch_with_request(&request)).await?;
    let resp: Response = resp_value.dyn_into().unwrap();
    
    // レスポンスをJSONとして解析
    let json = JsFuture::from(resp.json()?).await?;
    let data: ApiData = serde_wasm_bindgen::from_value(json)?;
    
    // 統計計算を実行
    let result = calculate_statistics(&data);
    
    // 結果をJavaScriptの値に変換して返す
    Ok(serde_wasm_bindgen::to_value(&result)?)
}

// 統計計算を行う関数
fn calculate_statistics(data: &ApiData) -> CalculationResult {
    let values = &data.values;
    let n = values.len();
    
    if n == 0 {
        return CalculationResult {
            id: data.id,
            mean: 0.0,
            median: 0.0,
            std_dev: 0.0,
            min: 0.0,
            max: 0.0,
        };
    }
    
    // 平均値を計算
    let sum: f64 = values.iter().sum();
    let mean = sum / n as f64;
    
    // 中央値を計算
    let mut sorted_values = values.clone();
    sorted_values.sort_by(|a, b| a.partial_cmp(b).unwrap());
    let median = if n % 2 == 0 {
        (sorted_values[n/2 - 1] + sorted_values[n/2]) / 2.0
    } else {
        sorted_values[n/2]
    };
    
    // 標準偏差を計算
    let variance = values.iter()
        .map(|&x| (x - mean).powi(2))
        .sum::<f64>() / n as f64;
    let std_dev = variance.sqrt();
    
    // 最小値と最大値を計算
    let min = *values.iter().min_by(|a, b| a.partial_cmp(b).unwrap()).unwrap();
    let max = *values.iter().max_by(|a, b| a.partial_cmp(b).unwrap()).unwrap();
    
    CalculationResult {
        id: data.id,
        mean,
        median,
        std_dev,
        min,
        max,
    }
}

// 計算結果をサーバーに送信する関数
#[wasm_bindgen]
pub async fn submit_results(api_url: String, result_json: String) -> Result<(), JsValue> {
    // POSTリクエストの設定
    let mut opts = RequestInit::new();
    opts.method("POST");
    opts.mode(RequestMode::Cors);
    opts.body(Some(&JsValue::from_str(&result_json)));
    
    // リクエストを作成
    let request = Request::new_with_str_and_init(&api_url, &opts)?;
    request.headers().set("Content-Type", "application/json")?;
    request.headers().set("Accept", "application/json")?;
    
    // fetchを使用してAPIを呼び出す
    let window = web_sys::window().unwrap();
    let resp_value = JsFuture::from(window.fetch_with_request(&request)).await?;
    let resp: Response = resp_value.dyn_into().unwrap();
    
    // レスポンスをチェック
    if !resp.ok() {
        return Err(JsValue::from_str(&format!("Error: {}", resp.status())));
    }
    
    Ok(())
}
```

対応するJavaScriptコード：

```javascript
// app.js
import init, { fetch_and_calculate, submit_results } from './pkg/laravel_wasm_module.js';

async function run() {
  // Wasmモジュールを初期化
  await init();
  
  const datasetId = document.getElementById('dataset-id').value;
  const apiUrl = `/api/datasets/${datasetId}`;
  const resultUrl = `/api/results`;
  const statusElement = document.getElementById('status');
  
  try {
    // 処理開始
    statusElement.textContent = 'データを取得中...';
    
    // Wasmモジュールを使用してデータを取得し、計算を実行
    const result = await fetch_and_calculate(apiUrl);
    
    // 結果を表示
    displayResults(result);
    
    // 結果をサーバーに送信
    statusElement.textContent = '結果を送信中...';
    await submit_results(resultUrl, JSON.stringify(result));
    
    statusElement.textContent = '処理完了';
  } catch (error) {
    statusElement.textContent = `エラー: ${error}`;
    console.error(error);
  }
}

function displayResults(result) {
  document.getElementById('result-mean').textContent = result.mean.toFixed(2);
  document.getElementById('result-median').textContent = result.median.toFixed(2);
  document.getElementById('result-std-dev').textContent = result.std_dev.toFixed(2);
  document.getElementById('result-min').textContent = result.min.toFixed(2);
  document.getElementById('result-max').textContent = result.max.toFixed(2);
}

document.getElementById('calculate-button').addEventListener('click', run);
```

Laravel側のコントローラー：

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Models\Dataset;
use App\Models\CalculationResult;

class DatasetController extends Controller
{
    // データセットを取得するAPI
    public function getDataset($id)
    {
        $dataset = Dataset::findOrFail($id);
        
        return response()->json([
            'id' => $dataset->id,
            'values' => json_decode($dataset->values),
        ]);
    }
    
    // 計算結果を保存するAPI
    public function saveResult(Request $request)
    {
        $validated = $request->validate([
            'id' => 'required|integer|exists:datasets,id',
            'mean' => 'required|numeric',
            'median' => 'required|numeric',
            'std_dev' => 'required|numeric',
            'min' => 'required|numeric',
            'max' => 'required|numeric',
        ]);
        
        $result = new CalculationResult();
        $result->dataset_id = $validated['id'];
        $result->mean = $validated['mean'];
        $result->median = $validated['median'];
        $result->std_dev = $validated['std_dev'];
        $result->min = $validated['min'];
        $result->max = $validated['max'];
        $result->save();
        
        return response()->json(['status' => 'success']);
    }
}
```

Blade テンプレート：

```html
<!-- resources/views/datasets/show.blade.php -->
@extends('layouts.app')

@section('content')
<div class="container">
    <h1>データセット #{{ $dataset->id }}</h1>
    
    <div class="card mb-4">
        <div class="card-header">統計計算</div>
        <div class="card-body">
            <input type="hidden" id="dataset-id" value="{{ $dataset->id }}">
            
            <button id="calculate-button" class="btn btn-primary">
                クライアント側で計算を実行
            </button>
            
            <p id="status" class="mt-2"></p>
            
            <div class="results mt-4">
                <h4>計算結果</h4>
                <table class="table">
                    <tr>
                        <th>平均値</th>
                        <td id="result-mean">-</td>
                    </tr>
                    <tr>
                        <th>中央値</th>
                        <td id="result-median">-</td>
                    </tr>
                    <tr>
                        <th>標準偏差</th>
                        <td id="result-std-dev">-</td>
                    </tr>
                    <tr>
                        <th>最小値</th>
                        <td id="result-min">-</td>
                    </tr>
                    <tr>
                        <th>最大値</th>
                        <td id="result-max">-</td>
                    </tr>
                </table>
            </div>
        </div>
    </div>
</div>

<script type="module" src="{{ asset('js/wasm/app.js') }}"></script>
@endsection
```

## PHP/Laravel開発者向けのポイント

### PHPとWebAssemblyの比較

PHPはサーバーサイドで実行される言語であり、WebAssemblyはクライアントサイド（ブラウザ）で実行されるバイナリフォーマットです。両者の主な違いを理解することが重要です。

#### 実行環境

- **PHP**: サーバー上で実行され、HTMLを生成してクライアントに送信
- **WebAssembly**: ブラウザ上で実行され、JavaScriptと連携してDOMを操作

#### パフォーマンス特性

- **PHP**: リクエストごとに処理を実行し、結果をクライアントに返す
- **WebAssembly**: クライアント側で高速に処理を実行し、必要に応じてサーバーと通信

#### ユースケース

- **PHP**: サーバーサイドロジック、データベース操作、認証、セッション管理
- **WebAssembly**: 計算負荷の高い処理、リアルタイム処理、オフライン機能

### LaravelアプリケーションとWebAssemblyの連携

LaravelアプリケーションとWebAssemblyを連携させる方法はいくつかあります：

#### 1. フロントエンドでのWasm使用

最も一般的なアプローチは、Laravelがバックエンドを担当し、フロントエンドでWebAssemblyを使用する方法です：

```php
// routes/web.php
Route::get('/app', function () {
    return view('app');
});

// routes/api.php
Route::get('/data', [DataController::class, 'getData']);
Route::post('/results', [DataController::class, 'saveResults']);
```

```html
<!-- resources/views/app.blade.php -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Laravel + WebAssembly</title>
</head>
<body>
    <div id="app">
        <!-- アプリケーションのUI -->
    </div>
    
    <!-- WebAssemblyモジュールを読み込むスクリプト -->
    <script type="module">
        import init, { process_data } from './wasm/my_module.js';
        
        async function run() {
            await init();
            
            // APIからデータを取得
            const response = await fetch('/api/data');
            const data = await response.json();
            
            // WebAssemblyで処理
            const result = process_data(data);
            
            // 結果をサーバーに送信
            await fetch('/api/results', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').content
                },
                body: JSON.stringify(result)
            });
        }
        
        run();
    </script>
</body>
</html>
```

#### 2. Laravel Mixを使用したWasmのビルド統合

Laravel MixでWebAssemblyのビルドプロセスを統合できます：

```javascript
// webpack.mix.js
const mix = require('laravel-mix');

mix.js('resources/js/app.js', 'public/js')
   .sass('resources/sass/app.scss', 'public/css');

// WebAssemblyのビルドプロセスを追加
mix.then(() => {
    const { execSync } = require('child_process');
    console.log('Building WebAssembly module...');
    execSync('cd wasm && wasm-pack build --target web', { stdio: 'inherit' });
    execSync('cp -r wasm/pkg public/wasm', { stdio: 'inherit' });
});
```

#### 3. PHPからWebAssemblyを生成

PHP自体をWebAssemblyにコンパイルすることも理論的には可能ですが、現時点では実用的ではありません。代わりに、PHPとRust/WebAssemblyを適切に組み合わせることが重要です。

### PHPとRust/WebAssemblyの役割分担

PHP/LaravelとRust/WebAssemblyを組み合わせる際の一般的な役割分担：

#### PHPの役割

- ユーザー認証と認可
- データベース操作
- ビジネスロジック
- APIエンドポイントの提供
- テンプレートレンダリング

#### WebAssemblyの役割

- 計算負荷の高い処理
- リアルタイムデータ処理
- クライアントサイドの暗号化
- 画像/動画処理
- オフライン機能

### 移行戦略

既存のPHP/Laravelアプリケーションに段階的にWebAssemblyを導入する戦略：

1. **パフォーマンスのボトルネックを特定**: プロファイリングを行い、クライアント側で実行すべき計算負荷の高い処理を特定

2. **小さな機能から始める**: 完全な書き換えではなく、特定の機能をWebAssemblyに移行

3. **APIの設計**: PHP/LaravelバックエンドとWasmフロントエンド間の通信を適切に設計

4. **段階的な拡張**: 成功した機能を基に、徐々にWebAssemblyの使用範囲を拡大

5. **ハイブリッドアプローチの維持**: PHPとWebAssemblyの両方の長所を活かす設計を維持

### パフォーマンス最適化のポイント

PHP/LaravelとWebAssemblyを組み合わせる際のパフォーマンス最適化のポイント：

1. **データ転送の最小化**: 必要なデータのみをサーバーとクライアント間で転送

2. **バッチ処理**: 複数の小さな処理をまとめて実行

3. **キャッシュの活用**: 計算結果をクライアント側でキャッシュ

4. **非同期処理**: ユーザーインターフェースをブロックしない設計

5. **適切な役割分担**: 各技術の強みを活かした処理の分担

## 演習問題

### 演習1: 基本的なWebAssemblyモジュールの作成

**目標**: RustでWebAssemblyモジュールを作成し、Webブラウザで実行する

**要件**:
1. 基本的な数学関数（加算、減算、乗算、除算）を実装する
2. 文字列処理関数（逆順、大文字変換、小文字変換）を実装する
3. `wasm-bindgen`を使用してJavaScriptとのインターフェースを作成する
4. HTMLページでモジュールを読み込み、関数を呼び出す

**ヒント**:
- `wasm-pack`を使用してプロジェクトを初期化する
- `wasm-bindgen`属性を使用して関数をエクスポートする
- 文字列の受け渡しに注意する
- 簡単なHTMLフォームを作成して関数をテストする

**スケルトンコード**:

```rust
// lib.rs
use wasm_bindgen::prelude::*;

// 数学関数
#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    // TODO: 実装する
}

#[wasm_bindgen]
pub fn subtract(a: i32, b: i32) -> i32 {
    // TODO: 実装する
}

#[wasm_bindgen]
pub fn multiply(a: i32, b: i32) -> i32 {
    // TODO: 実装する
}

#[wasm_bindgen]
pub fn divide(a: i32, b: i32) -> Result<i32, JsValue> {
    // TODO: 実装する（ゼロ除算に注意）
}

// 文字列処理関数
#[wasm_bindgen]
pub fn reverse(s: &str) -> String {
    // TODO: 実装する
}

#[wasm_bindgen]
pub fn to_uppercase(s: &str) -> String {
    // TODO: 実装する
}

#[wasm_bindgen]
pub fn to_lowercase(s: &str) -> String {
    // TODO: 実装する
}
```

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>WebAssembly Basic Demo</title>
</head>
<body>
    <h1>WebAssembly Basic Demo</h1>
    
    <div>
        <h2>数学関数</h2>
        <input type="number" id="num1" value="5">
        <select id="operation">
            <option value="add">+</option>
            <option value="subtract">-</option>
            <option value="multiply">×</option>
            <option value="divide">÷</option>
        </select>
        <input type="number" id="num2" value="3">
        <button id="calculate">=</button>
        <span id="result"></span>
    </div>
    
    <div>
        <h2>文字列処理</h2>
        <input type="text" id="text-input" value="Hello, WebAssembly!">
        <button id="reverse-button">逆順</button>
        <button id="uppercase-button">大文字</button>
        <button id="lowercase-button">小文字</button>
        <div id="text-result"></div>
    </div>
    
    <script type="module">
        // TODO: WebAssemblyモジュールを読み込み、イベントリスナーを設定する
    </script>
</body>
</html>
```

### 演習2: 画像フィルターアプリケーション

**目標**: WebAssemblyを使用して画像フィルターを実装する

**要件**:
1. 画像をキャンバスに読み込む機能を実装する
2. グレースケールフィルターを実装する
3. セピアフィルターを実装する
4. ぼかしフィルターを実装する
5. フィルターを適用した画像を保存する機能を実装する

**ヒント**:
- `web-sys`クレートを使用してキャンバスとコンテキストを操作する
- ピクセルデータを取得し、WebAssemblyで処理する
- 処理後のピクセルデータをキャンバスに戻す
- パフォーマンスを測定し、JavaScriptとの比較を行う

**スケルトンコード**:

```rust
// lib.rs
use wasm_bindgen::prelude::*;
use web_sys::{ImageData, CanvasRenderingContext2d};

#[wasm_bindgen]
pub fn grayscale(ctx: &CanvasRenderingContext2d, width: u32, height: u32) -> Result<(), JsValue> {
    // TODO: グレースケールフィルターを実装する
}

#[wasm_bindgen]
pub fn sepia(ctx: &CanvasRenderingContext2d, width: u32, height: u32) -> Result<(), JsValue> {
    // TODO: セピアフィルターを実装する
}

#[wasm_bindgen]
pub fn blur(ctx: &CanvasRenderingContext2d, width: u32, height: u32, radius: u32) -> Result<(), JsValue> {
    // TODO: ぼかしフィルターを実装する
}
```

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>WebAssembly Image Filters</title>
    <style>
        .controls { margin: 20px 0; }
        button { margin-right: 10px; }
        canvas { border: 1px solid #ddd; }
    </style>
</head>
<body>
    <h1>WebAssembly Image Filters</h1>
    
    <div class="controls">
        <input type="file" id="image-input" accept="image/*">
    </div>
    
    <div class="controls">
        <button id="grayscale-button">グレースケール</button>
        <button id="sepia-button">セピア</button>
        <button id="blur-button">ぼかし</button>
        <input type="range" id="blur-radius" min="1" max="20" value="3">
        <span id="blur-value">3</span>
    </div>
    
    <div class="controls">
        <button id="reset-button">リセット</button>
        <button id="save-button">保存</button>
    </div>
    
    <canvas id="canvas"></canvas>
    
    <script type="module">
        // TODO: WebAssemblyモジュールを読み込み、イベントリスナーを設定する
    </script>
</body>
</html>
```

### 演習3: PHP/LaravelアプリケーションとWebAssemblyの連携

**目標**: PHP/LaravelアプリケーションとWebAssemblyを連携させる

**要件**:
1. Laravelアプリケーションで大量のデータを提供するAPIエンドポイントを作成する
2. WebAssemblyモジュールでデータを処理する関数を実装する
3. フロントエンドでAPIからデータを取得し、WebAssemblyで処理する
4. 処理結果をLaravelアプリケーションに送信して保存する
5. 処理時間を計測し、サーバーサイド処理との比較を行う

**ヒント**:
- Laravelで簡単なAPIエンドポイントを作成する
- WebAssemblyモジュールで`fetch`を使用してAPIを呼び出す
- 処理結果をLaravelに送信するためのエンドポイントを作成する
- パフォーマンスを測定するためのコードを追加する

**スケルトンコード**:

```rust
// lib.rs
use wasm_bindgen::prelude::*;
use serde::{Serialize, Deserialize};
use wasm_bindgen::JsCast;
use web_sys::{Request, RequestInit, RequestMode, Response};
use js_sys::{Promise, JSON, Date};

#[derive(Serialize, Deserialize)]
struct ApiData {
    // TODO: APIから取得するデータの型を定義する
}

#[derive(Serialize, Deserialize)]
struct ProcessedResult {
    // TODO: 処理結果の型を定義する
}

#[wasm_bindgen]
pub async fn process_data(api_url: String) -> Result<JsValue, JsValue> {
    // TODO: APIからデータを取得し、処理を行う関数を実装する
}

#[wasm_bindgen]
pub async fn submit_result(api_url: String, result_json: String) -> Result<(), JsValue> {
    // TODO: 処理結果をAPIに送信する関数を実装する
}
```

```php
// Laravel側のコントローラー
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;

class DataController extends Controller
{
    public function getData()
    {
        // TODO: 大量のデータを生成して返すメソッドを実装する
    }
    
    public function saveResult(Request $request)
    {
        // TODO: 処理結果を保存するメソッドを実装する
    }
}
```

```html
<!-- Blade テンプレート -->
@extends('layouts.app')

@section('content')
<div class="container">
    <h1>WebAssembly Data Processing</h1>
    
    <div class="card">
        <div class="card-header">データ処理</div>
        <div class="card-body">
            <button id="process-button" class="btn btn-primary">処理開始</button>
            
            <div class="mt-3">
                <h5>処理時間</h5>
                <p>WebAssembly: <span id="wasm-time">-</span> ms</p>
                <p>サーバーサイド: <span id="server-time">-</span> ms</p>
            </div>
            
            <div class="mt-3">
                <h5>処理結果</h5>
                <pre id="result"></pre>
            </div>
        </div>
    </div>
</div>

<script type="module">
    // TODO: WebAssemblyモジュールを読み込み、イベントリスナーを設定する
</script>
@endsection
```

これらの演習問題を通じて、WebAssemblyの基本から応用まで、実践的なスキルを身につけることができます。特に、PHP/Laravel開発者としての既存の知識を活かしながら、WebAssemblyの強みを理解し、適切に組み合わせる方法を学ぶことが重要です。

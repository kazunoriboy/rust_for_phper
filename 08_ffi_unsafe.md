# トピック4: FFI (Foreign Function Interface) と `unsafe` Rust

## 学習目標

このセクションを学ぶことで、以下のことができるようになります：

- Rustから他の言語（主にC言語）のコードを呼び出す方法を理解する
- 他の言語からRustのコードを呼び出せるようにする
- `unsafe`ブロックの適切な使用方法と安全な抽象化の作成方法を習得する
- FFIを使用する際の一般的な落とし穴と、それを回避する方法を理解する
- メモリ安全性を維持しながら低レベル操作を行う方法を学ぶ

## 主要な概念の説明

### FFI (Foreign Function Interface) とは

FFI（Foreign Function Interface）は、あるプログラミング言語で書かれたコードから別の言語で書かれた関数を呼び出すための仕組みです。Rustでは、主にC言語のコードとのインターフェースを提供しています。

FFIを使用する主な理由：

1. 既存のCライブラリを再利用する
2. オペレーティングシステムのAPIにアクセスする
3. パフォーマンスクリティカルなコードをCで実装する
4. RustのコードをCから呼び出せるようにする

### `unsafe` Rustの基本

Rustの安全性保証は、コンパイラによる静的解析に基づいています。しかし、一部の操作（特に低レベル操作やFFI）は、コンパイラが安全性を保証できません。このような操作を行うために、Rustは`unsafe`ブロックを提供しています。

`unsafe`ブロック内では、以下の「安全でない」操作が許可されます：

1. 生ポインタの参照外し
2. 可変静的変数の変更
3. 外部関数の呼び出し
4. `unsafe`トレイトの実装
5. ユニオンフィールドへのアクセス

```rust
fn main() {
    let mut num = 5;
    
    // 生ポインタの作成（これ自体は安全）
    let ptr = &mut num as *mut i32;
    
    // 生ポインタの参照外しは安全でない
    unsafe {
        *ptr = 10;
    }
    
    println!("num: {}", num); // 10
}
```

### CからRustの関数を呼び出す

Rustの関数をCから呼び出せるようにするには、以下の手順が必要です：

1. 関数に`#[no_mangle]`属性を付ける（名前修飾を無効化）
2. 関数を`extern "C"`として宣言（Cの呼び出し規約を使用）
3. Cと互換性のある型のみを使用

```rust
// Cから呼び出し可能なRust関数
#[no_mangle]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

この関数は、以下のようにCから呼び出すことができます：

```c
// C言語側のコード
#include <stdint.h>
#include <stdio.h>

// Rust関数の宣言
extern int32_t add(int32_t a, int32_t b);

int main() {
    int32_t result = add(5, 7);
    printf("Result: %d\n", result); // "Result: 12"
    return 0;
}
```

### RustからCの関数を呼び出す

Cの関数をRustから呼び出すには、以下の手順が必要です：

1. `extern "C"`ブロックでCの関数を宣言
2. 関数呼び出しを`unsafe`ブロックで囲む

```rust
// Cの関数を宣言
extern "C" {
    fn printf(format: *const i8, ...) -> i32;
}

fn main() {
    unsafe {
        // Cの関数を呼び出す
        printf(b"Hello, %s!\0".as_ptr() as *const i8, b"world\0".as_ptr() as *const i8);
    }
}
```

### `libc`クレートの使用

`libc`クレートは、標準Cライブラリの関数やデータ型へのバインディングを提供します：

```rust
use libc::{c_char, c_int, printf, size_t};
use std::ffi::CString;

fn main() {
    // CStringを使用して、NULL終端の文字列を作成
    let hello = CString::new("Hello, world!").unwrap();
    
    unsafe {
        // printfを呼び出す
        printf(b"%s\n\0".as_ptr() as *const c_char, hello.as_ptr());
    }
}
```

### データ型の変換

RustとC間でデータを受け渡す際には、型の変換が必要です：

#### 基本型

| Rust型 | C型 |
|--------|-----|
| i8     | char / int8_t |
| u8     | unsigned char / uint8_t |
| i16    | short / int16_t |
| u16    | unsigned short / uint16_t |
| i32    | int / int32_t |
| u32    | unsigned int / uint32_t |
| i64    | long long / int64_t |
| u64    | unsigned long long / uint64_t |
| f32    | float |
| f64    | double |
| bool   | _Bool (C99) |

#### 文字列

Rustの文字列（`String`や`&str`）は、C言語のNULL終端文字列と互換性がありません。変換には`std::ffi`モジュールの`CString`と`CStr`を使用します：

```rust
use std::ffi::{CString, CStr};
use std::os::raw::c_char;

// RustからCへの文字列変換
fn rust_to_c(s: &str) -> CString {
    CString::new(s).unwrap()
}

// CからRustへの文字列変換
unsafe fn c_to_rust(s: *const c_char) -> String {
    CStr::from_ptr(s).to_string_lossy().into_owned()
}

// Cの関数を宣言
extern "C" {
    fn strlen(s: *const c_char) -> usize;
}

fn main() {
    let rust_str = "Hello, world!";
    let c_str = rust_to_c(rust_str);
    
    unsafe {
        // Cの関数を呼び出す
        let len = strlen(c_str.as_ptr());
        println!("Length: {}", len); // "Length: 13"
        
        // CからRustへの変換
        let back_to_rust = c_to_rust(c_str.as_ptr());
        println!("Back to Rust: {}", back_to_rust); // "Back to Rust: Hello, world!"
    }
}
```

#### 構造体

Rustの構造体をCと共有するには、`#[repr(C)]`属性を使用してメモリレイアウトをC互換にする必要があります：

```rust
// C互換の構造体
#[repr(C)]
pub struct Point {
    x: f64,
    y: f64,
}

// Cから呼び出し可能な関数
#[no_mangle]
pub extern "C" fn distance(p1: Point, p2: Point) -> f64 {
    let dx = p2.x - p1.x;
    let dy = p2.y - p1.y;
    (dx * dx + dy * dy).sqrt()
}
```

C言語側：

```c
// C言語側のコード
typedef struct {
    double x;
    double y;
} Point;

// Rust関数の宣言
extern double distance(Point p1, Point p2);

int main() {
    Point p1 = {1.0, 2.0};
    Point p2 = {4.0, 6.0};
    
    double d = distance(p1, p2);
    printf("Distance: %f\n", d); // "Distance: 5.000000"
    
    return 0;
}
```

### コールバック関数

Cの関数にRustの関数をコールバックとして渡すこともできます：

```rust
use libc::{c_int, c_void};
use std::mem;

// Cのqsort関数の宣言
extern "C" {
    fn qsort(
        base: *mut c_void,
        num: usize,
        size: usize,
        compar: extern "C" fn(*const c_void, *const c_void) -> c_int,
    );
}

// コールバック関数
extern "C" fn compare(a: *const c_void, b: *const c_void) -> c_int {
    unsafe {
        let a = *(a as *const i32);
        let b = *(b as *const i32);
        
        if a < b { -1 } else if a > b { 1 } else { 0 }
    }
}

fn main() {
    let mut numbers = [5, 3, 1, 4, 2];
    
    unsafe {
        qsort(
            numbers.as_mut_ptr() as *mut c_void,
            numbers.len(),
            mem::size_of::<i32>(),
            compare,
        );
    }
    
    println!("Sorted: {:?}", numbers); // "Sorted: [1, 2, 3, 4, 5]"
}
```

### 安全な抽象化の作成

`unsafe`コードを使用する場合、それを安全なインターフェースで包むことが重要です：

```rust
// 安全でない操作を行う関数
unsafe fn dangerous_operation(ptr: *mut i32) {
    *ptr = 42;
}

// 安全な抽象化
fn safe_wrapper(x: &mut i32) {
    unsafe {
        dangerous_operation(x as *mut i32);
    }
}

fn main() {
    let mut num = 0;
    
    // 安全なインターフェースを使用
    safe_wrapper(&mut num);
    
    println!("num: {}", num); // "num: 42"
}
```

安全な抽象化を作成する際の原則：

1. `unsafe`コードを最小限に保つ
2. 安全な前提条件を明確にドキュメント化する
3. 可能な限り多くのチェックをコンパイル時または実行時に行う
4. 不変条件を維持する

### FFIの落とし穴と回避策

#### メモリ管理

Rustとは異なり、C言語には自動メモリ管理がありません。CのAPIを使用する際には、メモリリークや二重解放を避けるために、適切なメモリ管理が必要です：

```rust
use libc::{c_char, c_void, free, malloc, strcpy};
use std::ffi::CString;
use std::mem;

extern "C" {
    fn strdup(s: *const c_char) -> *mut c_char;
}

fn main() {
    let rust_str = "Hello, FFI!";
    let c_str = CString::new(rust_str).unwrap();
    
    unsafe {
        // Cのメモリ割り当て関数を使用
        let ptr = strdup(c_str.as_ptr());
        if ptr.is_null() {
            panic!("Memory allocation failed");
        }
        
        // 文字列を使用
        println!("C string: {:?}", CStr::from_ptr(ptr));
        
        // メモリを解放
        free(ptr as *mut c_void);
        
        // 解放後のポインタにアクセスしない！
        // println!("After free: {:?}", CStr::from_ptr(ptr)); // 危険！
    }
}
```

#### エラー処理

C言語のAPIは、エラーを返すために様々な方法（戻り値、グローバル変数など）を使用します。Rustでは、これらのエラーを適切に処理し、`Result`型に変換することが重要です：

```rust
use libc::{c_int, c_void, malloc, size_t};
use std::io::{Error, ErrorKind, Result};

fn allocate(size: usize) -> Result<*mut u8> {
    unsafe {
        let ptr = malloc(size as size_t) as *mut u8;
        
        if ptr.is_null() {
            // mallocが失敗した場合
            Err(Error::new(ErrorKind::OutOfMemory, "malloc failed"))
        } else {
            Ok(ptr)
        }
    }
}

fn main() -> Result<()> {
    let size = 1024;
    
    // 安全な抽象化を使用
    let ptr = allocate(size)?;
    
    unsafe {
        // メモリを使用...
        
        // メモリを解放
        libc::free(ptr as *mut c_void);
    }
    
    Ok(())
}
```

#### スレッド安全性

C言語のライブラリは、必ずしもスレッド安全ではありません。Rustからマルチスレッドでこれらのライブラリを使用する場合は、適切な同期が必要です：

```rust
use std::sync::Mutex;
use std::thread;

// スレッド安全でないCライブラリの関数
extern "C" {
    fn unsafe_c_function();
}

// グローバルミューテックスを使用して同期
lazy_static::lazy_static! {
    static ref C_API_MUTEX: Mutex<()> = Mutex::new(());
}

// スレッド安全なラッパー
fn safe_c_function() {
    let _lock = C_API_MUTEX.lock().unwrap();
    unsafe {
        unsafe_c_function();
    }
}

fn main() {
    let mut handles = vec![];
    
    for _ in 0..10 {
        let handle = thread::spawn(|| {
            safe_c_function();
        });
        handles.push(handle);
    }
    
    for handle in handles {
        handle.join().unwrap();
    }
}
```

### `bindgen`を使用した自動バインディング生成

手動でFFIバインディングを書くのは面倒で、エラーが発生しやすいです。`bindgen`クレートを使用すると、CのヘッダーファイルからRustのバインディングを自動生成できます：

```rust
// build.rs
extern crate bindgen;

use std::env;
use std::path::PathBuf;

fn main() {
    // libclangへのパスを指定
    println!("cargo:rustc-link-lib=clang");
    
    // バインディングを生成
    let bindings = bindgen::Builder::default()
        .header("wrapper.h")
        .generate()
        .expect("Unable to generate bindings");
    
    // 出力パスを取得
    let out_path = PathBuf::from(env::var("OUT_DIR").unwrap());
    
    // バインディングを書き込む
    bindings
        .write_to_file(out_path.join("bindings.rs"))
        .expect("Couldn't write bindings!");
}
```

```rust
// src/main.rs
// 生成されたバインディングをインクルード
#![allow(non_upper_case_globals)]
#![allow(non_camel_case_types)]
#![allow(non_snake_case)]

include!(concat!(env!("OUT_DIR"), "/bindings.rs"));

fn main() {
    unsafe {
        // 生成されたバインディングを使用
        println!("Version: {}", clang_getClangVersion());
    }
}
```

### `cc`クレートを使用したCコードのビルド

Rustプロジェクト内でCコードをビルドするには、`cc`クレートを使用できます：

```rust
// build.rs
extern crate cc;

fn main() {
    cc::Build::new()
        .file("src/native/example.c")
        .compile("example");
}
```

```c
// src/native/example.c
#include <stdio.h>

void print_hello() {
    printf("Hello from C!\n");
}
```

```rust
// src/main.rs
extern "C" {
    fn print_hello();
}

fn main() {
    unsafe {
        print_hello();
    }
}
```

## 具体的なコード例

### 例1: SQLiteデータベースへのアクセス

SQLiteデータベースにアクセスするRustコードの例を示します。`rusqlite`クレートを使用しますが、内部的にはSQLiteのCライブラリを呼び出しています：

```rust
use rusqlite::{params, Connection, Result};

// ユーザーデータを表す構造体
#[derive(Debug)]
struct User {
    id: i32,
    name: String,
    email: String,
    age: Option<i32>,
}

fn main() -> Result<()> {
    // インメモリデータベースに接続
    let conn = Connection::open_in_memory()?;
    
    // テーブルを作成
    conn.execute(
        "CREATE TABLE users (
            id    INTEGER PRIMARY KEY,
            name  TEXT NOT NULL,
            email TEXT NOT NULL UNIQUE,
            age   INTEGER
        )",
        [],
    )?;
    
    // データを挿入
    let users = vec![
        (1, "Alice", "alice@example.com", Some(30)),
        (2, "Bob", "bob@example.com", Some(25)),
        (3, "Charlie", "charlie@example.com", None),
    ];
    
    for user in users {
        conn.execute(
            "INSERT INTO users (id, name, email, age) VALUES (?1, ?2, ?3, ?4)",
            params![user.0, user.1, user.2, user.3],
        )?;
    }
    
    // データを取得
    let mut stmt = conn.prepare("SELECT id, name, email, age FROM users")?;
    let user_iter = stmt.query_map([], |row| {
        Ok(User {
            id: row.get(0)?,
            name: row.get(1)?,
            email: row.get(2)?,
            age: row.get(3)?,
        })
    })?;
    
    for user in user_iter {
        println!("User: {:?}", user?);
    }
    
    // トランザクションを使用
    let tx = conn.transaction()?;
    
    tx.execute(
        "UPDATE users SET age = ?1 WHERE id = ?2",
        params![35, 1],
    )?;
    
    tx.commit()?;
    
    // 更新されたデータを確認
    let mut stmt = conn.prepare("SELECT id, name, email, age FROM users WHERE id = ?1")?;
    let user = stmt.query_row(params![1], |row| {
        Ok(User {
            id: row.get(0)?,
            name: row.get(1)?,
            email: row.get(2)?,
            age: row.get(3)?,
        })
    })?;
    
    println!("Updated user: {:?}", user);
    
    Ok(())
}
```

このコードは、`rusqlite`クレートを使用してSQLiteデータベースにアクセスしています。内部的には、FFIを通じてSQLiteのCライブラリを呼び出しています。

### 例2: OpenSSLを使用した暗号化

OpenSSLライブラリを使用して、データを暗号化・復号化する例を示します：

```rust
use openssl::symm::{Cipher, Crypter, Mode};
use rand::Rng;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 暗号化するデータ
    let data = b"Secret message";
    
    // 暗号化アルゴリズムを選択
    let cipher = Cipher::aes_256_cbc();
    
    // ランダムなキーと初期化ベクトル（IV）を生成
    let mut key = [0; 32]; // AES-256は32バイトのキーを使用
    let mut iv = [0; 16];  // AES-CBCは16バイトのIVを使用
    
    rand::thread_rng().fill(&mut key);
    rand::thread_rng().fill(&mut iv);
    
    // 暗号化
    let encrypted = encrypt(cipher, &key, Some(&iv), data)?;
    println!("Encrypted: {:?}", encrypted);
    
    // 復号化
    let decrypted = decrypt(cipher, &key, Some(&iv), &encrypted)?;
    println!("Decrypted: {:?}", String::from_utf8_lossy(&decrypted));
    
    Ok(())
}

// 暗号化関数
fn encrypt(
    cipher: Cipher,
    key: &[u8],
    iv: Option<&[u8]>,
    data: &[u8],
) -> Result<Vec<u8>, openssl::error::ErrorStack> {
    let mut crypter = Crypter::new(cipher, Mode::Encrypt, key, iv)?;
    let mut output = vec![0; data.len() + cipher.block_size()];
    
    let count = crypter.update(data, &mut output)?;
    let final_count = crypter.finalize(&mut output[count..])?;
    
    output.truncate(count + final_count);
    Ok(output)
}

// 復号化関数
fn decrypt(
    cipher: Cipher,
    key: &[u8],
    iv: Option<&[u8]>,
    data: &[u8],
) -> Result<Vec<u8>, openssl::error::ErrorStack> {
    let mut crypter = Crypter::new(cipher, Mode::Decrypt, key, iv)?;
    let mut output = vec![0; data.len() + cipher.block_size()];
    
    let count = crypter.update(data, &mut output)?;
    let final_count = crypter.finalize(&mut output[count..])?;
    
    output.truncate(count + final_count);
    Ok(output)
}
```

このコードは、`openssl`クレートを使用してデータを暗号化・復号化しています。内部的には、FFIを通じてOpenSSLのCライブラリを呼び出しています。

### 例3: 独自のFFIバインディングの作成

独自のCライブラリに対するFFIバインディングを作成する例を示します：

```c
// example.h
#ifndef EXAMPLE_H
#define EXAMPLE_H

typedef struct {
    double x;
    double y;
} Point;

// 2点間の距離を計算
double distance(Point p1, Point p2);

// 文字列を逆順にする
char* reverse_string(const char* str);

// 文字列を解放
void free_string(char* str);

#endif
```

```c
// example.c
#include "example.h"
#include <math.h>
#include <stdlib.h>
#include <string.h>

double distance(Point p1, Point p2) {
    double dx = p2.x - p1.x;
    double dy = p2.y - p1.y;
    return sqrt(dx * dx + dy * dy);
}

char* reverse_string(const char* str) {
    size_t len = strlen(str);
    char* result = (char*)malloc(len + 1);
    
    if (result == NULL) {
        return NULL;
    }
    
    for (size_t i = 0; i < len; i++) {
        result[i] = str[len - i - 1];
    }
    
    result[len] = '\0';
    return result;
}

void free_string(char* str) {
    free(str);
}
```

```rust
// build.rs
extern crate cc;

fn main() {
    // Cコードをビルド
    cc::Build::new()
        .file("src/native/example.c")
        .compile("example");
}
```

```rust
// src/lib.rs
use std::ffi::{CStr, CString};
use std::os::raw::{c_char, c_double};

#[repr(C)]
pub struct Point {
    pub x: c_double,
    pub y: c_double,
}

extern "C" {
    fn distance(p1: Point, p2: Point) -> c_double;
    fn reverse_string(s: *const c_char) -> *mut c_char;
    fn free_string(s: *mut c_char);
}

// 安全なラッパー関数
pub fn calculate_distance(p1: Point, p2: Point) -> f64 {
    unsafe { distance(p1, p2) }
}

pub fn reverse(s: &str) -> Result<String, std::str::Utf8Error> {
    let c_str = CString::new(s).unwrap();
    unsafe {
        let ptr = reverse_string(c_str.as_ptr());
        if ptr.is_null() {
            return Err(std::str::Utf8Error::new(b"", 0));
        }
        
        let result = CStr::from_ptr(ptr).to_str()?.to_owned();
        free_string(ptr);
        Ok(result)
    }
}
```

```rust
// src/main.rs
use example::{calculate_distance, reverse, Point};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 距離を計算
    let p1 = Point { x: 1.0, y: 2.0 };
    let p2 = Point { x: 4.0, y: 6.0 };
    
    let d = calculate_distance(p1, p2);
    println!("Distance: {}", d);
    
    // 文字列を逆順にする
    let s = "Hello, FFI!";
    let reversed = reverse(s)?;
    println!("Original: {}", s);
    println!("Reversed: {}", reversed);
    
    Ok(())
}
```

このコードは、独自のCライブラリに対するFFIバインディングを作成し、安全なRustインターフェースを提供しています。

## PHP/Laravel開発者向けのポイント

### PHPの拡張モジュールとの比較

PHPでは、C言語で書かれた拡張モジュールを使用して、パフォーマンスを向上させたり、低レベルの機能にアクセスしたりします：

```php
<?php
// PHPのOpenSSL拡張モジュールを使用した暗号化
$data = "Secret message";
$method = "AES-256-CBC";

// キーと初期化ベクトル（IV）を生成
$key = openssl_random_pseudo_bytes(32);
$iv = openssl_random_pseudo_bytes(16);

// 暗号化
$encrypted = openssl_encrypt($data, $method, $key, 0, $iv);
echo "Encrypted: " . $encrypted . "\n";

// 復号化
$decrypted = openssl_decrypt($encrypted, $method, $key, 0, $iv);
echo "Decrypted: " . $decrypted . "\n";
```

Rustでは、FFIを使用して同様の機能を実現できますが、より型安全で、メモリ安全性が保証されています：

```rust
use openssl::symm::{encrypt, decrypt, Cipher};
use rand::Rng;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let data = b"Secret message";
    let cipher = Cipher::aes_256_cbc();
    
    // キーと初期化ベクトル（IV）を生成
    let mut key = [0; 32];
    let mut iv = [0; 16];
    
    rand::thread_rng().fill(&mut key);
    rand::thread_rng().fill(&mut iv);
    
    // 暗号化
    let encrypted = encrypt(cipher, &key, Some(&iv), data)?;
    println!("Encrypted: {:?}", base64::encode(&encrypted));
    
    // 復号化
    let decrypted = decrypt(cipher, &key, Some(&iv), &encrypted)?;
    println!("Decrypted: {}", String::from_utf8_lossy(&decrypted));
    
    Ok(())
}
```

### PHPのFFI拡張機能との比較

PHP 7.4以降では、FFI拡張機能を使用してCライブラリを直接呼び出すことができます：

```php
<?php
// PHPのFFI拡張機能を使用したCライブラリの呼び出し
$ffi = FFI::cdef("
    typedef struct {
        double x;
        double y;
    } Point;
    
    double distance(Point p1, Point p2);
", "libexample.so");

$p1 = $ffi->new("Point");
$p1->x = 1.0;
$p1->y = 2.0;

$p2 = $ffi->new("Point");
$p2->x = 4.0;
$p2->y = 6.0;

$d = $ffi->distance($p1, $p2);
echo "Distance: " . $d . "\n";
```

Rustでは、FFIはより統合されており、型安全性が高いです：

```rust
use std::os::raw::c_double;

#[repr(C)]
struct Point {
    x: c_double,
    y: c_double,
}

extern "C" {
    fn distance(p1: Point, p2: Point) -> c_double;
}

fn main() {
    let p1 = Point { x: 1.0, y: 2.0 };
    let p2 = Point { x: 4.0, y: 6.0 };
    
    let d = unsafe { distance(p1, p2) };
    println!("Distance: {}", d);
}
```

### Laravelパッケージとの対比

Laravelでは、パッケージを使用して機能を拡張します。一部のパッケージは、パフォーマンスのためにC言語の拡張モジュールに依存しています：

```php
<?php
// Laravelでのイメージ処理（Intervention Image）
use Intervention\Image\Facades\Image;

$img = Image::make('public/foo.jpg');
$img->resize(300, 200);
$img->save('public/bar.jpg');
```

Rustでは、FFIを使用して同様の機能を実現できますが、より型安全で、メモリ安全性が保証されています：

```rust
use image::{imageops, GenericImageView};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 画像を読み込み
    let mut img = image::open("public/foo.jpg")?;
    
    // リサイズ
    let resized = imageops::resize(
        &img,
        300,
        200,
        image::imageops::FilterType::Lanczos3,
    );
    
    // 保存
    resized.save("public/bar.jpg")?;
    
    Ok(())
}
```

### FFIの移行ポイント

1. **安全性の考慮**:
   - PHPでは、FFIを使用する際の安全性チェックは開発者の責任
   - Rustでは、`unsafe`ブロックを最小限に保ち、安全な抽象化を作成することが重要

2. **メモリ管理**:
   - PHPでは、ガベージコレクションがメモリを管理
   - Rustでは、所有権システムがメモリを管理するが、FFIを使用する際は手動でメモリを管理する必要がある場合がある

3. **エラー処理**:
   - PHPでは、例外を使用してエラーを処理
   - Rustでは、`Result`型を使用してエラーを処理し、FFIからのエラーを適切に変換する必要がある

4. **パフォーマンス**:
   - PHPのFFIは、ネイティブPHP関数よりも遅い場合がある
   - Rustでは、FFIのオーバーヘッドは最小限で、高いパフォーマンスを実現できる

5. **クロスプラットフォーム対応**:
   - PHPのFFIは、プラットフォームによって動作が異なる場合がある
   - Rustでは、条件付きコンパイルを使用して、異なるプラットフォームに対応できる

## 演習問題

### 演習1: 簡単なCライブラリのラッパー作成

**目標**: 簡単な数学関数を提供するCライブラリのRustラッパーを作成する

**要件**:
1. 以下の関数を持つCライブラリを作成する:
   - `add(a, b)`: 2つの数値を加算
   - `subtract(a, b)`: 2つの数値を減算
   - `multiply(a, b)`: 2つの数値を乗算
   - `divide(a, b)`: 2つの数値を除算（ゼロ除算チェック付き）

2. Rustから上記の関数を呼び出すFFIバインディングを作成する
3. 安全なRustインターフェースを提供する
4. エラー処理を適切に行う

**ヒント**:
- `cc`クレートを使用して、Cコードをビルドする
- `Result`型を使用して、エラーを適切に処理する
- ゼロ除算などのエラーケースを考慮する

**スケルトンコード**:

```c
// math.h
#ifndef MATH_H
#define MATH_H

// 加算
double add(double a, double b);

// 減算
double subtract(double a, double b);

// 乗算
double multiply(double a, double b);

// 除算（ゼロ除算チェック付き）
// 成功時は1、失敗時は0を返す
int divide(double a, double b, double* result);

#endif
```

```c
// math.c
#include "math.h"

double add(double a, double b) {
    return a + b;
}

double subtract(double a, double b) {
    return a - b;
}

double multiply(double a, double b) {
    return a * b;
}

int divide(double a, double b, double* result) {
    if (b == 0.0) {
        return 0; // 失敗
    }
    
    *result = a / b;
    return 1; // 成功
}
```

```rust
// build.rs
extern crate cc;

fn main() {
    // TODO: Cコードをビルドする
}
```

```rust
// src/lib.rs
use std::os::raw::{c_double, c_int};

// TODO: FFIバインディングを定義する

// TODO: 安全なRustインターフェースを提供する

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_add() {
        assert_eq!(add(2.0, 3.0), 5.0);
    }
    
    #[test]
    fn test_subtract() {
        assert_eq!(subtract(5.0, 3.0), 2.0);
    }
    
    #[test]
    fn test_multiply() {
        assert_eq!(multiply(2.0, 3.0), 6.0);
    }
    
    #[test]
    fn test_divide() {
        assert_eq!(divide(6.0, 3.0).unwrap(), 2.0);
        assert!(divide(1.0, 0.0).is_err());
    }
}
```

```rust
// src/main.rs
use math_ffi::{add, subtract, multiply, divide};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("2 + 3 = {}", add(2.0, 3.0));
    println!("5 - 3 = {}", subtract(5.0, 3.0));
    println!("2 * 3 = {}", multiply(2.0, 3.0));
    println!("6 / 3 = {}", divide(6.0, 3.0)?);
    
    match divide(1.0, 0.0) {
        Ok(result) => println!("1 / 0 = {}", result), // ここには到達しないはず
        Err(e) => println!("Error: {}", e),
    }
    
    Ok(())
}
```

### 演習2: 文字列処理ライブラリの作成

**目標**: 文字列処理関数を提供するCライブラリとそのRustラッパーを作成する

**要件**:
1. 以下の関数を持つCライブラリを作成する:
   - `to_upper(str)`: 文字列を大文字に変換
   - `to_lower(str)`: 文字列を小文字に変換
   - `reverse(str)`: 文字列を逆順にする
   - `count_chars(str)`: 文字列の長さを返す

2. Rustから上記の関数を呼び出すFFIバインディングを作成する
3. 安全なRustインターフェースを提供する
4. メモリ管理を適切に行う

**ヒント**:
- `CString`と`CStr`を使用して、Rustと文字列を変換する
- メモリリークを避けるために、Cで割り当てられたメモリを適切に解放する
- UTF-8エンコーディングの問題を考慮する

**スケルトンコード**:

```c
// string_utils.h
#ifndef STRING_UTILS_H
#define STRING_UTILS_H

// 文字列を大文字に変換
char* to_upper(const char* str);

// 文字列を小文字に変換
char* to_lower(const char* str);

// 文字列を逆順にする
char* reverse(const char* str);

// 文字列の長さを返す
int count_chars(const char* str);

// 文字列を解放
void free_string(char* str);

#endif
```

```c
// string_utils.c
#include "string_utils.h"
#include <ctype.h>
#include <stdlib.h>
#include <string.h>

char* to_upper(const char* str) {
    // TODO: 実装する
}

char* to_lower(const char* str) {
    // TODO: 実装する
}

char* reverse(const char* str) {
    // TODO: 実装する
}

int count_chars(const char* str) {
    // TODO: 実装する
}

void free_string(char* str) {
    free(str);
}
```

```rust
// build.rs
extern crate cc;

fn main() {
    // TODO: Cコードをビルドする
}
```

```rust
// src/lib.rs
use std::ffi::{CStr, CString};
use std::os::raw::{c_char, c_int};

// TODO: FFIバインディングを定義する

// TODO: 安全なRustインターフェースを提供する

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_to_upper() {
        assert_eq!(to_upper("hello").unwrap(), "HELLO");
    }
    
    #[test]
    fn test_to_lower() {
        assert_eq!(to_lower("HELLO").unwrap(), "hello");
    }
    
    #[test]
    fn test_reverse() {
        assert_eq!(reverse("hello").unwrap(), "olleh");
    }
    
    #[test]
    fn test_count_chars() {
        assert_eq!(count_chars("hello"), 5);
    }
}
```

```rust
// src/main.rs
use string_utils::{to_upper, to_lower, reverse, count_chars};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let s = "Hello, World!";
    
    println!("Original: {}", s);
    println!("Upper: {}", to_upper(s)?);
    println!("Lower: {}", to_lower(s)?);
    println!("Reverse: {}", reverse(s)?);
    println!("Length: {}", count_chars(s));
    
    Ok(())
}
```

### 演習3: 既存のCライブラリの利用

**目標**: 既存のCライブラリ（libcurl）を使用して、HTTPリクエストを行うRustプログラムを作成する

**要件**:
1. libcurlのFFIバインディングを作成する
2. 安全なRustインターフェースを提供する
3. HTTPリクエストを行い、レスポンスを取得する関数を実装する
4. エラー処理を適切に行う

**ヒント**:
- `pkg-config`を使用して、libcurlのリンク情報を取得する
- コールバック関数を使用して、レスポンスデータを取得する
- `unsafe`コードを最小限に保ち、安全な抽象化を作成する

**スケルトンコード**:

```rust
// build.rs
fn main() {
    // TODO: libcurlをリンクする
}
```

```rust
// src/lib.rs
use std::ffi::{CStr, CString};
use std::os::raw::{c_char, c_int, c_void};

// libcurlのFFIバインディング
#[allow(non_camel_case_types)]
type CURL = *mut c_void;

#[allow(non_camel_case_types)]
type CURLOPT = c_int;

// libcurlのオプション定数
const CURLOPT_URL: CURLOPT = 10002;
const CURLOPT_WRITEFUNCTION: CURLOPT = 20011;
const CURLOPT_WRITEDATA: CURLOPT = 10001;

// libcurlの関数
extern "C" {
    fn curl_easy_init() -> CURL;
    fn curl_easy_setopt(curl: CURL, option: CURLOPT, ...) -> c_int;
    fn curl_easy_perform(curl: CURL) -> c_int;
    fn curl_easy_cleanup(curl: CURL);
    fn curl_easy_strerror(code: c_int) -> *const c_char;
}

// TODO: 安全なRustインターフェースを提供する

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_http_get() {
        let response = http_get("https://httpbin.org/get").unwrap();
        assert!(response.contains("\"url\": \"https://httpbin.org/get\""));
    }
}
```

```rust
// src/main.rs
use curl_ffi::http_get;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let url = "https://httpbin.org/get";
    let response = http_get(url)?;
    
    println!("Response from {}: {}", url, response);
    
    Ok(())
}
```

これらの演習問題を通じて、RustからCコードを呼び出す方法や、安全な抽象化を作成する方法を学ぶことができます。FFIは強力なツールですが、適切に使用するには注意が必要です。演習を通じて、FFIの基本的な使い方と注意点を理解しましょう。

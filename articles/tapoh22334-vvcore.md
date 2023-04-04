---
title: "VOICEVOX COREをRustから使う方法"
emoji: "🥕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "Rust", "音声合成", "VOICEVOX" ]
published: false
---

## VOICEVOX CORE Rust Bindings

VOICEVOX COREの非公式のRust FFIラッパーを使用します。
このクレートは、VOICEVOX COREを呼び出すための高レベルAPIを提供します。

https://crates.io/crates/vvcore


## APIドキュメント

APIドキュメントはこちらにあります。
https://docs.rs/vvcore/0.0.2/vvcore/


## プロジェクトの作成

実際にサンプルを動かしてみましょう。
プロジェクトを作成します。
```
cargo new sample
cd sample
```

クレートを追加します。 
```
cargo add vvcore
```

## 依存ファイルの準備

以下の方法でVOICEVOX COREをダウンロードしてください。
https://github.com/VOICEVOX/voicevox_core#%E7%92%B0%E5%A2%83%E6%A7%8B%E7%AF%89

### Windowsの例

以下のコマンドでvoicevox_coreをダウンロードする。

```
Invoke-WebRequest https://github.com/VOICEVOX/voicevox_core/releases/latest/download/download-windows-x64.exe -OutFile ./download.exe
./download.exe
```

ダウンロードすると以下のような構成になる。

```
sample
├── Cargo.toml
├── src
│   └── main.rs
└── voicevox_core
    ├── model
    └── open_jtalk_dic_utf_8-1.11
    ...
```

## サンプルコード

main.rsに以下を記載。

```rust
use std::io::Write;
use vvcore::*;

fn main() {
    let dir = std::ffi::CString::new("open_jtalk_dic_utf_8-1.11").unwrap();
    let vvc = VoicevoxCore::new_from_options(AccelerationMode::Auto, 0, true, dir.as_c_str()).unwrap();

    let text: &str = "こんにちは";
    let speaker: u32 = 1;
    let wav = vvc.tts_simple(text, speaker).unwrap();

    let mut file = std::fs::File::create("audio.wav").unwrap();
    file.write_all(&wav.as_slice()).unwrap();
}
```

## ビルド

```
cargo build
```

## 実行

voicevox_coreディレクトリ内のファイルが実行時に必要です。
ビルドされたバイナリファイルをvoicevox_coreディレクトリに配置し実行してください。

```
sample
├── Cargo.toml
├── src
│   └── main.rs
└── voicevox_core
    ├── model
    ├── open_jtalk_dic_utf_8-1.11
    └── sample.exe <-------- ここに配置して実行する
    ...
```

---
title: "ChatGPTで自動プログラミング ~ラッパ関数を自動生成する呪文~"
emoji: "🥕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ChatGPT", "Rust"]
published: false
---

## はじめに

VOICEVOXという合成音声ソフトウェアをRustで直接呼び出せるようにするために、ラッパーライブラリを作成しました。
今回はChatGPTがFFIのラッパーを書いたので、その工程を紹介したいと思います。

https://crates.io/crates/vvcore


## VOICEVO CORE

VOICEVOXは、VOICEVO COREという名前でコアライブラリを公開しており、
FFI経由でコア機能にアクセスできます。

https://github.com/VOICEVOX/voicevox_core


## 全体像

全体の構想は以下の図のようになります。FFIが提供されているVOICEVOX COREを、低レベルなAPIでラップします。
その後、低レベルでunsafeなRustのAPIをよりRust-idiomaticなコードでラップして高レベルなAPIにします。

## ワークフロー

以下はUMLの図表で、Voicevox Coreのラッパー生成のフローを表しています。
VOICEVOX COREは、C言語のヘッダファイル(voicevox_core.h)と動的ライブラリが配布されています。
このヘッダファイルからRustのコードを生成して、アプリケーションからアクセスできるようにします。

### 1.低レベルAPI生成

「voicevox_core.h」からrust-bindgenを使用して、低レベルラッパを生成します。
生成された低レベルラッパは「voicevox_core.rs」に配置されます。

### 2.高レベルAPIの生成

高レベルラッパ生成はChatGPTが行います。
生成された高レベルラッパには手動で修正が必要です。
修正が完了するまで、手直しを繰り返します。
最終的に生成された高レベルラッパは「api.rs」に配置されます。

## 低レベルAPI生成

### 準備

```
cargo install bindgen-cli
```

### 生成

以下のコマンドで低レベルAPIを生成した。手直しは一切していない。

```
bindgen voicevox_core.h -o voicevox_core.rs
```


## 高レベルAPI生成

以下の呪文で生成します。
個数が多くなると出力が途中で止まるので、関数をひとつづつコピペして生成していきます。

```
 - Write wrapper function in rust code
 - Design interface Rust-idiomatic code
 - Insert doc comment
 - If return value is pointer value, wrap it into the original wrapper struct 
 - The wrapper struct frees pointer on RAII method
 - The wrapper struct has as_slice method


extern "C" {
    pub fn voicevox_tts(
        text: *const ::std::os::raw::c_char,
        speaker_id: u32,
        options: VoicevoxTtsOptions,
        output_wav_length: *mut usize,
        output_wav: *mut *mut u8,
    ) -> VoicevoxResultCode;
}

extern "C" {
    pub fn voicevox_wav_free(wav: *mut u8);
}
```

### ChatGPIの出力

Here's an example of a wrapper function for the voicevox_tts and voicevox_wav_free functions in Rust:

```
#[repr(C)]
pub struct VoicevoxWav {
    data: *mut u8,
    len: usize,
}

impl VoicevoxWav {
    pub fn as_slice(&self) -> &[u8] {
        unsafe { std::slice::from_raw_parts(self.data, self.len) }
    }
}

impl Drop for VoicevoxWav {
    fn drop(&mut self) {
        unsafe {
            voicevox_wav_free(self.data);
        }
    }
}

/// Generates speech audio from the given text using the specified speaker and options.
///
/// # Arguments
///
/// * `text` - The input text to generate speech from.
/// * `speaker_id` - The ID of the speaker to use.
/// * `options` - The options to use when generating speech.
///
/// # Returns
///
/// A `Result` containing a `VoicevoxWav` struct if successful, or a `VoicevoxResultCode` error code if not.
pub fn voicevox_tts(
    text: &str,
    speaker_id: u32,
    options: &VoicevoxTtsOptions,
) -> Result<VoicevoxWav, VoicevoxResultCode> {
    let c_text = CString::new(text).expect("CString::new failed");
    let mut output_wav_length: usize = 0;
    let mut output_wav: *mut u8 = std::ptr::null_mut();

    let result_code = unsafe {
        voicevox_tts(
            c_text.as_ptr(),
            speaker_id,
            *options,
            &mut output_wav_length,
            &mut output_wav,
        )
    };

    match result_code {
        VoicevoxResultCode::Success => {
            let wav = VoicevoxWav {
                data: output_wav,
                len: output_wav_length,
            };
            Ok(wav)
        }
        _ => Err(result_code),
    }
}
```

This code defines a VoicevoxWav struct to manage the output audio data from voicevox_tts, with an as_slice method to convert it to a slice of bytes. It also implements the Drop trait to automatically free the audio data using voicevox_wav_free.

The voicevox_tts function takes a &str for the input text, a u32 for the speaker ID, and a reference to a VoicevoxTtsOptions struct for the generation options. It returns a Result<VoicevoxWav, VoicevoxResultCode>, with the VoicevoxResultCode enum representing any errors that may occur.

The function first converts the input text to a CString for use with the C function, then calls voicevox_tts and checks the result code. If successful, it creates a VoicevoxWav struct from the output audio data and returns it in a Result::Ok variant. If an error occurs, it returns the error code in a Result::Err variant.

Note that the VoicevoxWav struct takes ownership of the audio data, so it will be automatically freed when it goes out of scope.


## 最後に

ChatGPTさん、なかなか良いコードを書いてくれますね。このような機械的な作業には、もうほとんどコードを書かなくてもよくなってきていますね。

クレートを作成するために行った作業は、主に名前の変更とコピー＆ペーストでした。複雑なことは言語で表現できないため、手動で作成する方が早い場合もありますね。

今後の開発においては、言語能力とプロンプト入力能力がますます重要になってくると思われます。


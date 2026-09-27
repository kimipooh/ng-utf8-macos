# macOS 27でng-utf8をインストールする方法：Homebrew Formulaの修正記録
[English](README-en.md)

macOS 27 に更新後、それまで使っていた ng がエラーで起動しなくなった。macOS 26 までは動作していた。以前の ng は Intel Mac でコンパイルしたと記憶しているが、この点は確認できていない。

そこで Homebrew の `ng-utf8` をインストールし直そうとしたところ、次の二つの問題が起きた。

1. `configure` がコンパイラの確認に使う `main(){return(0);}` は戻り値の型を書いていない。今回の Clang はこれをエラーとし、コンパイラ確認の段階で停止した。Formula の `CFLAGS` に `-Wno-error=implicit-int` を加えると、この段階を通過した。
2. コンパイルと `make install` の成功後、Formula が使用する `File.exists?` に対して Homebrew の Ruby が `NoMethodError` を返した。`File.exists?` は Ruby 3.2 で削除されたメソッドであり、`File.exist?` に修正するとインストールが完了した。

したがって、この手順で修正したのは、今回確認できたビルド時とインストール処理時の問題である。以前の ng が使えなくなった直接の原因を特定したものではない。

## 確認した環境

記録した機種は MacBook Pro 14インチの M4 Pro と M2 Pro である。次のバージョンは M4 Pro で記録したものである。

- macOS 27.0（arm64）
- Homebrew 7.0.6（`git describe`: `7.0.6-38-g27af95f6a3`）
- Apple Command Line Tools 27.0、Clang 21.0.0
- ng-utf8 1.5beta1（`matchy256/matchy/ng-utf8`）

M4 Pro のこの環境で、以下の修正を加えた後、Homebrewによるビルドとインストールが完了した。M2 Pro については、OS・ツールのバージョンと、どの操作まで確認したかをこの記録には残していない。

## 手順

以下は、今回実行した順序（tapの追加 → `brew update` → Formulaの信頼 → Formulaの編集 → インストール）で記載する。

### 1. tapを追加し、Formulaを信頼する

今回、最初に実行したtap追加コマンドは次のとおり（[matchy氏の紹介記事](https://tech.matchy.net/archives/344)に記載されたコマンド）。

```sh
brew tap matchy2/matchy
brew update
```

`brew update` は、次の手順でFormulaをローカル編集する前に実行する。

その後、Homebrewが認識したFormula名とローカルのtapディレクトリは `matchy256/matchy/ng-utf8` および `matchy256/homebrew-matchy` だった。ローカルのtapの取得元は `https://github.com/matchy256/homebrew-matchy.git` で、`https://github.com/matchy2/homebrew-matchy` へのアクセスはGitHub上で `matchy256/homebrew-matchy` へ転送される（2026年9月24日確認）。ただし、新規環境で `brew tap matchy2/matchy` または `brew tap matchy256/matchy` を実行した場合に同じ結果になるかは、今回の記録では未検証。Homebrewが認識したFormula名が `matchy256/matchy/ng-utf8` と異なる場合は、以降のコマンドを実行する前に、Formula名とtapの取得元を確認する。

```sh
brew trust --formula matchy256/matchy/ng-utf8
```

### 2. ローカルのFormulaを修正する

次のコマンドで、Formulaをnanoで開く。
viで開きたいなら、nano を vi に変更する。

```sh
HOMEBREW_EDITOR=nano brew edit matchy256/matchy/ng-utf8
```

`ng-utf8.rb` の既存の `CFLAGS` 指定に、次のオプションを追加する。既存のオプションは残す。

```text
-Wno-error=implicit-int
```

さらに、Formula内の次の呼び出しを変更する。

```text
File.exists?  →  File.exist?
```

修正前後の該当行は次のとおり（`-` が修正前、`+` が修正後。引用符の位置は元のFormulaのまま）。

```diff
-    system "CFLAGS=-'Wreturn-type -Wno-implicit-function-declaration' ./configure --enable-header_stdc --prefix=#{prefix}"
+    system "CFLAGS=-'Wreturn-type -Wno-implicit-function-declaration -Wno-error=implicit-int' ./configure --enable-header_stdc --prefix=#{prefix}"
```

```diff
-    unless (File.exists?(homerc)) then
+    unless (File.exist?(homerc)) then
```

なお、修正対象のFormulaは、今回の環境では次の場所にある。

```text
/opt/homebrew/Library/Taps/matchy256/homebrew-matchy/Formula/ng-utf8.rb
```

#### nanoでの編集方法

1. `Ctrl + W` を押し、`Wreturn-type` と入力して `Enter` で検索します。既存の `CFLAGS` の値の末尾（閉じ引用符 `'` の直前）に、半角スペースと `-Wno-error=implicit-int` を追加します。既存のオプションと引用符は残します。
2. もう一度 `Ctrl + W` を押し、`File.exists?` と入力して `Enter` で検索します。`File.exist?` に変更します（`s` を1文字削除）。
3. `Ctrl + O` を押し、ファイル名の確認が表示されたらそのまま `Enter` を押して保存します。続けて `Ctrl + X` で終了します。

`Ctrl` はMacの**Controlキー**です。Commandキーではありません。

### 3. インストールする

```sh
brew install --keep-tmp matchy256/matchy/ng-utf8
```

`--keep-tmp` は失敗時の調査用に一時ファイルを残すための指定で、通常の利用には必須ではない。今回の実行では `configure`、`make`、`make install` が成功し、Homebrewは次を表示した。

```text
/opt/homebrew/Cellar/ng-utf8/1.5beta1: 32 files, 1MB, built in 9 seconds
```

## 2箇所の修正が必要だった理由

1. 元のFormulaのままでは、古い `configure` がコンパイラ確認に使うテストプログラム `main(){return(0);}`（戻り値の型を省略した関数定義）を、Clang 21が `-Wimplicit-int` のエラーとして扱い、コンパイラ確認で停止した。既存の `CFLAGS` に `-Wno-error=implicit-int` を追加してこの診断をエラーではなく警告として扱わせたところ、`configure` とng本体のビルドが進んだ。
2. `make install` の後、Formulaが `File.exists?` を呼び、HomebrewのRubyで `NoMethodError` が発生した。同じ意味のメソッド `File.exist?` への変更後、Homebrewのインストール処理が完了した。

これは古いFormulaを今回の環境でビルドするためのローカルFormulaの互換性修正の記録であり、ng本体のCコードを現代の規格に合わせて修正したものではない。Formulaを更新する際は、ローカルで加えた2箇所の修正が維持されているか確認する。

## 参考資料

### ngとUTF-8対応版

- MURAMATSU Atsushi, [ng（GitHubリポジトリ）](https://github.com/amuramatsu/ng) — `LICENSE`、`COPYING`、ソースを確認するための資料。今回のFormulaは、`brew install` の実行時に、GitHubではなく `http://tt.sakura.ne.jp/~amura/archives/ng/ng-1.5beta1.tar.gz` からソースを取得する。
- [ng UTF-8 対応版（薄明日記、2005年）](https://startide.jp/diary/?2005/1/10/2) — Mac OS X向けの初期のUTF-8対応版に関する記録。今回のFormulaが適用するUTF-8パッチとの関係は、この記録では確認していない。
- matchy, [私家版Homebrew：Ng-utf8](https://tech.matchy.net/archives/344) — 今回利用したHomebrew版の紹介。`brew tap matchy2/matchy` の出典。
- jm8tsj, [ng editor インストール（Mac mini M1 OS 12.6下）](https://jm8tsj.com/2024/02/23/ng-editor-install-with-homebew/) — 旧環境でのインストール例。
- harenuma, [geminiくんにngをmacで動くようにしてもらった](https://harenuma.hatenablog.com/entry/2026/03/08/083355)（2026年3月8日）— M4 Mac miniで、Geminiの支援を受けてngを動作させたという報告。今回のHomebrew Formulaの修正とは別の記録。

### 今回のエラーと修正の根拠

- Homebrew, [Tap Trust](https://docs.brew.sh/Tap-Trust) — `brew trust --formula` の説明。
- Clang, [Diagnostic flags in Clang](https://clang.llvm.org/docs/DiagnosticsReference.html) — `-Wimplicit-int` の診断内容。
- Ruby, [File.exist?](https://docs.ruby-lang.org/ja/latest/method/File/s/exist%3D3f.html) — Formulaで使用するファイル存在確認メソッド。
- Ruby, [Ruby 3.2.0 Released](https://www.ruby-lang.org/en/news/2022/12/25/ruby-3-2-0-released/) — 「Removed methods」に、削除された非推奨メソッドとして `File.exists?` が記載されている（2026年9月27日閲覧）。

最終閲覧日：2026年9月24日（閲覧日を個別に記した資料を除く）。

## 著作者とライセンス

このインストール手順書の著作者：Kimiya Kitani

Copyright © 2026 Kimiya Kitani.

このREADMEの文章は [Creative Commons Attribution 4.0 International
（CC BY 4.0）](https://creativecommons.org/licenses/by/4.0/) の下で公開します。
Kimiya Kitaniはこの手順書の著作者であり、ng本体、UTF-8パッチ、Homebrew Formulaの著作者ではありません。
このリポジトリには、ng本体のソース、UTF-8パッチ、Homebrew Formulaのファイルは含まれていません。
ng本体、UTF-8パッチおよび第三者のHomebrew Formula（上記の修正前後の行として引用した部分を含む）には、
このREADMEのCC BY 4.0は適用されません。それぞれの再利用・再配布には、権利者の定める条件を確認してください。

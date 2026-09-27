# How to Install ng-utf8 on macOS 27: A Record of Homebrew Formula Modifications
[日本語](README.md)

After I updated to macOS 27, the ng I had been using failed to start with an error. It had worked through macOS 26. I believe I compiled that earlier ng on an Intel Mac, but I have not been able to confirm this.

I then tried to reinstall `ng-utf8` with Homebrew and ran into the following two problems.

1. The test program `main(){return(0);}` that `configure` uses to check the compiler omits the return type. The Clang used here treats this as an error, so the process stopped at the compiler check. Adding `-Wno-error=implicit-int` to the Formula's `CFLAGS` got it past this stage.
2. After compilation and `make install` succeeded, Homebrew's Ruby raised a `NoMethodError` for `File.exists?`, which the Formula calls. `File.exists?` was removed in Ruby 3.2; changing it to `File.exist?` allowed the installation to complete.

These steps therefore fix only the build-time and installation-time problems encountered here. They do not identify the direct cause of the earlier ng no longer working.

## Tested Environment

The machines recorded are 14-inch MacBook Pros with M4 Pro and M2 Pro. The following versions were recorded on the M4 Pro.

- macOS 27.0 (arm64)
- Homebrew 7.0.6 (`git describe`: `7.0.6-38-g27af95f6a3`)
- Apple Command Line Tools 27.0, Clang 21.0.0
- ng-utf8 1.5beta1 (`matchy256/matchy/ng-utf8`)

In this environment on the M4 Pro, after applying the modifications below, the build and installation with Homebrew completed. For the M2 Pro, this record does not include the OS and tool versions or how far the steps were checked.

## Steps

The steps below are listed in the order in which they were actually performed (add the tap → `brew update` → trust the Formula → edit the Formula → install).

### 1. Add the tap and trust the Formula

The tap command run first was the following (the command given in [matchy's introductory article](https://tech.matchy.net/archives/344), in Japanese).

```sh
brew tap matchy2/matchy
brew update
```

Run `brew update` before editing the Formula locally in the next step.

After that, the Formula name recognized by Homebrew and the local tap directory were `matchy256/matchy/ng-utf8` and `matchy256/homebrew-matchy`, respectively. The local tap was fetched from `https://github.com/matchy256/homebrew-matchy.git`, and on GitHub, access to `https://github.com/matchy2/homebrew-matchy` is redirected to `matchy256/homebrew-matchy` (checked on September 24, 2026). However, this record has not verified whether running `brew tap matchy2/matchy` or `brew tap matchy256/matchy` in a fresh environment produces the same result. If Homebrew recognizes a Formula name other than `matchy256/matchy/ng-utf8`, check the Formula name and the tap source before running the remaining commands.

```sh
brew trust --formula matchy256/matchy/ng-utf8
```

### 2. Modify the local Formula

Open the Formula in nano with the following command.
If you prefer vi, change nano to vi.

```sh
HOMEBREW_EDITOR=nano brew edit matchy256/matchy/ng-utf8
```

Add the following option to the existing `CFLAGS` setting in `ng-utf8.rb`. Keep the existing options.

```text
-Wno-error=implicit-int
```

Also change the following call in the Formula.

```text
File.exists?  →  File.exist?
```

The affected lines before and after the change are shown below (`-` is before, `+` is after; the quotation marks are positioned as in the original Formula).

```diff
-    system "CFLAGS=-'Wreturn-type -Wno-implicit-function-declaration' ./configure --enable-header_stdc --prefix=#{prefix}"
+    system "CFLAGS=-'Wreturn-type -Wno-implicit-function-declaration -Wno-error=implicit-int' ./configure --enable-header_stdc --prefix=#{prefix}"
```

```diff
-    unless (File.exists?(homerc)) then
+    unless (File.exist?(homerc)) then
```

In this environment, the Formula to be modified was located at:

```text
/opt/homebrew/Library/Taps/matchy256/homebrew-matchy/Formula/ng-utf8.rb
```

#### Editing in nano

1. Press `Ctrl + W`, type `Wreturn-type`, and press `Enter` to search. At the end of the existing `CFLAGS` value (just before the closing quotation mark `'`), add a space followed by `-Wno-error=implicit-int`. Keep the existing options and quotation marks.
2. Press `Ctrl + W` again, type `File.exists?`, and press `Enter` to search. Change it to `File.exist?` (delete the single character `s`).
3. Press `Ctrl + O`; when the file name prompt appears, press `Enter` as is to save. Then press `Ctrl + X` to exit.

`Ctrl` is the **Control key** on the Mac, not the Command key.

### 3. Install

```sh
brew install --keep-tmp matchy256/matchy/ng-utf8
```

`--keep-tmp` keeps temporary files for troubleshooting in case of failure and is not required for normal use. In this run, `configure`, `make`, and `make install` succeeded, and Homebrew displayed the following.

```text
/opt/homebrew/Cellar/ng-utf8/1.5beta1: 32 files, 1MB, built in 9 seconds
```

## Why the Two Modifications Were Needed

1. With the original Formula, Clang 21 treated the test program `main(){return(0);}` (a function definition that omits the return type), which the old `configure` uses to check the compiler, as a `-Wimplicit-int` error, and the compiler check stopped. After adding `-Wno-error=implicit-int` to the existing `CFLAGS` so that this diagnostic is treated as a warning rather than an error, `configure` and the build of ng itself proceeded.
2. After `make install`, the Formula called `File.exists?`, which raised a `NoMethodError` in Homebrew's Ruby. After changing it to `File.exist?`, the method with the same meaning, Homebrew's installation process completed.

This is a record of local compatibility fixes to an old Formula so that it builds in this environment; it is not a modification of ng's own C code to conform to modern standards. When updating the Formula, check that the two local modifications are still in place.

## References

### ng and the UTF-8 Version

- MURAMATSU Atsushi, [ng (GitHub repository)](https://github.com/amuramatsu/ng) — for checking `LICENSE`, `COPYING`, and the source. During `brew install`, the Formula used here downloads the source from `http://tt.sakura.ne.jp/~amura/archives/ng/ng-1.5beta1.tar.gz`, not from GitHub.
- [ng UTF-8 対応版（薄明日記、2005年）](https://startide.jp/diary/?2005/1/10/2) (in Japanese; "ng UTF-8 version", Hakumei Nikki, 2005) — a record of an early UTF-8 version for Mac OS X. This record has not confirmed its relationship to the UTF-8 patch applied by the Formula used here.
- matchy, [私家版Homebrew：Ng-utf8](https://tech.matchy.net/archives/344) (in Japanese; "Private Homebrew: Ng-utf8") — introduction to the Homebrew version used here. The source of `brew tap matchy2/matchy`.
- jm8tsj, [ng editor インストール（Mac mini M1 OS 12.6下）](https://jm8tsj.com/2024/02/23/ng-editor-install-with-homebew/) (in Japanese; "Installing the ng editor (on Mac mini M1, OS 12.6)") — an installation example in an older environment.
- harenuma, [geminiくんにngをmacで動くようにしてもらった](https://harenuma.hatenablog.com/entry/2026/03/08/083355) (in Japanese; "I had Gemini get ng working on my Mac", March 8, 2026) — a report of getting ng to work on an M4 Mac mini with help from Gemini. A separate record from the Homebrew Formula modifications described here.

### Basis for the Errors and Fixes

- Homebrew, [Tap Trust](https://docs.brew.sh/Tap-Trust) — explanation of `brew trust --formula`.
- Clang, [Diagnostic flags in Clang](https://clang.llvm.org/docs/DiagnosticsReference.html) — details of the `-Wimplicit-int` diagnostic.
- Ruby, [File.exist?](https://docs.ruby-lang.org/ja/latest/method/File/s/exist%3D3f.html) (in Japanese) — the file-existence check method used in the Formula.
- Ruby, [Ruby 3.2.0 Released](https://www.ruby-lang.org/en/news/2022/12/25/ruby-3-2-0-released/) — lists `File.exists?` under "Removed methods" as a removed deprecated method (accessed September 27, 2026).

Last accessed: September 24, 2026 (except where an access date is given for an individual reference).

## Author and License

Author of these installation instructions: Kimiya Kitani

Copyright © 2026 Kimiya Kitani.

The text of this README is published under the [Creative Commons Attribution 4.0 International
(CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/) license.
Kimiya Kitani is the author of these instructions, not of ng itself, the UTF-8 patch, or the Homebrew Formula.
This repository does not contain the ng source code, the UTF-8 patch, or the Homebrew Formula files.
The CC BY 4.0 license of this README does not apply to ng itself, the UTF-8 patch, or the third-party Homebrew Formula (including the portions quoted above as the lines before and after modification).
For reuse or redistribution of each of these, check the terms set by the respective rights holders.

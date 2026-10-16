# velopack-nonascii-delta-repro

Minimal reproduction: on a GitHub-hosted `windows-latest` runner, `vpk pack` fails to build a delta package when `--packTitle` contains non-ASCII characters (Japanese and Korean here).

## Symptom

Packing the second release fails while creating the delta:

```
[ERR] Failed to create zstd diff for file 'C:\Users\runneradmin\AppData\Local\Temp\velopack\temp.3\lib\app\テストアプリ_ExecutionStub.exe'.
Process failed: '...\vpk\1.2.161\vendor\zstd.exe --patch-from ...\temp.2\lib\app\テストアプリ_ExecutionStub.exe -o ...\temp.3\lib\app\テストアプリ_ExecutionStub.exe.zsdiff --force --single-thread ...\temp.3\lib\app\テストアプリ_ExecutionStub.exe'
zstd: error 31 : Stat failed on dictionary file C:\Users\runneradmin\AppData\Local\Temp\velopack\temp.2\lib\app\??????_ExecutionStub.exe: No such file or directory
[FTL] Delta creation failed for one or more files. See log for details. To skip delta generation, use the '--delta none' argument.
```

Which non-ASCII characters fail depends on the environment. For example, the Japanese title passes on Windows 11 with a Japanese system locale, while `café` passes on the hosted runner:

| Script           | Name          | GitHub-hosted runner (en-US) | Windows 11 (ja-JP) |
| ---------------- | ------------- | ---------------------------- | ------------------ |
| ASCII            | `ascii_test`  | ✅                            | ✅                  |
| Japanese         | `テストアプリ` | ❌ fails                      | ✅                  |
| Korean           | `테스트`       | ❌ fails                      | ❌ fails            |
| Latin (French)   | `café`        | ✅                            | ❌ fails            |
| Latin (German)   | `Größe`       | ✅                            | ❌ fails            |
| Latin (Polish)   | `Łódź`        | ❌ fails                      | ❌ fails            |
| Cyrillic         | `Тест`        | ❌ fails                      | ✅                  |
| Greek            | `Τεστ`        | ❌ fails                      | ✅                  |
| Arabic           | `اختبار`      | ❌ fails                      | ❌ fails            |
| Hebrew           | `בדיקה`       | ❌ fails                      | ❌ fails            |
| Thai             | `ทดสอบ`       | ❌ fails                      | ❌ fails            |
| Devanagari       | `परीक्षण`     | ❌ fails                      | ❌ fails            |
| Emoji            | `test😀`      | ❌ fails                      | ❌ fails            |

Every failure is the same `zstd: error 31 : Stat failed on dictionary file`. The runner column comes from `vpk pack` (run the workflow on the [`explore/charsets`](../../tree/explore/charsets) branch). The ja-JP column comes from running the bundled `zstd.exe --patch-from` directly; for Japanese and Korean, `vpk pack` gave the same result there.

## When it happens

All of these must be true:

- vpk includes [#985](https://github.com/velopack/velopack/pull/985) (1.2.158 or later). vpk 1.2.0 is not affected.
- `--packTitle` contains a character that the bundled `zstd.exe` cannot open on that machine. Which characters these are depends on the environment (see the table above).
- A delta is built (`--delta` is not `none`).
- The previous release was packed the same way, so it contains a stub with the same name. The first release after upgrading from vpk 1.2.0 still succeeds, because the previous package has no stub under that name.

## What is known

- Since [#985](https://github.com/velopack/velopack/pull/985) (the fix for [#982](https://github.com/velopack/velopack/issues/982)), the launcher stub in the package is named after `--packTitle` (`<packTitle>_ExecutionStub.exe`).
- vpk 1.2.0 named the stub after `--mainExe` (`ReproApp_ExecutionStub.exe`). With that name, `zstd.exe` builds the diff without error.
- When both the previous and the new release contain that stub, vpk diffs it with its bundled `zstd.exe`.
- vpk logs the correct path in the `zstd.exe` command line, but zstd's error shows the non-ASCII characters as `?` and reports that the file does not exist.
- Running the bundled `zstd.exe` directly, without vpk, on files with non-ASCII names fails the same way, so `zstd.exe` does not seem to handle non-ASCII characters in paths reliably.
- This matches [facebook/zstd#3882](https://github.com/facebook/zstd/issues/3882) ("zstd fails to process some filenames on Windows"), which is still open. vpk 1.2.161 bundles zstd v1.5.5.

## Reproduce

Actions → **Reproduce non-ASCII delta failure** → Run workflow.

Each job packs `1.0.0` and then `1.0.1` (which builds the delta). The job result is the result of `vpk pack`.

| Job | vpk | `--packTitle` | `--delta` | Result |
|---|---|---|---|---|
| non-ASCII packTitle | 1.2.161 | `テストアプリ` | default | ❌ fails |
| Korean packTitle | 1.2.161 | `테스트` | default | ❌ fails |
| ASCII packTitle | 1.2.161 | `Test App` | default | ✅ passes |
| non-ASCII packTitle, `--delta none` | 1.2.161 | `テストアプリ` | `none` | ✅ passes |
| non-ASCII packTitle, vpk 1.2.0 | 1.2.0 | `テストアプリ` | default | ✅ passes |

## Workaround

Pass `--delta none` to `vpk pack` (updates then always download the full package).

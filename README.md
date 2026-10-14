# velopack-nonascii-delta-repro

Minimal reproduction: on a GitHub-hosted `windows-latest` runner, `vpk pack` fails to build a delta package when `--packTitle` contains non-ASCII characters (Japanese here).

## Symptom

Packing the second release fails while creating the delta:

```
[ERR] Failed to create zstd diff for file 'C:\Users\runneradmin\AppData\Local\Temp\velopack\temp.3\lib\app\テストアプリ_ExecutionStub.exe'.
Process failed: '...\vpk\1.2.161\vendor\zstd.exe --patch-from ...\temp.2\lib\app\テストアプリ_ExecutionStub.exe -o ...\temp.3\lib\app\テストアプリ_ExecutionStub.exe.zsdiff --force --single-thread ...\temp.3\lib\app\テストアプリ_ExecutionStub.exe'
zstd: error 31 : Stat failed on dictionary file C:\Users\runneradmin\AppData\Local\Temp\velopack\temp.2\lib\app\??????_ExecutionStub.exe: No such file or directory
[FTL] Delta creation failed for one or more files. See log for details. To skip delta generation, use the '--delta none' argument.
```

Which non-ASCII characters fail depends on the environment. The same Japanese title passes on Windows 11 with a Japanese system locale, where a Korean file name fails instead:

| Environment                  | Japanese name (`テストアプリ`) | Korean name (`테스트`) | ASCII name |
| ---------------------------- | ------------------------------ | ---------------------- | ---------- |
| GitHub-hosted runner (en-US) | ❌ fails                        | not tested             | ✅          |
| Windows 11 (ja-JP)           | ✅                              | ❌ fails                | ✅          |

## What is known

- Since [#982](https://github.com/velopack/velopack/issues/982), the launcher stub in the package is named after `--packTitle` (`<packTitle>_ExecutionStub.exe`); vpk 1.2.0 named it after `--mainExe` (`ReproApp_ExecutionStub.exe`), which it diffs with `zstd.exe` without error.
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
| ASCII packTitle | 1.2.161 | `Test App` | default | ✅ passes |
| non-ASCII packTitle, `--delta none` | 1.2.161 | `テストアプリ` | `none` | ✅ passes |
| non-ASCII packTitle, vpk 1.2.0 | 1.2.0 | `テストアプリ` | default | ✅ passes |

## Workaround

Pass `--delta none` to `vpk pack` (updates then always download the full package).

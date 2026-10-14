# velopack-nonascii-delta-repro

Minimal reproduction: `vpk pack` fails to build a delta package when `--packTitle` contains characters that the machine's ANSI code page cannot represent (e.g. Japanese on a GitHub-hosted `windows-latest` runner, code page 1252).

## Symptom

Packing the second release fails while creating the delta:

```
[ERR] Failed to create zstd diff for file '...\lib\app\テストアプリ_ExecutionStub.exe'.
zstd: error 31 : Stat failed on dictionary file ...\lib\app\??????(???)_ExecutionStub.exe: No such file or directory
[FTL] Delta creation failed for one or more files. See log for details. To skip delta generation, use the '--delta none' argument.
```

The same commands succeed on a machine whose system code page can encode the title (e.g. Windows with a Japanese system locale, code page 932).

## Cause (as observed)

- Since [#982](https://github.com/velopack/velopack/issues/982), the launcher stub in the package is named after `--packTitle` (`<packTitle>_ExecutionStub.exe`); vpk 1.2.0 named it after `--mainExe`.
- When both the previous and the new release contain that stub, vpk diffs it with its bundled `zstd.exe`.
- The path reaches `zstd.exe` through the ANSI code page, so unrepresentable characters become `?` and the file cannot be opened.

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

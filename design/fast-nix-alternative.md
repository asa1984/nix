# Design Doc: Nix 互換の高速な代替ツール

| 項目 | 内容 |
|---|---|
| ステータス | Draft |
| 最終更新 | 2026-10-01 |
| 対象 | 本ツールの設計・実装に関わる開発者 |
| 前提とした Nix | NixOS/nix master（2.36.0 dev, `209d2bc`） |

本文中では、開発するツールを「**本ツール**」、既存の C++ 実装を「**C++ Nix**」と呼ぶ。
`src/...:行番号` という参照は、すべて上記コミット時点の C++ Nix のソースを指す。

---

## 1. 概要

nixpkgs をそのままビルドでき、`cache.nixos.org` などの既存バイナリキャッシュを利用できることを最低限の互換性とする。そのうえで C++ Nix の設計上の負債を持ち込まずに作り直した、高速な Nix 代替ツールを開発する。

- flake と `nix` コマンド（new CLI）を前提とし、experimental の区別をなくす。
- content-addressed derivation（CA）と dynamic derivation を最初から利用可能にする。
- 互換性の難所は「評価器が C++ Nix と**バイト単位で同じ `.drv`** を出すこと」にほぼ集約される。そこで最初の成果物を評価器とし、ストアは当面既存の `nix-daemon` に任せる段階的な戦略をとる。

---

## 2. 背景と動機

C++ Nix のコードを調査し、以下の性能低下要因と技術的負債を確認した。いずれも漸進的な修正では解消しにくく、新規設計で回避する価値が高い。

### 2.1 flake 以前・以後の分断

| 問題 | C++ Nix の該当箇所 | 影響 |
|---|---|---|
| final input をフェッチする前に substituter を確認し、ヒットするとソース全体をダウンロードする。lazy trees の効果を打ち消す | `src/libfetchers/fetchers.cc:312-351`（`FIXME: add a setting to disable this.`） | 大 |
| git 管理外の flake（`path:`）は毎回ディレクトリ全体を store にコピーする | `src/libfetchers/path.cc:147-172` | 大 |
| `NIX_PATH=…=flake:…` や `-I flake:` は lazy trees を通らず全体をコピーする。ロジックも2箇所に重複 | `src/libcmd/common-eval-args.cc:28-41, 184-194` | 中〜大 |
| dirty な作業ツリーでは、1ファイル変えただけで全体を NAR として再ハッシュする。submodule があると fingerprint 自体が作れず、毎回全体をハッシュし eval cache も無効になる | `src/libfetchers/git.cc:1116-1144`（特に 1127） | 大 |
| eval cache を引く前に `lockFlake` を全部実行する | `src/libflake/flake.cc:1015-1017`（FIXME） | 大 |
| eval cache は pure eval でしか有効にならない。legacy 式（`--file`、`nix-build` など）では一切使えない | `src/libflake/flake.cc:1023-1026`、`src/libcmd/installable-value.cc:9-10` | 中 |
| filter 付きの `builtins.path` と `filterSource` はキャッシュされず、毎回シリアライズし直す | `src/libfetchers/fetch-to-store.cc:56,64` | 中〜大 |
| `builtins.getFlake` がメモ化されない | `src/libflake/flake-primops.cc:33-72` | 中 |
| `--file` を指定するとグローバルな `pureEval` を書き換える | `src/libcmd/installables.cc:455-460` | 保守性 |

### 2.2 評価器とストア

| 問題 | C++ Nix の該当箇所 | 影響 |
|---|---|---|
| thunk の blackhole が単純な代入で、並列評価の土台がない | `src/libexpr/include/nix/expr/eval-inline.hh:96-123` | 大（構造的） |
| string context を補間のたびにパースして `std::set` に入れ直し、GC 文字列を再生成する | `src/libexpr/eval.cc:906-916, 2476-2481` | 大 |
| `derivationStrict` が drv を三重にハッシュし、書き込み前に daemon との往復が2回ある | `src/libexpr/primops.cc:1911-1930`、`src/libstore/derivations.cc:118-143` | 大 |
| `isValidPath` は「存在しない」結果しかキャッシュしない | `src/libstore/store-api.cc:538-565` | 大 |
| daemon 接続の既定値が1本。path info を一括で問い合わせる op がない | `remote-store.hh:36-37`、`worker-protocol.hh:282` | 大 |
| ローカルパス（`src = ./.`）がプロセスをまたいでキャッシュされない | `source-accessor.hh:232-235` | 大 |
| グローバル設定への参照が約 365 箇所あり、並列化やマルチテナント化を阻む | `src/libstore/include/nix/store/globals.hh:431` ほか | 保守性 |
| build worker が単一スレッドで、出力の登録（3回読む）が他のビルドを止める | `src/libstore/build/worker.cc:374-470`、`derivation-builder-impl.cc:222,428,635` | 中〜大 |
| LocalStore は SQLite 接続1本を mutex で保護し、GC は削除1件ごとに1トランザクション | `local-store.cc`、`gc.cc:594-725` | 中〜大 |

### 2.3 CA と dynamic derivation

| 問題 | C++ Nix の該当箇所 |
|---|---|
| dynamic derivation が ca-derivations に暗黙に依存している（feature 定義に依存関係の記述がない） | `downstream-placeholder.cc:15`、`derivation-resolution-goal.cc:99` |
| build trace を resolve 前のキーで引くため、最初の問い合わせが必ず外れる | `derivation-goal.cc:88-106` と `derivation-builder-impl.cc:693-705` の食い違い |
| substitute するためだけに入力を全部 realise させる | `derivation-resolution-goal.cc:40-73`（上流 #11928） |
| build trace が GC されず、非決定的なビルドの再ビルドを阻む | `ca-specific-schema.sql`、`local-store.cc:682-700` |
| `queryMissing` と `--dry-run` が dynamic derivation を無視する | `src/libstore/misc.cc:194-200` |
| エラーにならずに `abort`/`assert` で落ちる。エラーの握りつぶしもある | `derivation-goal.cc:246,252`、`derivation/resolution.cc:64-68` |
| build trace のキー変更でプロトコル互換が壊れ、古い daemon に対して黙って劣化する | `remote-store.cc:559-564` |
| goal（scheduler）の構造そのものが負債 | 上流 issue #12628 |

C++ Nix 側の CA 安定化マイルストーン（#35）は 71%、dynamic derivation（#39）は 85% にとどまる。本ツールでは CA を常に有効にすることで、この依存関係の問題自体をなくす。

---

## 3. ゴールと非ゴール

### 3.1 ゴール

1. **G1 nixpkgs の互換性**：固定した nixpkgs のリビジョンで、評価できる全属性の drvPath が C++ Nix と一致する。
2. **G2 既存のバイナリキャッシュを利用できる**：`cache.nixos.org` やサードパーティ（Cachix、Attic、S3/HTTP の binary cache）から narinfo と NAR を取得して検証できる。
3. **G3 flake が既定**：`flake.nix` と `flake.lock`（v7）をそのまま扱える。experimental feature のフラグは不要にする。
4. **G4 CA と dynamic derivation**：常に有効にし、C++ Nix と同じ drv 形式（`outputHashMode = "text"`、`builtins.outputOf`、`drv^out^out`）を扱える。
5. **G5 高速**：§9 の指標で、C++ Nix と比べて明確な改善を示す。
6. **G6 戻りやすさ**：既存の NixOS マシンに入れても、いつでも C++ Nix に戻れる。

### 3.2 非ゴール

- C++ Nix の全機能を再現すること。
- legacy CLI：`nix-env`、チャンネル（`nix-channel`）、`NIX_PATH` による `<…>` の解決（`<nix/…>` の corepkgs は除く。§4.3 を参照）。
- serve protocol（`ssh://`）、古い worker protocol バージョン（下限は §6.5 で決める）。
- Windows 対応（初期段階）。
- nixpkgs が使っていない、あるいは非推奨の builtin を網羅すること。

---

## 4. 互換性の契約

### 4.1 必須（MUST）

| 対象 | 一致の基準 | 理由 |
|---|---|---|
| derivation の ATerm 形式 | バイト単位で一致 | drv の store path と、input-addressed な出力パスは、ここから計算される |
| store path の計算（input-addressed と CA の両方）、hash modulo | 完全一致 | キャッシュヒットの前提 |
| NAR 形式 | 完全一致 | narHash と substitute の前提 |
| narinfo、`nix-cache-info`、ed25519 署名の形式 | 読み込みと検証ができる | キャッシュの利用 |
| store の場所 | `/nix/store` 固定 | パス文字列に埋め込まれている |
| `flake.lock` v7、`flake.nix` のスキーマ、flake reference の構文 | 読み書きの両方に対応 | 既存リポジトリでそのまま動くため |
| 評価結果 | nixpkgs と NixOS モジュールで drvPath が一致 | G1 |

### 4.2 推奨（SHOULD）

- 必要最小限の worker protocol：既存の `nix` CLI から本ツールの daemon を、本ツールの CLI から既存の `nix-daemon` を使えるようにする。
- `db.sqlite` のスキーマ互換（§6.5.3）。
- C++ Nix のサンドボックスと同等のビルド環境：ビルドディレクトリのパス、uid/gid、`/bin/sh`、ネットワーク分離、fixed-output derivation だけネットワークを許可。
  - input-addressed な出力パスはビルド環境に依存しないので、キャッシュとの互換には影響しない。ただし、パッケージがビルドに成功するか、再現性が保てるかに直結する。
- derivation の JSON 形式（v4）。
- build trace の JSON 形式（`build-trace-entry-v3`）と、binary cache 上の配置（`build-trace-v2/<drvPath>/<out>.doi`）。

### 4.3 評価器で互換の落とし穴になる箇所

評価器の互換性で、特に注意すべき点の一覧。それぞれ差分テストで個別に確認する。

- **言語の基準バージョン**：nixpkgs が要求する最低 Nix バージョンに合わせる（25.11 時点で 2.18。追随の仕組みは §10 を参照）。`builtins.nixVersion` と `langVersion` はこの基準を名乗る。
- **corepkgs**：stdenv の bootstrap が `import <nix/fetchurl.nix>` を使う。`builtin:fetchurl` builder も含めて実装が必要。
- **string context**：`inputDrvs` と `inputSrcs` を決める。`unsafeDiscardStringContext`、`unsafeDiscardOutputDependency`、`appendContext`、`getContext` を含めて完全に同じ意味にする。
- **`derivationStrict` の細部**：`__structuredAttrs`、`__json`、`outputHashMode` / `outputHashAlgo` の組み合わせ、`allowedReferences` 系、`__impureHostDeps`、`requiredSystemFeatures`、`passAsFile`、`exportReferencesGraph`、`__contentAddressed`、`__impure`。
- **値の文字列化**：float の出力形式（`toString`、`toJSON`）、整数のオーバーフロー時の挙動（64 ビット符号付き、オーバーフローはエラー）、`toXML`。
- **derivation の env への変換（`coerceToString` の癖）**：
  - `true` は `"1"`、`false` と `null` は `""`。
  - **float は `std::to_string`（`%f`、小数点以下6桁固定）**。
  - リストは空白区切りで連結するが、空のネストリストの後には空白を入れない（`eval.cc:2590-2629`、`/* !!! not quite correct */`）。
  - `__structuredAttrs` の JSON は、nlohmann/json のキー順、float 表記、エスケープ規則で出力される（`parsed-derivations.cc:58`）。
- **順序**：`attrNames` と `attrValues` は名前のバイト単位の辞書順、`sort` は安定ソート、`toJSON` のキー順。
  - **attrset の内部順序は自由に決めてよい。** 2022 年（`8775be339`）より前の C++ Nix は、シンボルを文字列のアドレスで比較していた。つまり内部順序は malloc 次第で、意味を持ったことがない。
- **正規表現**：`match` と `split` は libstdc++ の `std::regex`（POSIX extended）の挙動に、バグまで含めて合わせる。C++ Nix 自身が boost::regex に移行しようとしたが、方言の違い（`"\\{}"`）が原因で revert している（`908a011a4`）。
- **masked hash の衝突時の挙動**：2026-08 の `b26204ba1`（#16317）で、ハッシュが衝突した入力 drv の出力集合をマージするよう変わり、該当する drv の store path が変わった。どの Nix バージョンの挙動に合わせるかを決める（§11 Q11）。
- **`//` の評価順序**：連鎖した `a // b // c` は右辺から強制される（`eval.cc:2081-2087`）。影響するのは、どのエラーが報告されるかだけ。
- **パーサ**：`fromTOML`、`fromJSON` の境界的な挙動（重複キー、大きな数値）。
- **パスの扱い**：パス値の正規化、`baseNameOf`、`dirOf`、パスから文字列への変換時に store にコピーされるタイミング。
- **ハッシュと名前**：`hashString`、`hashFile`、`convertHash`、store path の name の検証規則。
- **エラーの種類**：`tryEval` で捕捉されるのは `throw` と `assert` だけで、`abort` は捕捉されないという区別。

---

## 5. 全体アーキテクチャ

```
                         ┌───────────────────────────────┐
  CLI (build/eval/        │        tool-cli               │
  develop/profile/...)    │  installable 解決・UI・ログ     │
                         └──────┬───────────────┬────────┘
                                │               │
                 ┌──────────────▼───┐   ┌───────▼───────────────┐
                 │  tool-flake      │   │  tool-eval-cache      │
                 │  lock / registry │   │  (fingerprint 別 DB)  │
                 └──────┬───────────┘   └───────┬───────────────┘
                        │                       │
                 ┌──────▼───────────────────────▼──────┐
                 │  tool-eval                          │
                 │  parser → 静的スコープ解決 → bytecode │
                 │  並列 VM / GC / builtins             │
                 │  drv グラフ（メモリ上）               │
                 └──────┬──────────────────┬───────────┘
                        │                  │
              ┌─────────▼──────┐   ┌───────▼────────────┐
              │ tool-source    │   │ tool-store-client  │
              │ SourceAccessor │   │ (trait Store)      │
              │ fetchers/lazy  │   └───┬────────────┬───┘
              └────────────────┘       │            │
                       既存 nix-daemon ◄┘            └► tool-daemon
                     （フェーズ 1〜2）                  （フェーズ 3〜）
                                                         │
                                ┌────────────────────────┼─────────────────┐
                                ▼                        ▼                 ▼
                         local store DB          substituter         scheduler/builder
                       （SQLite WAL, pool）   （HTTP/2/3, 並列）    （動的 DAG, sandbox）
                                                                         │
                                                                build trace（CA/dyn-drv）

  共通基盤: nix-compat 相当（NAR, narinfo, ATerm, store path, hash, wire protocol）
```

### 5.1 実装言語とコードの再利用

- **実装言語は Rust を推奨する。** メモリ安全性、並列化のしやすさ、既存資産の多さが理由。
- **形式まわりは既存実装を流用する。** [snix](https://git.snix.dev/snix/snix) の `nix-compat` crate（NAR、narinfo、ATerm、store path、wire protocol など）の採用を第一候補とする。フェーズ 0 で、どこまで網羅しているか、ライセンス（GPL-3.0）が許容できるか、API が安定しているかを評価する（§11 Q1）。
- **評価器は自前で設計する。** 理由は2つ。
  - 並列評価、GC、string context の表現が性能の核であり、ここに設計の自由度が必要。
  - snix-eval、[fix](https://github.com/psyclyx/fix)（Zig）、C++ Nix の設計は参考にする。

### 5.2 crate 構成（案）

| crate | 責務 |
|---|---|
| `tool-compat` | 形式・ハッシュ・パス（`nix-compat` を包むか、流用する） |
| `tool-eval` | パーサ、コンパイラ、VM、builtins、drv の生成 |
| `tool-source` | `SourceAccessor`、fetcher（git、tarball、github など）、fingerprint キャッシュ |
| `tool-flake` | lock の計算、レジストリ、`call-flake` 相当 |
| `tool-eval-cache` | 属性単位の評価結果キャッシュ |
| `tool-store` | `Store` trait、ローカルストア、リモートストア、binary cache クライアント |
| `tool-build` | scheduler、builder、sandbox、build trace |
| `tool-daemon` | daemon 本体（worker protocol の互換層と新プロトコル） |
| `tool-cli` | CLI |

**設計原則として、グローバル状態を持たない。** 設定はすべて明示的な `Context` / `Settings` 値として渡す（C++ Nix の §2.2 にある、グローバル設定への参照が約 365 箇所ある問題の回避）。

---

## 6. コンポーネント設計

### 6.1 評価器（`tool-eval`）

#### 6.1.1 実行モデル
- **コンパイルの流れ**：パース → 静的スコープ解決 → バイトコード → VM。
  - `with` は「動的な lookup が必要な変数」として、コンパイル時に区別しておく。
  - 参照する位置ごとにインラインキャッシュを持たせる（attrset の識別子とスロット位置をキーにする）。C++ Nix はアクセスのたびに `forceAttrs` と最大8層の二分探索を行い、`with` 変数については thunk の回避も無効になっている（`eval.cc:933-964`）。
- **thunk は本当に遅延が必要なものだけにする。** 次のものは確保だけで失敗しないので、即座に構築する。
  - 関数リテラル（クロージャ）
  - 静的なキーだけを持つ attrset リテラル
  - リストリテラル
  - C++ Nix の `maybeThunk` は、変数、リテラル、空リストしか特別扱いしない（`eval.cc:1070-1111, 1426`）。
- **関数呼び出し**：
  - 形式引数（`{ a, b ? x, ... }`）の照合は、コンパイル時に「ソート済みの形式引数配列と、ソート済みの引数 attrset のマージ」として準備しておく。
  - 引数の数が分かっているカリー化された関数は、まとめて呼び出す。
- **VM のスタックと深さ制限**：
  - VM は自前のスタックを持ち、末尾呼び出しを最適化する。
  - 深さの制限は、強制と呼び出しの両方を数える1本にし、捕捉できるエラーにする。C++ Nix は関数呼び出しの深さ制限と、`_exit(1)` する SIGSEGV ハンドラの2本立て（`src/libmain/unix/stack.cc:77`）。
- **エラーは値として返す。**
  - トレースは「位置、テンプレート id、引数」の形で記録し、表示するときにだけ整形する。「もしかして」の候補もそのときに計算する。
  - `tryEval` に安い高速経路を用意する。
  - C++ Nix はエラーを作る時点で整形し、候補探索を全属性名に対して行い、巻き戻しのたびに `addTrace` する。
- **デバッガとプロファイラは別モードの VM（または計装したバイトコード）にし、通常の実行経路に条件分岐を置かない。** C++ Nix はホットパスで `debugRepl` を確認している。
- **パース結果のキャッシュ**：ファイルの内容ハッシュをキーにして、バイトコードをディスクにキャッシュできる構造にする（任意。§11 Q4）。nixpkgs 全体のパースだけで 700 MiB 以上、数秒かかる（`5d9fdab3d`）。
- **起動処理**：
  - 組み込み関数のテーブルは静的で不変なものにする。
  - `derivation` はネイティブに実装する。C++ Nix は `derivation-internal.nix` を起動のたびにパース・評価しており、呼び出しごとに `//` が出力の数＋2回走る。
  - ただし外から見える属性は保つ：`all`、`drvAttrs`、`outputName`、`type`、`outPath`、`drvPath`、そして「`outputs` の先頭が既定の出力」。
- **主要な組み込み関数に高速経路を用意する**：
  - `sort` の比較関数が `lessThan` のときの特殊化
  - `genericClosure` をハッシュベースにする
  - `replaceStrings` を多パターン一括検索にする
  - `attrNames` は辞書順のキャッシュを使う

#### 6.1.2 値の表現
- **基本方針**：タグ付き値と、ヒープオブジェクトの組み合わせ。
  - 評価器が扱う値は 16 バイト（タグ＋即値かポインタ）。
  - ヒープオブジェクト（Env のスロット、attrset、リストの要素、thunk）の中では、下位 3 ビットをタグにした 8 バイトの語に詰める。
    - 整数（-2^60 以上 2^60 未満）、bool、null は語に直接置き、ヒープに確保しない。
    - float と範囲外の整数は箱詰めする。評価中にヒープへ格納される float はまれで、NixOS 最小構成の評価で 71 個だった。
    - PoC で 16 バイトの値から変えたところ、NixOS 最小構成の確保量が 926 MB から 679 MB、最大 RSS が 899 MB から 731 MB に減った。命令数は 2% 減った（PoC の `docs/gc.md`）。
    - 1 語なので、並列評価で thunk の値やスロットを書き換えるときも 1 語のアトミック操作で済む。
  - C++ Nix はすべての値を別々に確保した `Value*` で持つ。これは 2010 年に保守的 GC の誤保持を避けるために選ばれた構造（`41c45a9b3`、`4dee28955`）で、正確な GC なら不要になる。
- **文字列**：GC がスキャンしない領域に置く。C++ Nix は 2025-11 の `7cd325294` で、スキャンされない `GC_MALLOC_ATOMIC` からスキャンされる `GC_MALLOC` に変わっている。
  - 連結はロープかスライスにし、`foldl'` で文字列を積み上げる O(n²) を避ける。
- **attrset**：
  - symbol id で引けるようにしつつ、名前順の列挙を O(n) で取り出せる表現にする。
  - C++ Nix は symbol id 順に並んでいるため、`attrNames` や `toJSON` のたびに文字列順で並べ直している（`attr-set.hh:424-434`）。
  - `//` は永続データ構造（HAMT など）か、層を重ねる方式。C++ Nix の layering（`maxLayers = 8`）は参考にする。
- **list**：小さいものは配列、大きいものは RRB-vector などの永続ベクタにし、`++` を O(log n) にする。`foldl'` による連結の O(n²) を回避する。
- **string context**：
  - 要素（store path、`drv^out`、`DrvDeep`）をインターンし、集合は「ソート済みの小さな配列」か「共有される永続集合」で表現する。
  - 補間するたびに起きていた、パース、`std::set` への構築、GC 文字列の再生成をなくす。

#### 6.1.3 並列評価
- **thunk を状態機械として設計する。**
  - 状態は「未評価 / 評価中（所有者 + 待機者キュー）/ 評価済み / 失敗」の4つ。
  - 状態の遷移は CAS で行う。
  - 自分が評価中の thunk に戻ってきたときだけ「無限再帰」と判定する（同じスレッドで再入した場合のみ）。
- **work-stealing scheduler で並列化する対象**：
  - `nix flake check`
  - 複数の installable
  - `nix-eval-jobs` 相当の全属性評価
  - `mapAttrs` の結果を後で強制する場面。ここは投機的に並列評価するかどうかを判断する（§11 Q5）。
- **呼び出しの深さ**：スレッドごとに持つ。
- **1 つの式の評価も、投機せずに並列化する**（PoC で実装。PoC の `docs/parallel.md`）：
  - `derivationStrict`、`--strict`、`deepSeq`、JSON 出力は、引数を必ず全部強制する。その引数を補助スレッドが先に強制する。
  - 補助スレッドは成功だけを記録する。エラー、`trace` などの副作用、循環する待ちに当たったら、評価中の thunk を評価前に戻して手を引く。最初のスレッドだけを見れば評価の順序は逐次と同じなので、エラーと出力の順序も逐次評価と一致する。
  - 評価中の印は状態タグだけで表し、thunk の本体（Env とコード、関数と引数）を残す。これで評価前に戻せる。
  - PoC（4 コア）では、`firefox.drvPath` が 1.9 倍、nixpkgs の 3000 パッケージが 1.8 倍速くなった。nixpkgs の全属性で、逐次評価と出力が一致した。NixOS の評価は 1.2 倍にとどまった。モジュールシステムの評価には、まだ並列にできる場所が見つかっていない。
- **独立したルートが多数ある評価（`nix-eval-jobs` 相当）は、複数の主スレッドで並列化する**（PoC の `altnix eval-jobs`）：
  - 補助スレッドで先回りする方式では、出力（エラー、`trace`、`warn`）を決める最初のスレッドが律速になった。nixpkgs の約 2 割のジョブはエラーで終わり、補助スレッドはエラーで手を引くためである。
  - そこで、各ジョブを別の主スレッドが評価する。主スレッドは手を引かず、エラーを thunk に記録する。ヒープは共有し、`stdenv` などの共通の依存は一度だけ評価する。主スレッドだけからなる待ちの循環は、逐次評価でも無限再帰になる場面なので、エラーにする。
  - 出力の順序は属性パスの順に固定し、ワーカーの数によらず同じ出力にする。
  - GC のマークは評価のスレッドと同じ数のスレッドで行う。
  - PoC（4 コア、nixpkgs 全体の 119,108 ジョブ）では、nix-eval-jobs と同じメモリ（7.8 GB）で同じ壁時計時間、CPU 時間は 12% 少なかった。12.7 GB を使えば 239 s（nix-eval-jobs は 286 s）。出力は `warnings` を除いて一致した。どのジョブに別名の警告が出るかは評価の順序で決まり、nix-eval-jobs 自身も実行ごとに違う。
  - 残る直列の部分は GC の停止と、メモリの上限でのルートの評価し直し。世代別 GC（§6.1.4）で減らす。

#### 6.1.4 メモリ管理
- **正確な GC を基本とする**（世代別で、アリーナ割り当て）。C++ Nix の Boehm GC は保守的なうえ、tagged pointer のために変位登録を広げており、偽の参照による保持を招きやすい（`eval-gc.cc:53,70-73`）。
- **失敗値は軽量にする。** C++ Nix は失敗した thunk ごとに finalizer を持つ例外を保持している（`value.hh:441-475`）。エラートレースは遅延して整形する。
- **ヒープの初期サイズと上限は設定で変えられるようにする。**
- **一意性による in-place 更新**（Perceus の reuse の考え方を部分的に取り入れる）：
  - オブジェクトに「参照が1本だけ」を示す一意性ビットを持たせる。共有された時点（thunk に捕まった、別の変数に束縛されたなど）で印を落とす。
  - `++`、`//`、`listToAttrs`、`mapAttrs` などの primop は、引数が一意なら破壊的に更新する。
  - バイトコードのコンパイル時に、`foldl'` のアキュムレータのように明らかに一意な経路を見つけ、専用の命令を出す。
  - これで `foldl' (acc: x: acc ++ [x])` や `foldl' (acc: x: acc // {…})` の O(n²) を償却 O(1) にできる。C++ Nix では毎回コピーしている（`eval.cc:2104-2134`）。
  - 先行事例：Lean 4 の `Array`（参照カウントが 1 なら破壊的に更新する）。

##### 検討して採用しなかった方式：Perceus（参照カウント）を主たるメモリ管理にする

Perceus（Reinking, Xie, de Moura, Leijen, PLDI 2021。Koka で採用し、Lean 4 も近い方式）は、コンパイラが参照カウントの増減命令（dup/drop）を正確な位置に挿入する方式。最後の使用の直後に解放され、参照カウントが 1 のオブジェクトはその場で再利用できる。前提は、正格評価であること、データが不変であること、循環参照がほぼ生じないこと。Nix の評価器はこの前提を満たさないため、主たるメモリ管理としては採用しない。

| 理由 | 内容 |
|---|---|
| 循環参照が構造的に、大量に生じる | Nix の `let` はすべて再帰的で、評価するたびに「Env → thunk → 同じ Env」の循環ができる（C++ Nix の `eval.cc:1398-1408`）。thunk を評価した後も、結果がクロージャなら循環は残る（再帰関数はすべてこの形）。`rec { }`、固定点（`lib.fix`、`pkgs = f pkgs`、overlay の `final: prev:`）、`derivation` の出力同士の相互参照も同じ。参照カウントは循環を回収できず、循環回収器を足せば結局トレース型 GC を併用することになる |
| 遅延評価で thunk を書き換える | Perceus の解析は、値が不変で最後の使用がコンパイル時に決まることを前提にしている。thunk は作られた後で結果に書き換えられ、どこから参照され続けるかは実行時にしか決まらない。dup/drop を省く最適化や借用がほとんど効かず、素朴な参照カウントになってしまう |
| 並列評価と相性が悪い | スレッドをまたいで共有されるオブジェクトの参照カウントにはアトミック操作が必要。並列評価では nixpkgs のグラフがほぼ丸ごと共有されるので、参照をコピーするたびにアトミック操作が走る |
| 評価器の使われ方と合いにくい | 評価はバッチ処理で、`pkgs` 経由で多くの値が最後まで生き続ける。重要なのはすぐに解放することより確保の速さで、大量に出る短命なゴミ（thunk、Env）は、世代別 GC の nursery が最も安く処理できる。同じく遅延評価の関数型言語である GHC も、トレース型の世代別 GC を採用している |

結論として、確保の速さと短命オブジェクトの処理は世代別のトレース型 GC に任せる。Perceus からは、上の「一意性による in-place 更新」だけを取り入れる。

#### 6.1.5 derivation の生成（`derivationStrict` 相当）
- **drv はいったんメモリ上の drv グラフに追加するだけにする。**
  - ATerm への直列化は1回だけ行い、そのバッファから drv の store path と masked hash の両方を求める。masked hash は、出力を空にしたメモリ上の構造から直接計算し、`Derivation` をコピーしない。
  - C++ Nix は、評価器の中で直列化と SHA-256 を3回、深いコピーを2〜3回行う。さらに store 側の `registerValidPath` で `.drv` を読み直してパースし、masked hash を4回目として再計算している。
  - store への書き込みは、ビルドやコピーが必要になった時点でまとめて行う（`--read-only` が自然に既定の動作になる）。
- **hash modulo のメモはグローバルにしない。** 評価セッションと store dir をキーにしたキャッシュにする（C++ Nix の `masked::hashes` は「FIXME: global」）。
- **`DrvDeep` context の展開**：drv の closure をメモ化して再利用する（`primops.cc:1794-1806`）。

#### 6.1.6 IFD（import-from-derivation）
- IFD はビルダーへの非同期の依頼として扱う。並列評価の場合、IFD を待っているあいだも他の thunk の評価を進める。

### 6.2 ソースアクセスと fetcher（`tool-source`）

#### 6.2.1 `SourceAccessor`
- すべてのファイル読み込みを、抽象化した `SourceAccessor` を通して行う。
  - 実装：実ファイルシステム、git の object、tarball キャッシュ、store、マウントされた仮想ツリー。
- **store へのコピーは「derivation の入力になったとき」と「IFD」に限る。** パスを文字列に変換しても、即座にはコピーしない。C++ Nix では `eval.cc:2636-2653` が即座にコピーしている。
  - 互換性のため、文字列としての値は、本物の CA store path になる。
  - 評価器内部では「まだコピーしていない store path」として扱い、derivation の入力に入ったときに実体化する。

#### 6.2.2 fingerprint とハッシュのキャッシュ
- **ローカルのディレクトリ**：`(dev, inode, mtime, ctime, size)` をキーに、ファイル単位の内容ハッシュを永続キャッシュする。
  - NAR 全体のハッシュは逐次計算が必要で、部分木のハッシュから合成できない。ただし内容に変化がなければ、NAR ハッシュの結果そのものをキャッシュから返せる。
  - 変化があった場合も、未変更ファイルのメタ情報を再利用して、走査コストを下げる。
- **git の作業ツリー**：fingerprint は「HEAD + 変更ファイルの (path, 内容ハッシュ)」から計算する。
  - **submodule があっても fingerprint を作れるようにする。** 各 submodule の fingerprint を合成する（C++ Nix の `git.cc:1127` の問題を回避）。
- **filter 付きのパス**（`builtins.path` + filter、`filterSource`、fileset）：
  - 「元のツリーの fingerprint + filter 関数を通過したパスの一覧」を、キャッシュのキーにする。
  - 通過したパスの一覧は評価しないと得られない。ただし filter の結果が同じなら、NAR のシリアライズとハッシュは省略できる。

#### 6.2.3 fetcher
- **git**：
  - shallow fetch を既定にする（C++ Nix ではドキュメントと実装が食い違っている）。
  - libgit2 か gitoxide を使い、認証ヘルパーへの対応方針を決める（§11 Q6）。
  - キャッシュの repack と GC を定期的に行う。
- **tarball / github**：
  - アーカイブは展開せず、git object store に取り込む方式を基本にする（C++ Nix の `tarball-cache-v2` に相当）。
  - API の応答は store ではなく HTTP キャッシュに置く。
- **substituter を先に確認しない。** final input は fetcher のキャッシュ（ローカル）を最優先する。substituter は fetcher が失敗したときのフォールバックとし、設定で無効にできるようにする（C++ Nix の `fetchers.cc:312-351` の問題を回避）。
- **TTL**：ドメインごと・入力の種類ごとに設定できるようにする。

- **PoC での実装（`altnix-fetch` と評価器の `fetch.rs`）**：
  - 取得したものはキャッシュに置き、NAR ハッシュで store path を決めて、その store path にマウントする（C++ Nix の `mountInput` と同じ）。store へのコピーは、ビルドや `eval-jobs` で derivation の入力として必要になったときだけ行う。C++ Nix の lazy trees と同じ方向。
  - tarball は展開してディレクトリとしてキャッシュし、NAR ハッシュもキャッシュする（git object store への取り込みはまだ）。Git は `git` コマンドで扱い、コミットの木は blob をフィルタなしで書き出す（`git archive` は `export-ignore` を必ず適用するので使わない）。
  - `tarball`、`file`、`path`、`git`、`github`、`gitlab`、`sourcehut` の入力で、返す属性と store path が C++ Nix 2.32.1 と一致した。サブモジュール、LFS、レジストリ、`access-tokens` は未対応。

### 6.3 flake（`tool-flake`）

- **ルート flake の `outPath` と `narHash` を遅延させる。**
  - ルート flake には lock file による narHash の検証が不要。そこで `self.outPath` を thunk にし、触られたときだけツリー全体をハッシュする。
  - `nix build .#foo` で `self` を derivation の `src` に使う場合は結局必要になる。それでも、評価だけのコマンドや eval cache がヒットしたときは、全体ハッシュを省略できる。
- **lock の計算**：
  - lock file が存在し、`flake.nix` の inputs 宣言が変わっていなければ、lock の再計算を省略する。
  - 変わっていれば、差分だけ再計算する。
  - follows の解決、override の照合、循環の検出は、マップを使って線形時間で行う。C++ Nix は O(ノード数 × override 数) で、検出も線形スキャン。
- **`call-flake` 相当**：
  - input の `outPath` だけに触れた場合、その input の `outputs` は評価しない。C++ Nix の `call-flake.nix:79-95` では `//` の構造上、強制されてしまう。
- **`builtins.getFlake`**：(locked ref, 設定) をキーにメモ化する。
- **レジストリ**：グローバル、ユーザー、flake ごとのレジストリをサポートする。lookup はハッシュマップで行う。
- **legacy との接続**：`<nixpkgs>` 形式は既定では無効にする。オプトインで「レジストリの flake の `outPath`」に解決する互換モードを用意する（§11 Q3）。

- **PoC での実装（評価器の `flake.rs`）**：`readFlake`、`lockFlake`（`computeLocks`）、`callFlake` は C++ Nix を移植し、`call-flake.nix` はそのまま埋め込んだ。まず出力の一致を優先し、上の遅延化（`self.outPath`、入力の `outputs`）はまだ行っていない。
  - lock file の作成と更新（全体、一部の入力、入力の変更と削除）で、書かれる lock file はバイト単位で、差分の表示も C++ Nix と一致した。
  - C++ Nix の挙動で合わせる必要があったもの：入力の取得はプロセスごとに一度（`InputCache`）、Git の作業ツリーの状態もプロセスごとに一度（`getCachedWorkdirInfo`。lock file を書いた後に読み直しても dirty にならない）、`--override-input` では lock file を書かない。

### 6.4 eval cache（`tool-eval-cache`）

- **キー**：(ルートの fingerprint, 各 input の fingerprint の集合, 評価設定のハッシュ)。lock file 全体はキーに含めず、input の更新が影響を与える範囲を最小にする。
- **lock の確定前に引く**：lock file と `flake.nix` が変わっていなければ、lock を計算する前にキャッシュを引く（C++ Nix の `flake.cc:1015` の FIXME の実現）。
- **impure でも使う**：
  - `--impure` でも、評価中に impure な値（環境変数、`currentTime`、外部ファイル）に触れなかった属性はキャッシュしてよい。
  - そのために「impure な入力への依存」を追跡する。
- **legacy 式（`--file`、`--expr`）**：ファイルの fingerprint をキーにすれば、同じ仕組みで扱える。
- **ストレージ**：
  - SQLite（WAL）を使い、書き込みは短いトランザクションでまとめて行う。C++ Nix はプロセスの生存期間中、ずっと書き込みロックを保持している。
  - 最終アクセス時刻に基づく GC を行う。

- **PoC での実装（評価器の `deps.rs`、CLI の `evalcache.rs`）**：上のキーの代わりに、**評価が読んだものを記録し、それが変わっていなければ結果を再利用する**方式にした。
  - 記録するもの：実ファイルシステムを通して読んだファイルの内容、ディレクトリの一覧、ファイルの種類、環境変数、Git の index と `HEAD`。ストアと fetcher のキャッシュは変わらないので記録しない。`currentTime` やロックされていない入力の取得があった実行は再利用しない。`trace` と `warning` は記録して出し直す。
  - ロックされた flake の fingerprint をキーにする方式（C++ Nix）は、この方式の特別な場合になる（ロックされた入力はすべてストアかキャッシュにあり、記録されない）。加えて、dirty な flake とファイルの評価（`nix-build file.nix -A attr` 相当）でも使える。
  - 単位はコマンド全体（評価全体の依存の和集合なので、余分に無効になることはあっても、古い結果を返すことはない）。属性単位・ジョブ単位で再利用するには、共有される thunk を通して依存を伝える仕組み（thunk 単位の依存の追跡）が要る。`eval-jobs` を差分で評価するにはこれが必要で、今後の課題。
  - nixpkgs の `hello` のビルドで、評価ありの 0.48 s に対し再利用は 0.08 s（依存 1,676 件の検査を含む）。

### 6.5 ストアと daemon（`tool-store` / `tool-daemon`）

#### 6.5.1 `Store` trait
- **非同期 API を基本にする。** 同期 API は薄いラッパーにとどめる（C++ Nix は呼び出しのたびに promise/future で包んでいる）。
- **一括で扱う API を一級市民にする**：`query_path_infos(&[StorePath])`、`query_build_traces(&[DrvOutput])` など。
- **キャッシュ**：path info を LRU でキャッシュし、「存在する」「存在しない」の両方を記録する。

#### 6.5.2 プロトコル
- **既存の worker protocol の互換層を持つ。**
  - サーバー側：既存の `nix` CLI から本ツールの daemon を使えるようにする。
  - クライアント側：本ツールの CLI から既存の `nix-daemon` を使えるようにする。フェーズ 1〜2 ではこちらが主になる。
  - サポートするバージョンの下限は、ネストした `DerivedPath` を送れる 1.30 以上を目安にする（§11 Q2）。
- **本ツール同士は新しいプロトコルで通信する。**
  - 多重化（要求 ID 付き）。
  - 一括で扱う op。
  - サーバーからのストリーミング（ビルドログ、進捗）。
  - スキーマはバージョン付きにし、機能の有無をネゴシエーションで決める。
  - トランスポートは Unix ソケットと SSH。

#### 6.5.3 ローカルストア
- **SQLite（WAL）を使う。** 読み取り接続をプールして並列化し、書き込みは1本の接続で行う。
- **スキーマの方針**（§11 Q7）：
  - 案 A：C++ Nix の `db.sqlite` と互換にする。戻りやすさ（G6）は最大だが、スキーマを改善する自由はない。
  - 案 B：独自スキーマ + C++ Nix 形式との相互変換ツールを用意する。
  - **推奨は案 A から始めること。** 必要になったら、追加のテーブル（build trace、キャッシュ）は別ファイルの DB に置く。
- **GC**：
  - 到達可能性の判定は、メモリ上の参照グラフを一括で構築してから行う。
  - 削除は一括トランザクションで行う。
  - build trace の参照も GC の対象に含める（§6.7.4）。
- **出力の登録**：参照のスキャン、NAR ハッシュの計算、重複排除（optimise）を、1回の読み込み（ストリーム）で行う。
- **ロック**：同じ drv（CA を含む）を同時にビルドしないよう、drv の単位でロックする（C++ Nix の `derivation-building-goal.cc:400` の FIXME）。

#### 6.5.4 substituter クライアント
- HTTP/2 または HTTP/3 で、narinfo を多重化して並列に問い合わせる。substituter の優先度は維持する。
- NAR の取得、展開、ハッシュの検証を、パイプライン化したストリーム処理にする（xz、zstd、brotli に対応）。
- 署名の検証（ed25519）と、trusted-public-keys のポリシーは C++ Nix と同じ意味にする。
- 「存在しない」という narinfo の結果は TTL 付きでキャッシュする。「存在する」結果もディスクにキャッシュする。

### 6.6 scheduler と builder（`tool-build`）

#### 6.6.1 scheduler
- **「実行中に伸びる DAG」として設計する。** ノードの種類は次のとおり。
  - `Substitute(path)`
  - `ResolveDrv(drv)`
  - `Build(resolved drv)`
  - `QueryTrace(drv, out)`
  - `Materialize(DerivedPath)`
  - dynamic derivation では、`Build` の結果から新しい `Materialize` ノードが生まれる。
- **async ランタイム上のタスクとして実装する。**
  - ジョブ数の上限（ビルド、ダウンロード、CPU）は、セマフォで個別に管理する。
  - 出力の登録は別スレッドで行い、ログ処理を止めない。
- **`--check`、`--repair`、remote build は別のノード種別として表現する。** goal の中のフラグで分岐させない（上流 #12628 の反省）。
- **エラーは型で扱う。** `abort` や `assert` で落ちる経路を作らない。resolve の失敗は理由を保持して伝える。

#### 6.6.2 builder
- **Linux のサンドボックス**：
  - mount、pid、net、uts、ipc、user の各 namespace、cgroup v2、seccomp を使う。
  - C++ Nix と同等の環境を用意する：ビルドディレクトリのパス、`/bin/sh`、`/dev`、`/proc`、FOD だけネットワークを許可。
- **macOS**：sandbox-exec のプロファイル互換（後のフェーズ）。
- **remote build**：新プロトコル上で、resolve 済みの drv と入力を送る。worker protocol（ssh-ng）互換も持つ。
- **recursive build / builder RPC**：dynamic derivation のツール（nix-ninja など）が必要とする範囲で、ビルド中から store に追加したり、drv を登録したりできるようにする（§11 Q8）。

### 6.7 CA、dynamic derivation、build trace

#### 6.7.1 方針
- **CA と dynamic derivation は常に有効にする。** feature フラグは設けない。
- **drv と JSON の形式は C++ Nix と互換にする。** 読み書きに対応する対象は次のとおり。
  - `__contentAddressed`、`outputHashMode = "text"`
  - `DrvWithVersion("xp-dyn-drv")` の ATerm
  - derivation の JSON v4
  - placeholder の計算方式（`nix-upstream-output:<drv の store path のハッシュ部>:<outputPathName>` の SHA-256。`src/libstore/downstream-placeholder.cc:12-20`）
  - `DerivedPath` の文字列表記（`drv^out^out`）
- **nixpkgs 本体は input-addressed のまま使う。** CA と dynamic derivation は、利用者自身の derivation と、そこから派生するツールチェーンに使う。既存キャッシュとの互換（G2）には影響しない。

#### 6.7.2 build trace のモデル
- **base build trace**：キーは (resolve 済み drv の store path, 出力名)、値は (出力の store path, 署名の集合)。
- **derived build trace**：resolve 前のキーで引けるようにするキャッシュ。必ず対応する base entry と一緒に保持する。
  - 上流のドキュメント（`doc/manual/source/store/build-trace.md`）が定義する coherence の要件に従う。
- **登録時に署名を検証する。** 信頼ポリシーは trusted-public-keys と同じ枠組みで扱う（上流 #11393 の回避）。

#### 6.7.3 resolve の「浅い」実行
- drv を resolve するのに必要なのは、**入力の出力パスだけ**。入力の実体（NAR）を取得する必要はない。
  - `ResolveDrv` ノードは、入力の出力パスを build trace（ローカル → substituter の順）から得る。実体は取得しない。
  - resolve した drv のキーで build trace を引き、ヒットすれば、最終出力だけを substitute する。
  - 入力の実体を取得するのは、ローカルでビルドすることが決まった場合だけ。
- これにより C++ Nix の「substitute するために入力を全部 realise させる」問題（#11928）と、「resolve 前のキーで引くと必ず外れる」問題を、構造的に回避する。

#### 6.7.4 GC との統合
- build trace のエントリを、GC の根または参照として扱う。
  - 出力パスが GC されたら、対応する base entry も無効化するか、削除可能にする。
  - これにより、非決定的なビルドの再ビルドが「already have another one locally」で失敗する問題をなくす。
- build trace を GC の root として保持するかどうかは、ポリシーで設定できるようにする。

#### 6.7.5 キャッシュとの互換
- **binary cache の build trace**：C++ Nix の配置（`build-trace-v2/<drvPath>/<out>.doi`）を読み書きできるようにする。
- **build trace の endpoint を持たないキャッシュ**：一般的な S3/HTTP キャッシュなど。これらは「存在しない」として扱い、エラーにしない。
- **`queryMissing` / `--dry-run` 相当**：DAG を実行せずに走らせる「計画モード」として実装する。dynamic derivation も、得られる範囲で見積もりに含める。

### 6.8 CLI（`tool-cli`）

- **flake installable を基本にする。** 互換の対象は、利用の多いサブコマンド：`build`、`eval`、`develop`、`shell`、`run`、`profile`、`copy`、`store`、`flake`、`derivation`、`path-info`、`log`、`why-depends`。
- **`--file` / `--expr` も同じ installable の仕組みで扱う。** グローバル設定は書き換えない。
- **出力**：`--json` の形式は C++ Nix と互換にする。ログと進捗の表示は独自でよい。
- **互換 shim**：`nixos-rebuild`、home-manager、direnv、nix-direnv が呼ぶ範囲で、`nix-build`、`nix-instantiate`、`nix-store` の最小限の shim を用意する（フェーズ 5）。

---

## 7. 開発フェーズ

各フェーズは、それ単体で利用価値がある状態で終える。途中で止まっても成果が残るようにするため。

### フェーズ 0：土台と測定基盤
- **作業**
  - `nix-compat` の評価と採用判断（§11 Q1）。
  - 差分テストの基盤を作る：固定した nixpkgs に対し、C++ Nix（`nix-eval-jobs`）で全属性の drvPath の一覧を生成し、正解データとして保存する。
  - ベンチマークの基盤を作り、C++ Nix の基準値を測る（§9）。
- **完了条件**
  - CI で「C++ Nix との drvPath の一致率」と「ベンチマークの基準値」を出せる。

### フェーズ 1：評価器（MVP）
- **作業**
  - パーサ、コンパイラ、VM、主要な builtins、`derivationStrict` 相当、corepkgs。
  - store は既存の `nix-daemon` に worker protocol でつなぐクライアントとして扱う。
  - 最小限の CLI：`eval`、`build`（`--file` / `--expr`）。
- **完了条件**
  - `tests/functional/lang` の期待値テストをすべて通る（一部はエラーメッセージの文面が異なることを許容する）。
  - 固定した nixpkgs で、評価可能な全属性の drvPath が 100% 一致する。
  - NixOS の最小構成の `system.build.toplevel` の drvPath が一致する。
  - `hello` をビルドし、`cache.nixos.org` から substitute できる（既存の daemon 経由）。
- **中間の成果物**：「高速な `nix eval` / `nix-eval-jobs` の代替」として単独でリリースできる。

### フェーズ 2：並列評価と flake
- **作業**
  - 並列評価（§6.1.3）。
  - `tool-source`（lazy なソース、fingerprint キャッシュ、fetcher）。
  - `tool-flake`、`tool-eval-cache`。
- **完了条件**
  - 既存の主要な flake（nixpkgs、home-manager、それなりの規模の NixOS 構成）の `flake.lock` を、変更せずに読み書きできる。
  - lock を更新した場合も、C++ Nix と同じ結果になる。
  - §9 の flake 関連の指標で、基準値を明確に上回る。

### フェーズ 3：ストア、daemon、substituter
- **作業**
  - ローカルストア、GC、substituter のクライアント。
  - daemon（worker protocol の互換層 + 新プロトコル）。
- **完了条件**
  - 既存の NixOS マシンで `nix-daemon` を本ツールの daemon に置き換え、既存の `nix` CLI と `nixos-rebuild` が動く。
  - C++ Nix の daemon に戻しても、ストアの整合性が保たれる（G6）。
  - substitute のスループットが基準値を上回る。

### フェーズ 4：scheduler、builder、CA、dynamic derivation
- **作業**
  - 動的な DAG の scheduler、Linux のサンドボックス、remote build。
  - build trace、CA、dynamic derivation、builder RPC。
- **完了条件**
  - nixpkgs の stdenv をソースから bootstrap して、C++ Nix と同じ出力パスになる（input-addressed）。
  - `tests/functional/ca`、`tests/functional/dyn-drv` 相当のシナリオを通る（移植したテスト）。
  - [nix-ninja](https://github.com/pdtpartners/nix-ninja) で中規模の C/C++ プロジェクトをインクリメンタルにビルドできる。
  - 浅い resolve（§6.7.3）で、中間物を取得せずに dynamic derivation の出力を substitute できる。

### フェーズ 5：NixOS 統合と移行
- **作業**
  - `nixos-rebuild`、home-manager、direnv との互換。
  - 互換 shim、NixOS モジュール（`nix.package` を置き換えるか、専用のモジュールを用意する）。
  - macOS 対応。
- **完了条件**
  - 実際に利用する NixOS / home-manager の構成を、本ツールだけで日常的に運用できる。

---

## 8. テスト戦略

| 種類 | 内容 |
|---|---|
| 言語の期待値テスト | C++ Nix の `tests/functional/lang/*.nix` と期待値ファイルを移植する（ライセンスと出典を明記する） |
| nixpkgs の差分テスト | 固定したリビジョンで、全属性の drvPath と `meta` の一部を C++ Nix と比較する。大規模なのでシャード化して CI で並列に実行する。nixpkgs の更新に合わせて正解データを再生成する |
| NixOS の差分テスト | 代表的な構成（最小構成、デスクトップ、サーバー）の `toplevel` の drvPath を比較する |
| 形式の往復テスト | ATerm、JSON、NAR、narinfo、`flake.lock` について、C++ Nix が生成した値を本ツールで読み書きしても、バイト単位で一致することを確認する |
| プロトコルの相互運用 | 本ツールの CLI と C++ Nix の daemon、C++ Nix の CLI と本ツールの daemon の組み合わせで、ビルド、コピー、GC を確認する |
| fuzz テスト | パーサ、ATerm、NAR、worker protocol のデコーダ |
| CA / dyn-drv | C++ Nix の `tests/functional/ca`、`dyn-drv` のシナリオを移植し、C++ Nix ではカバーされていない部分を追加する：substitution、`copy`、GC、dry-run、remote build |
| VM テスト | NixOS VM で、daemon の置き換え、multi-user、remote build を確認する |

---

## 9. ベンチマーク指標

基準値はフェーズ 0 で C++ Nix（同じマシン、同じリビジョン）を測って確定する。目標値は基準値を測ったあとに設定する。

| ID | 指標 | 測定方法（案） |
|---|---|---|
| B1 | nixpkgs 全属性の評価時間、ピークメモリ | `nix-eval-jobs` 相当、ワーカー1本と N 本 |
| B2 | NixOS の `toplevel` の評価時間 | 代表的な構成 3 種 |
| B3 | 何もしないときの `build` のレイテンシ | `build nixpkgs#hello`（ビルド済み、キャッシュが温まった状態） |
| B4 | 大規模リポジトリの flake で、1ファイル変更後の評価時間 | 数 GB 規模の monorepo、submodule ありとなし |
| B5 | eval cache がヒットしたときのレイテンシ | `eval .#foo.name` |
| B6 | lock の更新時間 | input 数 50 以上で、follows が多い flake |
| B7 | substitute のスループット | NixOS の closure（数千パス）を空のストアに取得 |
| B8 | GC の時間 | 数十万パスのストア |
| B9 | dynamic derivation のオーバーヘッド | nix-ninja で、1ファイル変更後のリビルド時間 |
| B10 | ディスク書き込み量 | B3〜B5 での store と cache への書き込みバイト数 |

---

## 10. リスクと緩和策

| リスク | 影響 | 緩和策 |
|---|---|---|
| 開発期間が長期化する（Tvix は 2021 年に始まり、fork 後の snix も本番利用前） | 大 | 各フェーズが単独で価値を持つように区切る。最初の成果物を評価器に絞る |
| nixpkgs の最低 Nix バージョンが上がり、新しい builtin が必要になる | 中 | nixpkgs のリリースノートと `lib/minver.nix` を監視し、差分テストを定期的に回す |
| 評価器の細かい非互換（float、正規表現、context）が見つかりにくい | 大 | §4.3 の落とし穴ごとに個別テストを書き、差分テストで全数を検証し、fuzz で差分を取る |
| C++ Nix 側で形式が変わる（drv の JSON、build trace の配置、worker protocol） | 中 | C++ Nix のリリースを追い、形式ごとにバージョン付きで対応する |
| `db.sqlite` の互換によって、ストアの設計が制約される | 中 | 追加データは別の DB に置き、将来は相互変換ツールで移行する |
| サンドボックスの差異で、一部のパッケージのビルドが失敗する | 中 | Hydra のビルド結果を基準に、失敗したパッケージの差分を調べる |
| コミュニティとの分断（「また別の Nix」） | 中 | 形式とテストを上流と共有する。改善は可能な範囲で上流にも還元する |
| ライセンス（snix 由来のコードは GPL-3.0、C++ Nix は LGPL-2.1） | 中 | フェーズ 0 で方針を決める（§11 Q1） |

---

## 11. 未決事項

| ID | 論点 | 選択肢 / 現時点の考え |
|---|---|---|
| Q1 | `nix-compat` を採用するか（機能の網羅性、GPL-3.0、API の安定性） | 採用 / fork / 自前実装。網羅性とライセンスを評価してから決める |
| Q2 | worker protocol の互換範囲の下限 | 1.30 以上（ネストした `DerivedPath`）を目安。build trace のキー変更（`realisation-with-path-not-hash`）に対応するかどうか |
| Q3 | `<nixpkgs>` 互換モードを用意するか | 既定では無効。オプトインでレジストリに解決 |
| Q4 | バイトコードをディスクにキャッシュするか | 効果を測ってから決める（パースとコンパイルの時間が占める割合しだい） |
| Q5 | 投機的な並列評価をどこまで行うか | まずは明示的な並列（複数の installable、flake check、全属性評価）から |
| Q6 | git の実装（libgit2 / gitoxide / `git` CLI） | 認証ヘルパーと LFS への対応が論点 |
| Q7 | ローカルストア DB のスキーマ（案 A：互換 / 案 B：独自） | 案 A から始める |
| Q8 | recursive-nix と builder RPC のどちらの方式に対応するか | nix-ninja など実際の利用者が必要とする方式を優先する |
| Q9 | Darwin 対応の優先度 | フェーズ 5 以降 |
| Q10 | 名前、ライセンス、ガバナンス | 未定 |
| Q11 | 評価結果の互換の対象とする C++ Nix のバージョン | nixpkgs が要求する最低バージョン（2.18）か、最新版か。masked hash の衝突時の挙動（`b26204ba1`）のように、バージョンによって drv path が変わる変更がある |

---

## 12. 参考

### C++ Nix のドキュメント（本リポジトリ）
- `doc/manual/source/release-notes/rl-2.35.md`：lazy trees（ソースを store に遅延コピーする）
- `doc/manual/source/store/build-trace.md`：build trace の定義と coherence
- `doc/manual/source/store/resolution.md`：derivation の resolve
- `doc/manual/source/store/derivation/outputs/input-address.md`：hash modulo と deferred output

### 上流の issue とマイルストーン
- [ca-derivations stabilisation（milestone #35）](https://github.com/NixOS/nix/milestone/35)
- [dynamic-derivations stabilisation（milestone #39）](https://github.com/NixOS/nix/milestone/39)
- [Tracking issue for RFC 92: Dynamic derivations（#6316）](https://github.com/NixOS/nix/issues/6316)
- [Reworking the goals（#12628）](https://github.com/NixOS/nix/issues/12628)

### メモリ管理の文献
- Reinking, Xie, de Moura, Leijen. [Perceus: Garbage Free Reference Counting with Reuse](https://www.microsoft.com/en-us/research/publication/perceus-garbage-free-reference-counting-with-reuse/)（PLDI 2021）
- Ullrich, de Moura. [Counting Immutable Beans: Reference Counting Optimized for Purely Functional Programming](https://arxiv.org/abs/1908.05647)（IFL 2019。Lean 4 の参照カウントと破壊的更新）

### 関連プロジェクト
- [snix](https://git.snix.dev/snix/snix) / [Announcing Snix](https://snix.dev/blog/announcing-snix/)
- [fix: Fast nIX language evaluator（Zig）](https://github.com/psyclyx/fix)
- [nix-ninja](https://github.com/pdtpartners/nix-ninja)（[dynamic derivations の解説](https://github.com/pdtpartners/nix-ninja/blob/main/docs/dynamic-derivations.md)）
- [nix-eval-jobs](https://github.com/NixOS/nix-eval-jobs)
- [devenv is switching its Nix implementation to Tvix](https://devenv.sh/blog/2024/10/22/devenv-is-switching-its-nix-implementation-to-tvix/)
- [Changelog: introducing lazy trees（Determinate Systems）](https://determinate.systems/blog/changelog-determinate-nix-352/)
- [Nixpkgs release notes](https://nixos.org/manual/nixpkgs/stable/release-notes)（最低 Nix バージョンの推移）

---

## 付録 A：C++ Nix 評価器の変遷と性能劣化要因

フェーズ 1 の設計判断の根拠として、C++ Nix の評価器の歴史を git の履歴から整理した。数値は上流のコミットメッセージにある計測値。

### A.1 3つの時代

| 時代 | 内容 | 主なコミット |
|---|---|---|
| 2003〜2010 | ATerm 上の項書き換え評価器。call-by-name で、関数本体に引数を代入し、結果をメモ化していた | — |
| 2010 fast-eval | AST を直接たどるインタプリタ、call-by-need（thunk を更新）、静的スコープ解決（level/displacement）に全面書き換え。「旧評価器のほぼ7倍速い」 | `0910ae956`、`9985230c0`、`04c4bd362`、マージは `8032f26ca` |
| 2010-10 | Boehm GC を導入（それまで新しい評価器はメモリを解放していなかった）。保守的 GC に合わせて、Env や Attr が `Value*` を持つ構造に作り直された | `e879a0371`、`41c45a9b3`、`4dee28955`、`2dc6d5094`（`maybeThunk`） |
| 2022〜2026 | 改修の時代。位置情報とシンボルのテーブル化、Value の 16 バイト化、attrset の構造共有、AST 用アリーナ、並列評価の下準備（16 バイトのアトミックな Value、並行コンテナ、失敗値） | #6218、#13407、#13987、#15447、#15286 |

**現在の負債の大半は、2010 年の2つの決定から来ている。** 1つ目は保守的 GC（Boehm）を採用したこと。2つ目は、AST を C++ の再帰でたどり、エラーを C++ 例外だけで扱う実行モデルにしたこと。

### A.2 規模感（nixpkgs で `nix-env -qa`、`6138bc3de` 2025-09）

| 項目 | 値 |
|---|---|
| CPU 時間 | 約 21 秒（GC は約 0.85〜1.6 秒） |
| Value | 3,850 万個（616 MB） |
| thunk | 2,940 万個（回避できた分は別に 2,820 万個） |
| Env | 2,330 万個（461 MB） |
| 関数呼び出し / primop 呼び出し | 2,100 万回 / 1,230 万回 |
| attrset | 520 万個（構造共有の導入後で 0.9 GB） |
| `//` | 217 万回（コピーした値は 6,100 万個から 2,600 万個に削減） |
| GC ヒープ | 1.78 GB から 1.25 GB に削減 |

nixpkgs 全体のパースだけで 700 MiB 以上、数秒かかる（`5d9fdab3d`）。

### A.3 主な劣化要因と経緯

| 要因 | 現状 | 経緯 | 本設計での対応 |
|---|---|---|---|
| 値を全部ヒープに確保し、ポインタで持つ | Env も Attr もリストも `Value*` で持つ | 保守的 GC の誤保持を避けるため（2010） | 正確な GC にし、スカラーはスロットに直接置く |
| 保守的 GC | 全ワードをポインタ候補としてスキャンし、オブジェクトを動かせない。毎回ヒープ全体をマークする | 誤保持の実例：glibc の内部変数で RSS が 718→455 MiB（`455d1f01d`）、カウンタで 455→292 MiB（`0629601da`） | 世代別の正確な GC |
| 文字列が GC にスキャンされる | `GC_MALLOC` で確保 | 2025-11 の `7cd325294`（理由の記載なし） | スキャンしない領域に置く |
| `with` の検索 | アクセスのたびに強制と二分探索。thunk の回避も無効 | 2023 env-diet（`1fe66852f`）でトレードオフとして選ばれた | 参照位置ごとのインラインキャッシュ |
| thunk の作りすぎ | 変数とリテラル以外はすべて thunk になる | `maybeThunk`（2010）以降ほとんど拡張されていない | クロージャ、attrset リテラル、リストは即座に構築 |
| C スタックへの依存 | 末尾呼び出しがない。スタックは 60 MiB。SIGSEGV で `_exit(1)` | 2013 の SIGSEGV ハンドラ、2023 の関数呼び出しの深さ制限 | VM のスタック、末尾呼び出しの最適化、単一の深さ制限 |
| 例外が唯一のエラー経路 | エラーを作る時点で整形し、全属性名に候補探索。行と列を求めるのにファイルを読み直す | #6204（2023）、`5d9fdab3d`（2024）。デバッガ経由の過剰な呼び出しで 22.9 s→784 ms の修正（`adbd08399`） | エラーは値で返し、表示時に整形 |
| string context を毎回テキストからパース | 補間のたびにパース、`std::set` への構築、再確保 | 2006 年に導入。2023 年の型付け（`5576d5e98`）でも「性能が怖くて」保存形式は文字列のまま | インターンした id のソート済み配列 |
| attrset が symbol id 順 | `attrNames` などで毎回文字列ソート | 2022 年以前はアドレス順（`8775be339`） | 辞書順の情報を持つ表現にする |
| `//` の層構造 | 最大8層の検索と k-way マージ | 2025 #13987：ヒープ −30% と引き換えに最大 10% 遅く | 永続データ構造と定期的な平坦化 |
| パース結果のキャッシュがない | 毎回すべてを字句解析からやり直す | 2013 年に削除（`6f809194d`） | バイトコードのキャッシュ |
| `derivation` が Nix で実装されている | 起動のたびにパースし、呼び出しごとに `//` が走る | 2012（`921111d19`） | ネイティブ実装 |
| drv の多重ハッシュ | 評価器内で3回、store 側で4回目 | 評価中の書き込みは 2003 年から、メモ化は 2005 年から | 直列化とハッシュ計算を1回にし、書き込みは遅延 |
| 並列化の後付け | thunk の更新、呼び出しの深さ、トレースがスレッド非安全 | 2024〜2026 年に少しずつ下準備（本体はまだない） | 最初から CAS ベースの thunk とスレッドごとの割り当て |

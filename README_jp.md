# llm-wiki

個人用の LLM ベース知識ウィキを構築・維持する Claude Code スキル。

LLM が元の文書を読み、構造化された Markdown ウィキを生成し、常に最新の状態に保ちます。質問のたびに一から推論し直す RAG 方式とは異なり、一度整理した知識を蓄積して再利用します。

Karpathy の [llm-wiki.md](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) パターンに基づきます。

## アーキテクチャ

3 つのレイヤー:

```
raw/          # 元のソース文書 (不変、ここにファイルを置く)
wiki/         # LLM が生成・管理する Markdown ページ
purpose.md    # このウィキが存在する理由
CLAUDE.md     # スキーマとワークフロー
```

`wiki/` はネームスペースが散らからないよう、固定されたバケットに分割されています:

- `entities/` — 人物、チーム、組織、プロダクト
- `concepts/` — パターン、方法論、技術用語、理論
- `sources/` — インジェストされた各ソースの要約ページ
- `synthesis/` — クエリから導かれたクロスソースのページ
- `synthesis/daily/` — 1 日単位のダイジェスト
- `index.md`, `overview.md`, `log.md` — カタログ、スナップショット、履歴

すべてのページは frontmatter に `sources: [raw/...]` を持ちます。このフィールドが source-overlap クエリシグナルと delete カスケードを支えています。

## サブコマンド

`/llm-wiki <subcommand> [args]` の形で呼び出します:

| サブコマンド | 用途 |
|---|---|
| `build [scenario]` | カレントディレクトリにウィキ構造をブートストラップ |
| `ingest [path]` | `raw/` 内のソースを読み込んでウィキに統合 (パス省略時は `raw/` 全体を処理) |
| `query <question>` | ウィキの内容で回答 (引用付き) |
| `daily [date]` | 1 日分の raw 入力をダイジェストに合成 |
| `lint` | 矛盾・孤立ページ・抜けのヘルスチェック |
| `delete <path>` | ソースを削除しウィキをカスケードクリーンアップ |

## クイックスタート

1. ウィキを置くディレクトリで `/llm-wiki build` を実行。シナリオ (research, reading, personal, business, general) を選び、`purpose.md` を埋めます。
2. `raw/` にソースファイルを置きます (複数可)。
3. `/llm-wiki ingest` を実行 — `raw/` 配下のすべてのファイルを走査してウィキに統合します。特定のファイルだけ処理したい場合は `/llm-wiki ingest raw/<file>`。SHA256 ハッシュで既にインジェスト済みのファイルは自動でスキップされ、各ファイルごとに「分析 → 3〜5 個の要点を議論 → 書き込み」の順で進みます。
4. `/llm-wiki query <question>` で質問。回答にはウィキページと元ファイルが引用されます。
5. 1 日の終わりに `/llm-wiki daily` を実行し、その日の raw を 1 つのダイジェストに圧縮します。
6. 定期的に `/llm-wiki lint` を回し、矛盾・孤立ページ・空白領域を点検します。

## 設計原則

- **キュレーションは人、メンテは LLM。** ソース選定と質問は人間、整理と接続は LLM。
- **Raw は不変。** `raw/` は読み取り専用。`delete` はウィキページのみ整理し、元ファイルには触れません。
- **2 段階インジェスト。** 先に分析 → 要点を議論 → 書き込み。この間の停止点が矛盾を捕まえます。
- **すべての主張に出典を。** ウィキ上のあらゆる主張は、元ソースまたは他のウィキページまで遡れます。
- **再導出ではなく蓄積。** 良い回答は synthesis ページとしてウィキに戻して保存します。
- **ハッシュで重複読み込みを回避。** SHA256 で raw ファイルをチェックし、変わっていないソースは再読込しません。
- **早すぎるツール化はしない。** 200 ページ未満の規模では index ファイル + 4 つの関連性シグナルで十分。ベクトル検索は実際の限界に達するまで導入しません。

## クエリの関連性 (Markdown のみで完結)

`query` は埋め込みを使わず、4 つのシグナルで候補ページを拡張します:

| シグナル | 重み |
|---|---|
| Source overlap (`sources[]` の共有) | ×4.0 |
| 直接の `[[wikilink]]` | ×3.0 |
| 共通の隣接ノード (Adamic-Adar) | ×1.5 |
| タイプの親和性 | ×1.0 |

## 互換性

`wiki/` ディレクトリは Obsidian 互換の `[[wikilink]]` と YAML frontmatter を持つ素の Markdown です。Obsidian で開いてグラフビューを使うも良し、git リポジトリとして扱いインジェストのたびにコミットするも良しです。

## 関連リンク

- スキル全仕様: [`.claude/llm-wiki/SKILL.md`](.claude/llm-wiki/SKILL.md)
- 他の言語: [English](README_en.md) / [한국어](README.md)
- 元の gist: [karpathy/llm-wiki.md](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
